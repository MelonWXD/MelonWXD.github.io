---
title: 注入之道(二) so文件加载流程 
date: 2017-11-28 20:08:47   
tags: [注入] 
categories: Android逆向安全  
---
乱贴源码系列：结合Android6.0源码[linker.cpp](http://androidxref.com/6.0.1_r10/xref/bionic/linker/linker.cpp)来学习一下so库的加载流程
<!-- more -->
# so的加载和链接
## 整体流程
### System.load
通常我们要在代码中使用so库的话，都要在静态块手动的去载入so库，形如：
```
    static {
        System.loadLibrary("native-lib");
    }
```
跟进代码，会一直走到`java.lang.Runtime`包下的native方法
`nativeLoad`,再到源码中的[java_lang_Runtime.cc](http://androidxref.com/6.0.1_r10/xref/art/runtime/native/java_lang_Runtime.cc#70)中的`Runtime_nativeLoad`,然后跳转到[java_vm_ext.cc](http://androidxref.com/6.0.1_r10/xref/art/runtime/java_vm_ext.cc#596)的`LoadNativeLibrary`，在line645调用了dlopen:
```
643  Locks::mutator_lock_->AssertNotHeld(self);
644  const char* path_str = path.empty() ? nullptr : path.c_str();
645  void* handle = dlopen(path_str, RTLD_NOW);
```
看到dlopen基本就是跳转到linker下执行了，`dlopen`是linker目录下的[dlfcn.cpp](http://androidxref.com/6.0.1_r10/xref/bionic/linker/dlfcn.cpp)的函数:
```
70	static void* dlopen_ext(const char* filename, int flags, const android_dlextinfo* extinfo) {
71	  ScopedPthreadMutexLocker locker(&g_dl_mutex);
72	  soinfo* result = do_dlopen(filename, flags, extinfo);
73	  if (result == nullptr) {
74  	  __bionic_format_dlerror("dlopen failed", linker_get_error_buffer());
75 	   return nullptr;
76 	 }
77 	 return result;
78	 }
79
80	void* android_dlopen_ext(const char* filename, int flags, const android_dlextinfo* extinfo) {
81	  return dlopen_ext(filename, flags, extinfo);
82	}
83
84	void* dlopen(const char* filename, int flags) {
85	  return dlopen_ext(filename, flags, nullptr);
86	}
```
自下往上，最后调用到[linker.cpp](http://androidxref.com/6.0.1_r10/xref/bionic/linker/linker.cpp#1676)的`do_dlopen`。
### do_dlopen
```
	  ...
1694  ProtectedDataGuard guard;
1695  soinfo* si = find_library(name, flags, extinfo);
1696  if (si != nullptr) {
1697    si->call_constructors();
1698  }
1699  return si;
```
前面做了一堆非空值的判断，然后调用了一个`ProtectedDataGuard`类的`protect_data`方法
```
791class ProtectedDataGuard {
792 public:
793  ProtectedDataGuard() {
794    if (ref_count_++ == 0) {
795      protect_data(PROT_READ | PROT_WRITE);
796    }
797  }
798
799  ~ProtectedDataGuard() {
800    if (ref_count_ == 0) { // overflow
801      __libc_fatal("Too many nested calls to dlopen()");
802    }
803
804    if (--ref_count_ == 0) {
805      protect_data(PROT_READ);
806    }
807  }
808 private:
809  void protect_data(int protection) {
810    g_soinfo_allocator.protect_all(protection);
811    g_soinfo_links_allocator.protect_all(protection);
812  }
813
814  static size_t ref_count_;
815};
816
```
最后通过`LinkerTypeAllocator`调用到`LinkerBlockAllocator`的`protect_all`，实际上调的是系统函数`mprotect`来修改页的读写属性。
```
89void LinkerBlockAllocator::protect_all(int prot) {
90  for (LinkerBlockAllocatorPage* page = page_list_; page != nullptr; page = page->next) {
91    if (mprotect(page, PAGE_SIZE, prot) == -1) {
92      abort();
93    }
94  }
95}
```
ok，回归正文，刚才`do_dlopen`最主要一行代码，通过find_library就完成了so的加载，返回一个结构体soinfo，然后调用这个soinfo的call_constructors()方法，即so库的构造函数，来完成so的加载。
```
1695  soinfo* si = find_library(name, flags, extinfo);
1696  if (si != nullptr) {
1697    si->call_constructors();
1698  }
```
### soinfo
在进入find_library之前先介绍一下表示一个so库的信息的结构体[soinfo](http://androidxref.com/6.0.1_r10/xref/bionic/linker/linker.h#172)  
我很想把代码都贴上来，但是那样凑字数意图太明显，我还是贴个简单版的
```
struct soinfo{
    char old_name_[SOINFO_NAME_LEN];
    const ElfW(Phdr)* phdr;
    size_t phnum;
    ElfW(Addr) entry;
    ElfW(Addr) base;
    size_t size;
    ElfW(Dyn)* dynamic;

    //以下与.hash表有关
    unsigned nbucket;
    unsigned nchain;
    unsigned *bucket;
    unsigned *chain;
    
    //以下与.gnu_hash表有关
    size_t gnu_nbucket_;
    uint32_t* gnu_bucket_;
    uint32_t* gnu_chain_;
    uint32_t gnu_maskwords_;
    uint32_t gnu_shift2_;
    ElfW(Addr)* gnu_bloom_filter_;
}
```
`phdr`在之前的ELF文件格式执行视图中已经介绍过了，就是程序头表，dynamic就是动态节区，特别注意有2个hash表相关的，这个放到后续我们hook的时候会再细说，主要是有个印象，知道符号查找不一定是根据hash表来，还有个gnuhash。

### find_library
内部调用`find_libraries`->`find_library_internal`->`load_library`->`load_library`，做一系加载，预链接，已加载判断等，在6个参数的[load_library](http://androidxref.com/6.0.1_r10/xref/bionic/linker/linker.cpp#1252)中开始通过ElfReader这个类来读取so文件信息并赋值到soinfo中来实现so文件加载到内存中。
```
1303  // Read the ELF header and load the segments.
1304  ElfReader elf_reader(realpath.c_str(), fd, file_offset, file_stat.st_size);
1305  if (!elf_reader.Load(extinfo)) {
1306    return nullptr;
1307  }
1308
1309  soinfo* si = soinfo_alloc(realpath.c_str(), &file_stat, file_offset, rtld_flags);
1310  if (si == nullptr) {
1311    return nullptr;
1312  }
1313  si->base = elf_reader.load_start();
1314  si->size = elf_reader.load_size();
1315  si->load_bias = elf_reader.load_bias();
1316  si->phnum = elf_reader.phdr_count();
1317  si->phdr = elf_reader.loaded_phdr();
```

## 加载
### ElfReader
ElfReader的定义在[linker_phdr](http://androidxref.com/6.0.1_r10/xref/bionic/linker/linker_phdr.cpp#136)中，就是通过解析so库的执行视图，来读取so文件的各种信息，因为后续介绍的hook的时候也是根据这里的思路来进行的。
### Load
根据上面的`load_library`代码，可以看到先执行了ElfReader的Load方法，如下：
```
149bool ElfReader::Load(const android_dlextinfo* extinfo) {
150  return ReadElfHeader() &&
151         VerifyElfHeader() &&
152         ReadProgramHeader() &&
153         ReserveAddressSpace(extinfo) &&
154         LoadSegments() &&
155         FindPhdr();
156}
```
很完美，读取头文件-验证-读取程序头-分配内存空间-加载段区-查找程序头表项。
每个步骤在参考中的**Linker与So加壳技术**都有说明，规则都很死，读取文件信息，找偏移，读取正确位置即可，我这里只强调一下分配空间这个步骤。
```
307bool ElfReader::ReserveAddressSpace(const android_dlextinfo* extinfo) {
308  ElfW(Addr) min_vaddr;
309  load_size_ = phdr_table_get_load_size(phdr_table_, phdr_num_, &min_vaddr);


315  uint8_t* addr = reinterpret_cast<uint8_t*>(min_vaddr);
316  void* start;

341  int mmap_flags = MAP_PRIVATE | MAP_ANONYMOUS;
342  start = mmap(mmap_hint, load_size_, PROT_NONE, mmap_flags, -1, 0);

351  load_start_ = start;
352  load_bias_ = reinterpret_cast<uint8_t*>(start) - addr;
353  return true;
354}
```
这段代码先是计算了so库在内存中需要的空间，然后调用系统函数`mmap`来映射。
在line352有个变量`load_bias_`，他的值是start-addr，这个addr的值是`min_vaddr`，这是so库在加载的时候指定的加载基址，通常来说这个值为0，对，通常来说。
在Android4.4及其之前的版本，load_bias_=load_start，但是在Anroid6.0（5.0没有机子测试）之后，min_vaddr不为0，所以load_bias_要做一个修正：
```
load_bias_ = reinterpret_cast<uint8_t*>(start) - addr;
```
这个在后续的hook中会进一步说明。

## 链接
在加载完毕之后，就要通过[perlink_image](http://androidxref.com/6.0.1_r10/xref/bionic/linker/linker.cpp#2499)方法来链接。
```
2499bool soinfo::prelink_image() {
2500  /* Extract dynamic section */
2501  ElfW(Word) dynamic_flags = 0;
2502  phdr_table_get_dynamic_section(phdr, phnum, load_bias, &dynamic, &dynamic_flags);

2527  // Extract useful information from dynamic section.
2528  // Note that: "Except for the DT_NULL element at the end of the array,
2529  // and the relative order of DT_NEEDED elements, entries may appear in any order."
2530  //
2531  // source: http://www.sco.com/developers/gabi/1998-04-29/ch5.dynamic.html
2533  for (ElfW(Dyn)* d = dynamic; d->d_tag != DT_NULL; ++d) {
		...	
      }
```
### 定位动态节区
通过`phdr_table_get_dynamic_section`来遍历phdr，程序头表，找到type为`PT_DYNAMIC`的动态节区。
### 遍历动态节区
找到动态节区之后，遍历来根据节区的结构体成员d_tag，确定需要用到的各种参数，各种节区，别如DT_PLTRELSZ就是与重定位有关的，DT_HASH DT_GNU_HASH就是和符号表索引相关的，DT_INIT就是初始化相关的节。
### 重定位
解析完信息之后，就要遍历重定位表，来重定位每个符号的实际地址。
占坑，放在（三）中和hook原理一起讲解。

# 参考
[Android Linker 与 SO 加壳技术](http://dev.qq.com/topic/57e3a3bc42eb88da6d4be143)
