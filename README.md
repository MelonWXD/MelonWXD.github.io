# MelonWXD.github.io

TCP连接过程、释放连接过程



# 分层

应用层：HTTP FTP DNS

传输层：TCP UDP

网络层：IP ARP

数据链路层：硬件驱动



网关用于跨子网传输

网关和路由的区分 

现代路由器大都包含了网关的功能



ARP协议 缓存ip和mac  还有 RARP

NAT映射

DNS域名解析  是由路由器去查询的

## 建立tcp

### 三次握手

-> 标志位：SYN    seq=N

<- 标志位SYN ACK    seq=M  ack=N+1

-> 标志位：ACK    seq=N+1 ack=M+1

### 四次挥手  要注意响应FIN时的ack值=seq+1  服务器响应之后主动发一次结束 ack跟上一次一样

-> FIN ACK  seq=A ack=B   发起方进入FIN_WAIT1状态  接收方收到后进入CLOSE_WAIT

<- ACK seq=B ack=A+1   发起方收到后进入 FIN_WAIT2

<- FIN ACK seq =C ack=A+1  接收方进入LAST_ACK  发起方收到后进入 TIME_WAIT

-> ACK seq=A+1 ack=C+1  发起方发送之后进入超时等待

![](http://img.blog.csdn.net/20150624175234142)

## 拥塞控制和滑动窗口





## 浏览器打开网页发生的网络过程

- DNS 把域名转换为ip地址：先查浏览器自身的dns缓存(如果有的话)，再查系统定义的host，还没有的话就像默认的DNS服务器发起请求，默认的DNS服务器如果没有的话自己会向 13台根服务器发出请求，根据.com .cn一步步递归查找直到返回 dstIP
- 拿到 dstIP ，建立tcp连接
- 建立HTTP 拉取网页的css js 资源 
- 浏览器渲染

## https

ssl tls

CA证书是保存服务器RSA公钥的 

客户端随机生成数字 当成AES的秘钥   把随机数通过RSA加密传输来告诉服务器



## Http常见包头

date

content-type

content-length

Connection:
