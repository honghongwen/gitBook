#  IO模型

### 概念 什么是io

> 在unix中，认为一切皆文件，而文件又是一串二进制流。对于文件的读写，就是对这串二进制流的读写，也叫input、output。


### io交互流程

> io的交互其实是分两阶段的，他有一个用户空间和内核空间的概念。首先是经过内核空间，这里会做一个缓冲。之后才会从内核空间拷贝到用户空间。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1635932159714-e754504b-d141-4747-bcfb-88a59d9f8e7e.png#clientId=ua15b185c-acde-4&from=paste&height=886&id=u26de5024&name=image.png&originHeight=886&originWidth=921&originalType=binary&ratio=1&size=280780&status=done&style=none&taskId=ud49dafe3-db9d-4058-ad4c-d50973a2218&width=921)
### 5种io通信模型

#### 1. 阻塞io模型

![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1635932444605-cee56ade-0eee-4023-8fb8-3e111f22d8f7.png#clientId=ua15b185c-acde-4&from=paste&height=803&id=u763855eb&name=image.png&originHeight=803&originWidth=978&originalType=binary&ratio=1&size=188986&status=done&style=none&taskId=u9d773935-99ae-4acf-967a-11514ce2c61&width=978)

#### 2. 非阻塞io模型
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1635932647269-3a34609d-af3c-47ad-a0f4-a53fa1792e95.png#clientId=ua15b185c-acde-4&from=paste&height=867&id=ue135ea2b&name=image.png&originHeight=867&originWidth=891&originalType=binary&ratio=1&size=228268&status=done&style=none&taskId=ud683f2e9-7df1-4eab-ba97-8b0d90ba7f7&width=891)

#### 3. 多路复用io模型
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1635933350829-792809e2-aeb1-4847-844d-0efc6cf68844.png#clientId=ua15b185c-acde-4&from=paste&height=879&id=u2eab3431&name=image.png&originHeight=879&originWidth=834&originalType=binary&ratio=1&size=273407&status=done&style=none&taskId=u2a493dfb-c930-4a88-b160-568971b9e11&width=834)

#### 4. 信号驱动io模型（应用较少）
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1635933625732-cdb3c343-ea32-488d-9703-eab23f6e3886.png#clientId=ua15b185c-acde-4&from=paste&height=636&id=uca0e832b&name=image.png&originHeight=636&originWidth=824&originalType=binary&ratio=1&size=176332&status=done&style=none&taskId=ub46f054b-002b-47af-b067-c9fab6a39f2&width=824)

#### 5. 异步io模型
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1635934815628-cf50bf3a-670d-4736-87dc-989981a92860.png#clientId=ua15b185c-acde-4&from=paste&height=828&id=ubc854aec&name=image.png&originHeight=828&originWidth=819&originalType=binary&ratio=1&size=217981&status=done&style=none&taskId=uec8b4263-4552-42f5-8861-b911a848f1e&width=819)


