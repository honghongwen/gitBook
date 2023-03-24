```java
    @Test
    void testCreatFileWithOIO() throws IOException {
        File file = new File("D:\\temp\\temp.sh");
        if (!file.exists()) {
            boolean flag = file.createNewFile();
        }
        FileOutputStream fos = new FileOutputStream(file);
        String hello = "nihaoa";
        fos.write(hello.getBytes(StandardCharsets.UTF_8));
        fos.flush();
        fos.close();


        FileInputStream fis = new FileInputStream(file);
        byte[] buffer = new byte[100];
        StringBuilder sb = new StringBuilder();
        int readLength = 100;
        while (readLength == 100) {
            readLength = fis.read(buffer);
            sb.append(new String(buffer, 0, readLength));
        }
        log.info("读取的结果为：{}", sb.toString());
    }

    @Test
    void testCreateFile() throws Exception {
        File file = new File("D:\\temp\\temp.txt");
        if (!file.exists()) {
            return;
        }
        FileInputStream fis = new FileInputStream(file);
        FileChannel fileChannel = fis.getChannel();
        int length = 100;
        ByteBuffer buffer = ByteBuffer.allocate(100);
        StringBuilder sb = new StringBuilder();
        while (length == 100) {
            length = fileChannel.read(buffer);
            log.info("读取的字节长度为：{}", length);
            byte[] bytes = buffer.array();
            sb.append(new String(bytes, 0, length));
        }
        log.info("字符串为：{}", sb.toString());
        fileChannel.close();
    }
```
