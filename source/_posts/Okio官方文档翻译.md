---
title: Okio官方文档翻译
date: 2019-09-13 00:27:40
category:
- 开源库
    - okio
tags:
- okio
- okhttp
- 源码
---

# Okio官方文档翻译

`Okio`是一个辅助`java.io`和`java.nio`变得更易于访问、存储、操作数据的库。它一开始是作为`Okhttp`的组件存在。`Okhttp`是在`Android`中非常好用的`HTTP`客户端。`Okio`经过充分测试，并且准备好处理新的问题了。

## ByteStrings and Buffers

`Okio`围绕两种类型构建。将大量的功能集成到一个简单的`API`中：
* `ByteString`是不可更改的字节序列。对于字符数据，`String`是基础。`ByteString`则是`String`失散多年的兄弟，更易于将二进制数据作为值来处理。这个类非常好用：它知道如何编码和解码为16进制、`base64`和`UTF-8`。
* `Buffer`是可更改的字节序列。和`ArrayList`一样，我们不需要提前设置它的大小。应该像一个队列一样读取和写入`Buffer`：在末尾写入数据，从头部读取数据。不必考虑管理位置、长度限制和容量。

在内部，`ByteString`和`Buffer`可以节省`CPU`和内存。如果将`UTF-8`字符串转换为`ByteString`，它将保存这个字符串的引用，需要时再进行解码，无需做执行任何操作。

`Buffer`使用一个`Segment`链表实现。当我们将数据从一个`Buffer`移到另一个`Buffer`时，它重新分配`Segment`的所有权，而不是复制数据。这在多线程编程中特别有用：一个线程用于网络交换数据，与另一个工作线程交换数据时不必对数据进行复制。

## Sources and Sinks

`java.io`包设计非常好的一点是能够将如加密、压缩等转换进行分层。`Okio`包含了它自己的流类型，被称为`Source`和`Sink`。它用起来和`InputStream`、`OutputStream`差不多，但是有几点关键的不同：

* `Timeout`，超时。流提供底层`IO`机制的超时访问。和`java.io`套接字流不同，`read()`和`write()`调用都会有超时信息。
* 易于实现。`Source`定义了三个方法：`read()`，`close()`和`timeout()`。没有如`available()`或是单字节读取造成正确性和性能的危险。
* 易于使用。即是`Source`和`Sink`都只有三个方法用于读写，调用者可以通过使用`BufferedSource`和`BufferedSink`接口获得更多的`API`。
* 字节流和字符流本质上并没有任何区别，都是数据。`UTF-8`字符串、`big-endian`32位整数，`little-endian`短整数或是任何想要的数据，都可以被认为是字节读取和写入。不必再使用`InputStreamReader`了。
* 易于测试。`Buffer`类实现了`BufferedSource`和`BufferedSink`接口，所以测试代码可以非常简单和清晰。

`Source`、`Sink`可以和`InputStream`、`OutputStream`相互操作，你可以将任何`Source`视为`InputStream`，也可以将`InputStream`视为`Source`。`Sink`和`OutputStream`同理。

## 例子

官方写了一些例子来演示如何使用`Okio`来处理一些常见的问题。阅读学习如何使用他们。需要的话可以进行复制粘贴。

### 按行读取一个文本文件

使用`Okio.source(File)`打开一个文件。返回的`Source`接口非常轻量级，并且只有有限的方法。通常会使用一个`Buffer`来包装这个`Source`。有两个好处：

1. `API`更有用。不像`Source`只提供了基础的方法，`BufferedSource`有非常多的方法易于使用。
2. 编程更快。`Buffer`可以使用更少的`IO`操作来完成工作。

`Source`每次打开都需要关闭。打开流的代码负责保证它的关闭。这里我们使用`Java`的`try`块来自动关闭`Source`。

