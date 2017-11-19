## 基于ptrace的Android so库注入分析  
### 获取要注入的进程的pid
```c
int main(int argc, char** argv) {    
    pid_t target_pid;    
    target_pid = find_pid_of("/system/bin/surfaceflinger");    
    if (-1 == target_pid) {  
        printf("Can't find the process\n");  
        return -1;  
    }  
    inject_remote_process(target_pid, "/data/libhello.so", "hook_entry",  "I'm parameter!", strlen("I'm parameter!"));    
    return 0;  
}
```
```c
find_pid_of(const char *process_name)  
```
根据进程名字找到pid
通过查看 /proc/进程PID/cmdline文件的内容，与process_name进行对比，当相同时即找到了该进程的pid，并返回

### 通过ptrace注入
```c
// 1、获取待注入进程的pid，即获取/system/bin/surfaceflinger进程的pid
pid_t target_pid;  
target_pid = find_pid_of("/system/bin/surfaceflinger"); 

// 2、attach到远程进程
if (ptrace_attach(target_pid) == -1)    
    goto exit;  

// 3、保存寄存器环境
struct pt_regs regs, original_regs;
if (ptrace_getregs(target_pid, &regs) == -1)    
    goto exit; 
/* save original registers */    
memcpy(&original_regs, &regs, sizeof(regs));   

// 4、远程调用mmap来远程分配内存空间
// 4.1 获取到远程mmap函数的地址
const char *libc_path = "/system/lib/libc.so";
mmap_addr = get_remote_addr(target_pid, libc_path, (void *)mmap);
// 4.2 构造mmap参数，并远程调用mmap函数
parameters[0] = 0;  // addr    
parameters[1] = 0x4000; // size    
parameters[2] = PROT_READ | PROT_WRITE | PROT_EXEC;  // prot    
parameters[3] =  MAP_ANONYMOUS | MAP_PRIVATE; // flags    
parameters[4] = 0; //fd    
parameters[5] = 0; //offset     

if (ptrace_call(target_pid, (uint32_t)mmap_addr, parameters, 6, regs) == -1)    
        return -1;    
if (ptrace_getregs(target_pid, regs) == -1)    
    return -1;     
// 4.3、得到分配的内存空间的初始地址，并将注入模块写入该内存
lchar *library_path = "/data/libhello.so";
map_base = ptrace_retval(&regs);
ptrace_writedata(target_pid, map_base, library_path, strlen(library_path) + 1);  

// 5、远程打开/data/libhello.so模块
// 5.1 获取到远程dlopen函数的地址
const char *linker_path = "/system/bin/linker";  
dlopen_addr = get_remote_addr( target_pid, linker_path, (void *)dlopen );  
// 5.2、构造dlopen参数，并远程调用dlopen函数

parameters[0] = map_base;       
parameters[1] = RTLD_NOW| RTLD_GLOBAL;   

if (ptrace_call(target_pid, (uint32_t)dlopen_addr, parameters, 2, regs) == -1)    
    return -1;

void * sohandle = ptrace_retval(&regs);

// 6、远程调用dlsym函数来实现对注入的libhello.so模块的hook_entry方法的调用
const char * func_name = "hook_entry";
#define FUNCTION_NAME_ADDR_OFFSET       0x100    
ptrace_writedata(target_pid, map_base + FUNCTION_NAME_ADDR_OFFSET, func_name, strlen(func_name) + 1);    
parameters[0] = sohandle;       
parameters[1] = map_base + FUNCTION_NAME_ADDR_OFFSET;  

dlsym_addr = get_remote_addr( target_pid, linker_path, (void *)dlsym );    
if (ptrace_call(target_pid, (uint32_t)dlsym_addr, parameters, param_num, regs) == -1)    
    return -1;  
void * hook_entry_addr = ptrace_retval(&regs);   

// 7、关闭远程调用，恢复寄存器环境并detach进程
parameters[0] = sohandle;       

dlclose_addr = get_remote_addr( target_pid, linker_path, (void *)dlclose );
if (ptrace_call(target_pid, (uint32_t)dlclose_addr, parameters, 1, regs) == -1)    
    return -1;  
/* restore */    
ptrace_setregs(target_pid, &original_regs);    
ptrace_detach(target_pid);
```


