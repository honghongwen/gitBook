# JVM历史

### 1. Sun Classic VM
> 这是世界上第一款商用Java虚拟机，在JDK1.2之前，用户java -version输出的都是Classic VM。
> 它只能用解释器来执行Java代码，如果想使用即时编译技术，必须使用外挂法，但是如果使用了即时编译技术，那解释器就停止工作。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1633941823292-9b14e118-7fdf-45de-ac83-f04f64d07234.png#clientId=udefba290-aa46-4&from=paste&height=416&id=u8b21b569&name=image.png&originHeight=416&originWidth=906&originalType=binary&ratio=1&size=80323&status=done&style=none&taskId=uafbec9b5-67a2-4349-84e7-1ec724547d6&width=906)
-- 深入理解Java虚拟机

#### 问题
> 由于是解释器和编译器不能共同工作，所以编译器的响应压力很大，不敢应用编译耗时稍高的优化，所以他的执行效率和传统c/c++有很大差距，也就是Java慢的由来。


### 2. Exact VM
> 出现的时间并不长，商业上仅出现了很短的时间就被HotSpot VM替代了。但是它的出现已经具备了很多高性能虚拟机雏形，如热点探针、编译器和解释器混合工作等。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1633943256298-a4da17b7-09ea-4e60-a68c-2240c67997b6.png#clientId=udefba290-aa46-4&from=paste&height=770&id=u953723cb&name=image.png&originHeight=770&originWidth=881&originalType=binary&ratio=1&size=210189&status=done&style=none&taskId=ucc40c894-957d-4724-8a78-934894cb4d2&width=881)

### 3. HotSpot VM
> 使用最广的Java虚拟机，能找出最具编译价值的热点代码，通过即时编译器和解释器共同工作，可以在最优响应时间和最佳性能中取得平衡。同时取消了永久代。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1633944730307-a08861e8-9623-4c77-992c-730d9b30d50a.png#clientId=udefba290-aa46-4&from=paste&height=1006&id=u83344cd1&name=image.png&originHeight=1006&originWidth=937&originalType=binary&ratio=1&size=284442&status=done&style=none&taskId=ufe25282d-ae16-4336-8fc1-aa37109aad1&width=937)
### 4. BEA JRockit/IBM J9
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1633946226523-cfb41273-75f9-4c62-ba6a-6970d2190566.png#clientId=udefba290-aa46-4&from=paste&height=991&id=u9ddd6635&name=image.png&originHeight=991&originWidth=854&originalType=binary&ratio=1&size=282510&status=done&style=none&taskId=u3e4c2f75-b2f7-4d7a-8945-bdcdafa38ed&width=854)
### 5. Graal VM
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1633947413635-5de2380d-dcee-4f34-be1f-f83cbebfdfd1.png#clientId=udefba290-aa46-4&from=paste&height=995&id=u695ba788&name=image.png&originHeight=995&originWidth=869&originalType=binary&ratio=1&size=284389&status=done&style=none&taskId=udbf067df-1405-41b1-88dc-e4f17143aec&width=869)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1633947723429-98034ad8-b5e9-47de-8539-09d1baaa1716.png#clientId=udefba290-aa46-4&from=paste&height=1000&id=uacd925f4&name=image.png&originHeight=1000&originWidth=864&originalType=binary&ratio=1&size=264723&status=done&style=none&taskId=ubfe49895-60c0-45f4-9c21-5abd682b3e0&width=864)

---


##### 历史惊人相似
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1633946805490-6a8b1d12-f017-411a-8b95-db2da407ffbc.png#clientId=udefba290-aa46-4&from=paste&height=741&id=u37f5db5d&name=image.png&originHeight=741&originWidth=933&originalType=binary&ratio=1&size=189545&status=done&style=none&taskId=u780667d0-bc09-4170-afaf-d8c4855b21c&width=933)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1633946907582-2ad46265-a6cc-4c9a-ae08-7cd146379e9f.png#clientId=udefba290-aa46-4&from=paste&height=989&id=u0a9cf7ae&name=image.png&originHeight=989&originWidth=916&originalType=binary&ratio=1&size=274492&status=done&style=none&taskId=u885c24a0-0d67-4625-99bb-74b22f66c1e&width=916)
