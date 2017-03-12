Java采用UTF-16编码作为内码，也就是说在JVM内部，文本是用16位码元序列表示的，常用的文本就是字符（char）和字符串（String）字面常量的内容。注：UTF-16是Unicode字符集的一种编码方案。

Java字符和字符串存在于以下几个地方：

- Java源码文件，*.java，可以是任意字符编码，如GBK，UTF-8
- Class文件，*.class，采用的是一种改进的UTF-8编码（Modified UTF-8）
- JVM，内存中使用UTF-16编码

Java编译器需要正确的读取源码，消除编码差异，然后编译成UTF-8编码的Class文件。比如javac，默认情况下它会取操作系统的编码，可以使用参数-encoding指定源码文件的字符编码。JVM加载Class文件，把其中的字符或字符串转成UTF-16编码序列。

Java中涉及编码的类主要有`String`和IO包中的字节字符转换流。`String.getBytes()`使用JVM启动时获得的字符集来编码字符串，也可以使用`getBytes(charset)`指定字符集；字节就是单纯的01，但转成字符时就要有字符集的概念了，IO包中的`InputStreamReader`和`OutputStreamWriter`，是字节流和字符流的桥梁，默认使用JVM默认字符集对字符解码和编码，可以通过构造方法指定字符集。

```java
String str = "创";
str.getBytes("UTF-8"); // 3字节，0xE5889B
str.getBytes("UTF-16"); // 2字节，0x521B

InputStreamReader(InputStream, charset);
OutputStreamWriter(OutputStream, charset);
```

## FAQ
### 1. Java中的字符主要有哪些？

Java编程语言主要有以下几种字符：
- 空白字符：空格、制表符、换页符和行终止符
- 注释：/*text*/ or // text
- 符号
- 标识符：就是变量名和类名，其中的字母和数字可以从Unicode字符集中提取，也就是说能用本地语言编写标识符，如`String 名字="cxcoder";`
- 关键字：比如class，new
- 字面常量：简单类型、String、空类型在源码中的表示
- 分隔符：也叫标点符号，`() {} [] ; , . ... @`
- 操作符：逻辑和算术运算符
- 转义字符
- Unicode转义字符：通过\u+4个十六进制数使用任何Unicode字符
- 字面常量转义字符：`\b \t \n \f \r \" \' \\`不使用Unicode转义字符也能表示一些特殊字符

### 2. 解释一下程序的输出结果

```java
String hello = "Hello", lo = "lo";
System out print(hello == "Hello"); // true 一个字符串字面常量总是引用String的同一个实例
System out print(hello == ("Hel" +"lo")): // true 常量表达式，编译时得出结果，当做字面常量对待
System out print(hello == ("Hel"+1o)); // false 运行时连接运算产生新String对象
System out println(hell0 == ("Hel"+lo).intern()); // true 查找常量池是否有此字符串，有返回，无放进去，之前已定义intern返回同一个String实例
```

### 3. Java中的null如何理解？
null看起来是关键字，但从技术上讲，它仅仅是空字面常量，表示空引用。像`true/false`也只是布尔字面常量。