## pt_regs结构体
### i386
`xref: /arch/x86/include/uapi/asm/ptrace.h`
```
struct pt_regs {
18	long ebx;
19	long ecx;
20	long edx;
21	long esi;
22	long edi;
23	long ebp;
24	long eax;
25	int  xds;
26	int  xes;
27	int  xfs;
28	int  xgs;
29	long orig_eax;
30	long eip;
31	int  xcs;
32	long eflags;
33	long esp;
34	int  xss;
};
```
### arm
`xref: /arch/arm/include/uapi/asm/ptrace.h`
```
#ifndef __KERNEL__
124struct pt_regs {
125	long uregs[18];
126};
127#endif /* __KERNEL__ */
128
129#define ARM_cpsr	uregs[16]
130#define ARM_pc		uregs[15]
131#define ARM_lr		uregs[14]
132#define ARM_sp		uregs[13]
133#define ARM_ip		uregs[12]
134#define ARM_fp		uregs[11]
135#define ARM_r10		uregs[10]
136#define ARM_r9		uregs[9]
137#define ARM_r8		uregs[8]
138#define ARM_r7		uregs[7]
139#define ARM_r6		uregs[6]
140#define ARM_r5		uregs[5]
141#define ARM_r4		uregs[4]
142#define ARM_r3		uregs[3]
143#define ARM_r2		uregs[2]
144#define ARM_r1		uregs[1]
145#define ARM_r0		uregs[0]
146#define ARM_ORIG_r0	uregs[17]
```


# hook 步骤
## ELF 
  ELF = executable linkable format   可执行、链接格式
       linux中的中ELF文件主要包括四类， 也即：
       1.  a.out
       2.  core
       3.  so文件
       4.  .o文件
通过file 命令可以查看
## ELF头文件 结构学习
![](http://blog.chinaunix.net/photo/94212_101201164532.jpg)
1. ELF头文件
    位于文件最开始处，包含整个文件的结构信息。
2. 节(section)
    是专门用于连接过程而言的，在每个节中包含指令数据、符号数据、重定位数据等等。
3. 程序头表
    在运行过程中是必须的，在链接过程中是可选的，因为它的作用是告诉系统如何创建进程的映像。
4. 节头表
    包含文件中所用节的信息。 
    
 
### 头-结构体Elf32_Ehdr
```cpp
typedef struct elf32_hdr{
  unsigned char e_ident[EI_NIDENT]; //16字节的信息，目标文件标识 也是结构体
  Elf32_Half e_type;  //目标文件类型 
  Elf32_Half e_machine;  //体系结构类型
  Elf32_Word e_version;  //目标文件版本
  Elf32_Addr e_entry; /* Entry point 程序入口的虚拟地址*/
  Elf32_Off e_phoff;  //程序头部表的偏移量
  Elf32_Off e_shoff;  //节区头部表的偏移量
  Elf32_Word e_flags;  //
  Elf32_Half e_ehsize;  //ELF头部的大小
  Elf32_Half e_phentsize;  //程序头部表的表项大小
  Elf32_Half e_phnum;  //程序头部表的数目
  Elf32_Half e_shentsize; //节区头部表的表项大小
  Elf32_Half e_shnum; //节区头部表的数目
  Elf32_Half e_shstrndx; //字符串表在节区中的索引
 } Elf32_Ehdr;  //此结构体一共52个字节
```

### 节-结构体Elf32_Shdr
![](http://img.blog.csdn.net/20150111161704826?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG9uZ2RlbmdodWk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### .rel.plt 和 .dynsym结构体
typedef struct{
Elf32_Addr  r_offset;
Elf32_Word  r_info;
} Elf32_Rel;

typedef structelf32_sym{
     Elf32_Word    st_name;
     Elf32_Addr    st_value;
     Elf32_Word    st_size;
     unsignedcharst_info;
     unsignedcharst_other;
     Elf32_Half     st_shndx;
} Elf32_Sym;

 

## PLT 与 GOT
PLT（Procedure Linkage Table）的作用是将位置无关的符号转移到绝对地址。当一个外部符号被调用时，PLT 去引用 GOT 中的其符号对应的绝对地址，然后转入并执行。

GOT（Global Offset Table）：用于记录在 ELF 文件中所用到的共享库中符号的绝对地址。在程序刚开始运行时，GOT 表项是空的，当符号第一次被调用时会动态解析符号的绝对地址然后转去执行，并将被解析符号的绝对地址记录在 GOT 中，第二次调用同一符号时，由于 GOT 中已经记录了其绝对地址，直接转去执行即可（不用重新解析）。


.rel.dyn记录了加载时需要重定位的变量，.rel.plt记录的是需要重定位的函数













 