```java
public void readLines(File file) throws IOException {
  try (Source fileSource = Okio.source(file);
       BufferedSource bufferedSource = Okio.buffer(fileSource)) {

    while (true) {
      String line = bufferedSource.readUtf8Line();
      if (line == null) break;

      if (line.contains("square")) {
        System.out.println(line);
      }
    }

  }
}
```

`readUtf8Line()`方法读取一行数据（以换行符`\n`或`\r\n`分隔），或者直到文件末尾（无换行符）。返回一个字符串，省略末尾的换行符。如果遇到空行，将返回空字符串。如果到文件末尾，则返回`null`。

上面的代码可以内联`fileSource`变量使得代码更加紧凑，使用漂亮的`for`循环代替`while`循环：

```java
public void readLines(File file) throws IOException {
  try (BufferedSource source = Okio.buffer(Okio.source(file))) {
    for (String line; (line = source.readUtf8Line()) != null; ) {
      if (line.contains("square")) {
        System.out.println(line);
      }
    }
  }
}
```

`readUtf8Line()`方法适用于解析大多数文件。对于某些特定例子，可以考虑使用`readUtf8LineStrict()`。两者非常相似，但是`readUtf8LineStrice()`方法要求每行以换行符（`\n`或`\r\n`）结尾。在此之前遇到文件末尾，将抛出`EOFException`异常。`readUtf8LineStrict()`方法还允许字节数限制，用于防止输入格式错误。

```java
public void readLines(File file) throws IOException {
  try (BufferedSource source = Okio.buffer(Okio.source(file))) {
    while (!source.exhausted()) {
      String line = source.readUtf8LineStrict(1024L);
      if (line.contains("square")) {
        System.out.println(line);
      }
    }
  }
}
```

### 写入文件

上面我们使用了`Source`和`BufferedSource`读取一个文件。对于写入，我们则使用`Sink`和`BufferedSink`。使用`Buffer`的好处一样：更好的`API`和性能。

```java
public void writeEnv(File file) throws IOException {
  try (Sink fileSink = Okio.sink(file);
       BufferedSink bufferedSink = Okio.buffer(fileSink)) {

    for (Map.Entry<String, String> entry : System.getenv().entrySet()) {
      bufferedSink.writeUtf8(entry.getKey());
      bufferedSink.writeUtf8("=");
      bufferedSink.writeUtf8(entry.getValue());
      bufferedSink.writeUtf8("\n");
    }

  }
}
```

因为没有提供写入换行的`API`，所以我们手动插入换行字符。大多数时候使用`\n`作为换行字符。在很少情况下，可以使用`System.lineSeparator()`代替`\n`：`System.lineSeparator()`在`Windows`系统返回`\r\n`，而在其他系统则返回`\n`。

我们内联`fileSink`变量，并且利用链式调用来重写上面的方法：

```java
public void writeEnv(File file) throws IOException {
  try (BufferedSink sink = Okio.buffer(Okio.sink(file))) {
    for (Map.Entry<String, String> entry : System.getenv().entrySet()) {
      sink.writeUtf8(entry.getKey())
          .writeUtf8("=")
          .writeUtf8(entry.getValue())
          .writeUtf8("\n");
    }
  }
}
```

上面的代码，我们使用了`writeUtf8()`方法。调用四次`writeUtf8()`比下面的代码更高效，因为`VM`不用去对产生的临时字符串进行垃圾回收。


```java
sink.writeUtf8(entry.getKey() + "=" + entry.getValue() + "\n"); // Slower!
```

### UTF-8

从上面用到的`API`可以看出，`Okio`非常喜欢`UTF-8`。早期的电脑系统经历了各种字符编码：`ISO-8859-1`，`ShiftJIS`，`ASCII`，`EBCDIC`等等。为了支持多种字符集的编程非常糟糕，甚至，我们还要使用`emoji`。目前非常幸运的是，世界各地统一都支持`UTF-8`。只有很少一部分老系统还在使用其他字符集。

