#  IO模型

### 概念 什么是io

> 在unix中，认为一切皆文件，而文件又是一串二进制流。对于文件的读写，就是对这串二进制流的读写，也叫input、output。


### io交互流程

> io的交互其实是分两阶段的，他有一个用户空间和内核空间的概念。首先是经过内核空间，这里会做一个缓冲。之后才会从内核空间拷贝到用户空间。

![image](/images/IoModel1.png)
### 5种io通信模型

#### 1. 阻塞io模型

![image](/images/IoModel2.png)

#### 2. 非阻塞io模型
![image](/images/IoModel3.png)

#### 3. 多路复用io模型
![image](/images/IoModel4.png)

#### 4. 信号驱动io模型（应用较少）
![image](/images/IoModel5.png)

#### 5. 异步io模型
![image](/images/IoModel6.png)