如果你需要另外的字符集，可以使用`readString()`和`writeString()`。这两个方法需要你指定字符集。除非数据只需要本机读取，否则大多数情况下，都应该使用`UTF-8`方法来编程。

当在解析字符串时，你需要记住字符串时如何表示和编码的。当一个字形有声调或是其他变形时，它代表一个单独的混合字符，如`é`，或是一个字符`e`接一个修饰符`´`。当整个字形是一个单独的字符时，它被称为`NFC`，当它是多个字符组成时，被称为`NFD`。

即使我们使用`IO`操作字符串时都使用`UTF-8`，他们在内存的存在形式是`Java String`，使用的是另一个变种`UTF-16`进行编码。这是非常糟糕的编码，因为它大部分字符都使用16字节字符，但是却用不到16个字节。特别是，`emoji`字符占两个`java`字符。这存在的问题是，`String.length()`返回不一样的结果：`UTF-16`的字符长度，并不是原生字形的长度。


| |Café 🍩|Café 🍩|
|---|---|---|
| Form | NFC | NFD |
| Code Points |	c  a  f  é    ␣   🍩 | c  a  f  e  ´    ␣   🍩 |
| UTF-8 bytes	| 43 61 66 c3a9 20 f09f8da9	| 43 61 66 65 cc81 20 f09f8da9 |
| String.codePointCount	| 6 |	7 |
| String.length | 7 |	8 |
| Utf8.size |	10	| 11 |

大部分情况，`Okio`可以让你忽略这些问题更关注数据。但是当你需要这些时，有一些方便的`API`处理低级的`UTF-8`字符串。

使用`Utf8.size()`来统计使用`UTF-8`编码时的字节长度。这在如协议缓冲等长度前缀编码时非常方便。

使用`BufferedSource.readUtf8CodePoint()`读取单个可变长度代码点，使用`BufferedSinkel.writeUtf8CodePoint()`写入。


### Golden Values

`Okio`喜欢测试。库已经经过了严格的测试，并且它有易于测试的特性。我们找到一种十分好用的测试模式，称为`golden value`测试。这种测试的目的是确认被早期版本的程序进行编码的数据能够安全的使用当前版本的程序进行解码。

我们将通过使用`Java Serialization`（`Java`序列化）编码来进行演示。题外话，我们一定要放弃糟糕的`Java`序列化，大多数编程我们应该使用其他编码，如`JSON`或`protobuf`。下面是对一个对象进行序列化的方法，返回一个`ByteString`：

```java
private ByteString serialize(Object o) throws IOException {
  Buffer buffer = new Buffer();
  try (ObjectOutputStream objectOut = new ObjectOutputStream(buffer.outputStream())) {
    objectOut.writeObject(o);
  }
  return buffer.readByteString();
}
```

这里对上面的代码进行解释：

1. 我们构造一个`Buffer`存储序列化后的数据。`Buffer`是`ByteArrayOutputStream`更好的替代者。
2. 我们获取`Buffer`的`OutputStream`，通过它，我们将数据写入`Buffer`中，并且总在`Buffer`结尾添加数据。
3. 构造一个`ObjectOutputStream`（`Java`序列化`API`）并且写入对象。`try`块为我们自动关闭流。注意关闭`Buffer`没有用。
4. 最后，调用`Buffer.readByteString()`方法，获取`ByteString`。这个方法允许我们指定读取的字节数，在这里我们不指定数量，获取整个字符串。从`Buffer`中读取数据，总是能保证数据从`Buffer`头部读取的。

通过上面的`serialize()`方法，我们准备好计算和打印`golden value`了。

```java
Point point = new Point(8.0, 15.0);
ByteString pointBytes = serialize(point);
System.out.println(pointBytes.base64());
```

打印出`ByteString`的`base64`值，是因为`base64`紧凑和便于嵌入测试用例的特性。程序打印如下：

```java
rO0ABXNyAB5va2lvLnNhbXBsZXMuR29sZGVuVmFsdWUkUG9pbnTdUW8rMji1IwIAAkQAAXhEAAF5eHBAIAAAAAAAAEAuAAAAAAAA
```

这就是我们的`golden value`了。我们再次使用`base64`键入到我们的测试用例中，将其转换回一个`ByteString`：

```java
ByteString goldenBytes = ByteString.decodeBase64("rO0ABXNyAB5va2lvLnNhbXBsZ"
    + "XMuR29sZGVuVmFsdWUkUG9pbnTdUW8rMji1IwIAAkQAAXhEAAF5eHBAIAAAAAAAAEAuA"
    + "AAAAAAA");
```

下一步，将`ByteString`反序列成我们需要的对象。这个方法和上面的`serialize()`方法相反：添加一个`ByteString`到`Buffer`中，然后使用一个`ObjectInputStream`进行读取。

```java
private Object deserialize(ByteString byteString) throws IOException, ClassNotFoundException {
  Buffer buffer = new Buffer();
  buffer.write(byteString);
  try (ObjectInputStream objectIn = new ObjectInputStream(buffer.inputStream())) {
    return objectIn.readObject();
  }
}
```

下面我们对`golden value`进行解码测试：

```java
ByteString goldenBytes = ByteString.decodeBase64("rO0ABXNyAB5va2lvLnNhbXBsZ"
    + "XMuR29sZGVuVmFsdWUkUG9pbnTdUW8rMji1IwIAAkQAAXhEAAF5eHBAIAAAAAAAAEAuA"
    + "AAAAAAA");
Point decoded = (Point) deserialize(goldenBytes);
assertEquals(new Point(8.0, 15.0), decoded);
```

通过这里的测试，我们可以在不破坏兼容性的情况下改变`Point`类的序列化。

### 写入二进制文件

对二进制文件编码和对文本文件编码没有什么不同。`Okio`同样使用`BufferedSink`和`BufferedSource`进行操作。这对那些既包含字节，有包含字符的二进制编码非常友好。

写入二进制数据比写入文字更加的危险，这是因为非常难诊断出出错的地方。围绕这些问题，可以通过注意下面几点来尽量避免：

1. 每个域的宽度。这是使用的字节数量。`Okio`不包含发射部分字节的机制。如果需要这个功能，那么就需要在写入前自己进行位移和屏蔽。
2. 每个域的字节序。所有超过一个字节的域都有字节序：字节是否经过了排序（从大到小的`big endian`和从小到达的`little endian`）。`Okio`使用在方法名加后缀`Le`的方式表示使用`little-endian`的方法；没有后缀则是使用`big-endian`。
3. 是否带符号。`Java`没有不带符号的基础类型（除了`char`），所以对于这种，通常是在应用层进行处理。为了保证更简单，`Okio`在`writeByte()`和`writeShort()`接收`int`类型。这样你就可以传入一个无符号的`byte`如255（`byte`值最大是127），`Okio`可以保证写入正确。

解释：对于上面的第二点，如果写入一个`int`值3，如果是`big endian`，那么写入的则是`00 00 00 03`，如果是`little endian`则是`03 00 00 00`。对于上面的第三点，如果写入的一个值是255，如果是`Java`中的`byte`类型，它最大值是127，并不能写入`255`，所以`Okio`可以使用`writeShort()`方法写入`int`值，保证一个字节可以表示`255`。

| Method |	Width |	Endianness |	Value |	Encoded Value |
| --- | --- | --- | --- | --- |
| writeByte |	1 |	 |3 |	03 |
| writeShort | 2 |	big |	3 |	00 03 |
| writeInt |	4 |	big |	3 |	00 00 00 03 |
| writeLong |	8 |	big |	3 | 00 00 00 00 00 00 00 03 |
| writeShortLe |	2 |	little |	3 |	03 00 |
| writeIntLe |	4 |	little |	3 |	03 00 00 00 |
| writeLongLe |	8 |	little |	3 |	03 00 00 00 00 00 00 00 |
| writeByte |	1 |	 |	Byte.MAX_VALUE |	7f |
| writeShort |	2 |	big |	Short.MAX_VALUE |	7f ff |
| writeInt |	4	| big |	Int.MAX_VALUE |	7f ff ff ff |
| writeLong |	8 |	big |	Long.MAX_VALUE |	7f ff ff ff ff ff ff ff |
| writeShortLe |	2 |	little |	Short.MAX_VALUE |	ff 7f |
| writeIntLe |	4 |	little |	Int.MAX_VALUE |	ff ff ff 7f |
| writeLongLe |	8 |	little |	Long.MAX_VALUE |	ff ff ff ff ff ff ff 7f |

下面的代码将一个`bitmap`写入成一个`BMP`的文件格式。

```java
void encode(Bitmap bitmap, BufferedSink sink) throws IOException {
  int height = bitmap.height();
  int width = bitmap.width();

  int bytesPerPixel = 3;
  int rowByteCountWithoutPadding = (bytesPerPixel * width);
  int rowByteCount = ((rowByteCountWithoutPadding + 3) / 4) * 4;
  int pixelDataSize = rowByteCount * height;
  int bmpHeaderSize = 14;
  int dibHeaderSize = 40;

  // BMP Header
  sink.writeUtf8("BM"); // ID.
  sink.writeIntLe(bmpHeaderSize + dibHeaderSize + pixelDataSize); // File size.
  sink.writeShortLe(0); // Unused.
  sink.writeShortLe(0); // Unused.
  sink.writeIntLe(bmpHeaderSize + dibHeaderSize); // Offset of pixel data.

  // DIB Header
  sink.writeIntLe(dibHeaderSize);
  sink.writeIntLe(width);
  sink.writeIntLe(height);
  sink.writeShortLe(1);  // Color plane count.
  sink.writeShortLe(bytesPerPixel * Byte.SIZE);
  sink.writeIntLe(0);    // No compression.
  sink.writeIntLe(16);   // Size of bitmap data including padding.
  sink.writeIntLe(2835); // Horizontal print resolution in pixels/meter. (72 dpi).
  sink.writeIntLe(2835); // Vertical print resolution in pixels/meter. (72 dpi).
  sink.writeIntLe(0);    // Palette color count.
  sink.writeIntLe(0);    // 0 important colors.

  // Pixel data.
  for (int y = height - 1; y >= 0; y--) {
    for (int x = 0; x < width; x++) {
      sink.writeByte(bitmap.blue(x, y));
      sink.writeByte(bitmap.green(x, y));
      sink.writeByte(bitmap.red(x, y));
    }

    // Padding for 4-byte alignment.
    for (int p = rowByteCountWithoutPadding; p < rowByteCount; p++) {
      sink.writeByte(0);
    }
  }
}
```

这段代码最麻烦的部分是格式需要填充。`BMP`格式期望一行以4个字节的边界开始，所以添加0值进行对齐非常重要。

对其他二进制格式进行编码通常非常相似。下面是一些建议：

1. 使用`golden values`编写测试代码。确保程序发送期望的结果，可以让测试更加的简单。
2. 使用`Utf8.size()`计算一个编码的字符串的字节长度。这对长度前缀的格式非常有效。
3. 使用`Float.floatToIntBits()`和`Double.doubleToLongBits()`对浮点值进行编码。

### 通过Socket进行通信

通过网络发送和接受数据，和写入和读取文件非常相似，使用`BufferedSink`对输出值编码，使用`BufferedSource`对输入值编码。和文件一样，网络协议可以使用文本、二进制，或是两者的组合。但是网络和文件系统中也有很多的不同。

你只能同时读、或者写入一个文件，但是通过网络，你两者可以同时进行。一些协议通过循环的方式完成：发送一个请求，读取结果，重复。你可以通过单线程来实现这种协议。另外有的协议可能就允许你同时进行读写。典型的，你想使用一个专门的线程进行读。对于写，你可以使用一个专门的线程进行写，或是在多线程中使用同一个`Sink`使用。`Okio`的流对于多线程使用是不安全的。

`Sink`对输出的数据进行缓冲以最大化地减少`IO`操作。这是非常有效的，但是这意味着你必须手动调用`flush()`方法发送数据。典型地面向消息协议在每个消息之后都会进行`flush`。当缓冲数据超过一定阈值时，`Okio`将自动进行`flush`。这是为了节省内存，但你也不应该依赖这个机制。

`Okio`基于`java.io.Socket`进行连接。一个服务器或是客户端构造一个`Socket`，然后使用`Okio.source(Socket)`读取，`Okio.sink(Socket)`进行写。这两个`API`也可以和`SSLSocket`使用。你也应该尽量使用`SSL`。

在任何线程中调用`Socket.close()`取消`Socket`，这样会导致`Source`和`Sink`立即失败并抛出`IOException`。你也可以为所有的`Socket`操作配置超时。不需要为了配合超时去持有一个`Socket`的引用：`Source`和`Sink`直接暴露超时。即使流被装饰过，这个`API`也可以工作。

作为使用`Okio`进行网络操作的一个完整的例子，我们写了一个基本的`SOCKS`代理服务器。下面是一些重点：

```java
Socket fromSocket = ...
BufferedSource fromSource = Okio.buffer(Okio.source(fromSocket));
BufferedSink fromSink = Okio.buffer(Okio.sink(fromSocket));
```

为`Socket`构造`Source`和`Sink`和为文件构造它们是一样的。一旦构造得到`Source`和`Sink`，那么你将禁止在使用`InputStream`和`OutputStream`。

```java
Buffer buffer = new Buffer();
for (long byteCount; (byteCount = source.read(buffer, 8192L)) != -1; ) {
  sink.write(buffer, byteCount);
  sink.flush();
}
```

上面的代码循环复制`Source`中的数据到`Sink`中，在每次读取之后进行`flush`。如果我们不需要`flush`，我们可以使用一句代码替换这个循环，`BufferedSink.writeAll(Source)`。

参数`8192`指的是在每次返回之前读取的最大的字节数。我们可以在其中传入任何的值。但是我们喜欢`8kb`，因为这是`Okio`在一次系统调用中的最大值。大多数时候，应用代码不需要处理这个限制。

```java
int addressType = fromSource.readByte() & 0xff;
int port = fromSource.readShort() & 0xffff;
```

`Okio`使用有符号的类型如`byte`和`short`，但是部分协议协议使用的是无符号的值。使用位操作符`&`将一个有无符号的值转换成一个无符号的值是`Java`的惯用方法。下面是`byte`、`short`、和`int`的一个转换备忘单（注意将`byte`和`short`转换成了`int`，`int`转换成了`long`）：

| Type |	Signed Range |	Unsigned Range |	Signed to Unsigned |
| --- | --- | --- | --- |
| byte |	-128..127 |	0..255 |	int u = s & 0xff; |
| short |	-32,768..32,767 |	0..65,535 |	int u = s & 0xffff; |
|int |	-2,147,483,648..2,147,483,647	| 0..4,294,967,295 |	long u = s & 0xffffffffL; |

`Java`没有可替代无符号`long`类型的基础类型。

### 哈希

作为`Java`程序员，我们都受到过哈希的轰炸。早期，我们被介绍`hashCode()`方法，我们知道需要重写这个方法，否则可能会发生意想不到的事情。后来我们我们接触`LinkedHashMap`和与其相关的类。它们组织数据以快速取出的机制都建立在`hashCode()`方法上。

在其他地方我们有用到加密哈希函数。这些都被广泛使用。如`HTTPS`证书、`git`提交，`BitTorrent`完整性检查和区块链分块，都使用了加密哈希。哈希使用得当可以提升性能、隐私、安全和应用的简洁性。

每个加密哈希函数接受一个变长的字节输入流，产生一个固定长度字节字符串（称为`hash`）。哈希函数有以下几个重要特征：

* 确定性：同一个输入总是产生同样的输出。
* 统一性：每个输出字节字符串都容易进行对比。很难去找到不同的输入有同样输出的哈希。这被称为“意外”。
* 不可逆：知道输出并不能帮助你找到输入。注意，如果你知道一些可能的输入，你可以对他们进行`hash`，看他们的结果是否匹配。
* 知名：哈希在各种地方都有实现，并且严格。

好的哈希函数计算消耗很低（大约10微秒），并且逆向消耗很高（以千年为单位）。计算稳定和数学性质使得一个好的哈希函数很难被逆向。当选择一个哈希函数时，一定注意不是所有的都是相等的。`Okio`支持以下几种知名的哈希加密算法：

* `MD5`：一个128位（16字节）的哈希值。不安全并且已经过时了，因为逆向消耗并不是非常巨大。提供这种哈希算法是因为它非常流行，并且对于早期对安全不敏感的系统来说非常便捷。
* `SHA-1`：一个160位（20字节）的哈希值。最近被证明是可能发生“意外”的。所以考虑从`SHA-1`升级到`SHA-256`。
* `SHA-256`：256位（32字节）的哈希值。`SHA-256`被广泛的理解，并且逆向消耗巨大。大多数系统应该使用这个哈希函数。
* `SHA-512`：512位（64字节）的哈希值。非常难以逆向。

每一个哈希构造一个固定长度的`ByteString`。使用`hex()`方法获取方便的16进制字符串。或者保留为`ByteString`，因为`ByteString`是一个方便的数据类型。

`Okio`使用`ByteString`生成加密哈希值：

```java
ByteString byteString = readByteString(new File("README.md"));
System.out.println("   md5: " + byteString.md5().hex());
System.out.println("  sha1: " + byteString.sha1().hex());
System.out.println("sha256: " + byteString.sha256().hex());
System.out.println("sha512: " + byteString.sha512().hex());
```

从`Buffer`中读取：

```java
Buffer buffer = readBuffer(new File("README.md"));
System.out.println("   md5: " + buffer.md5().hex());
System.out.println("  sha1: " + buffer.sha1().hex());
System.out.println("sha256: " + buffer.sha256().hex());
System.out.println("sha512: " + buffer.sha512().hex());
```

从`Source`对应的输入流：

```java
try (HashingSink hashingSink = HashingSink.sha256(Okio.blackhole());
     BufferedSource source = Okio.buffer(Okio.source(file))) {
  source.readAll(hashingSink);
  System.out.println("sha256: " + hashingSink.hash().hex());
}
```

从`Sink`对应的输出流：

```java
try (HashingSink hashingSink = HashingSink.sha256(Okio.blackhole());
     BufferedSink sink = Okio.buffer(hashingSink);
     Source source = Okio.source(file)) {
  sink.writeAll(source);
  sink.close(); // Emit anything buffered.
  System.out.println("sha256: " + hashingSink.hash().hex());
}
```

`Okio`也支持`HMAC`（），它组合密钥和一个哈希值。应用程序通常使用`HMAC`保证数据正确性和验证。

```java
ByteString secret = ByteString.decodeHex("7065616e7574627574746572");
System.out.println("hmacSha256: " + byteString.hmacSha256(secret).hex());
```

通过哈希，你可以使用`ByteString`，`Buffer`，`HashingSource`和`HashingSink`生成`HMAC`。注意`Okio`没有实现`MD5`的`HMAC`。`Okio`利用`Java`的`java.security.MessageDigest`生成哈希，使用`javax.crypto.Mac`生成`HMAC`。