## Source和Sink

java.io设计的一个优雅部分是如何对流进行分层来处理加密和压缩等转换。Okio有自己的stream类型: Source和Sink，分别类似于java的Inputstream和Outputstream，但是有一些关键区别：

- **超时(Timeouts)**。流提供了对底层I/O超时机制的访问。与java.io的socket字流不同，read()和write()方法都给予超时机制。
- **容易实现**。source只声明了三个方法：read()、close()和timeout()。没有像available()或单字节读取这样会导致正确性和性能意外的危险操作。
- **使用方便**。虽然source和sink的实现只有三种方法可写，但是调用方可以实现**BufferedSource**和**BufferedSink**接口, 这两个接口提供了丰富API能够满足你所需的一切。
- **字节流和字符流之间没有人为的区别**。都是数据。你可以以字节、UTF-8字符串、big-endian的32位整数、little-endian的短整数等任何你想要的形式进行读写；不再有InputStreamReader！
- **易于测试**。Buffer类同时实现了BufferedSource和BufferedSink接口，因此测试代码简单明了。

Source和Sink分别与InputStream 和 OutputStream交互操作。你可以将任何Source看做InputStream ，也可以将任何InputStream 当做Source。对于Sink和Outputstream也是如此。


### Sink

我们先看Sink的代码：

```
interface Sink : Closeable, Flushable {
  @Throws(IOException::class)
  fun write(source: Buffer, byteCount: Long)

  @Throws(IOException::class)
  override fun flush()

  fun timeout(): Timeout

  @Throws(IOException::class)
  override fun close()
}
```
Sink是一个接口，它用于将字节流传输到任何位置，比如网络中，存储器中或者内存中。Sink定义了四个函数：
1. `wirte(source: Buffer,byteCount: Long)`，用于将souce中byteCount个数据写出，当然Okio中的实现不会直接将数据写出，而是写到缓存区Buffer中，然后再写出。
2. `flush()`用于将缓冲区的数据推送给目标
3. `timeout()`返回设置的超时信息
4. `close()`将缓冲区的数据推送给目标并释放资源


**BufferedSink**

![image](https://note.youdao.com/yws/api/personal/file/EAA3BD3112CC42CEA736A93AD8E50646?method=download&shareKey=b0c23038295552fd82918bc6b576174b)

**BufferedSink**继承了**Sink**，并且定义了更多更丰富的写操作的函数。同时，BufferedSink还维护了一个Buffer，用于缓存数据。

#### Sink的使用

```
    val bufferedSink = File("hello.txt").sink(false).buffer()
    bufferedSink.write("hello world!".toByteArray())
            .writeUtf8("你好世界！")
            .close()
```

1. 首先我们创建了一个名为hello.txt的File对象，并通过Kotlin的扩展方法sink(false)，得到Sink对象，传入参数false表示写入文件时，不会在原本的末尾之后继续写，而是覆盖掉原本的文件。
2. 然后通过Sink的扩展方法buffer(),我们可以获得BufferedSink对象。
3. BufferedSink提供了丰富的接口，用于写数据，例如上面的代码中，使用了write(ByteArray)、writeUtf8(String)。
4. 最后，我们需要调用close()将BufferedSink中的内容推送到目标，并释放资源。

#### Sink是如何实现的？

```
@JvmOverloads
@Throws(FileNotFoundException::class)
fun File.sink(append: Boolean = false): Sink = FileOutputStream(this, append).sink()

/** Returns a sink that writes to `out`. */
fun OutputStream.sink(): Sink = OutputStreamSink(this, Timeout())

private class OutputStreamSink(
  private val out: OutputStream,
  private val timeout: Timeout
) : Sink {

  override fun write(source: Buffer, byteCount: Long) {
    checkOffsetAndCount(source.size, 0, byteCount)
    var remaining = byteCount
    while (remaining > 0) {
      timeout.throwIfReached()
      val head = source.head!!
      val toCopy = minOf(remaining, head.limit - head.pos).toInt()
      //使用outputstream来进行写操作
      out.write(head.data, head.pos, toCopy)

      head.pos += toCopy
      remaining -= toCopy
      source.size -= toCopy

      if (head.pos == head.limit) {
        source.head = head.pop()
        SegmentPool.recycle(head)
      }
    }
  }

  override fun flush() = out.flush()

  override fun close() = out.close()

  override fun timeout() = timeout

  override fun toString() = "sink($out)"
}

```
可以发现，File.sink()这个扩展方法实际上调用了FileOutputStream.sink()这个扩展方法，最终返回了一个**OutputStreamSink**。

OutputStreamSink是Okio中的私有类，它持有一个OutputStream对象和一个Timeout对象。
在OutputStreamSink实现的write方法中，需要传入一个Buffer对象以及操作的字节数byteCount。在方法中，我们可以看到，OutputStreamSink的写操作是利用了OutputStream的write方法，并且它的flush()、close()方法，都是对OutputStream的一个封装。那么，我们之前使用Sink的代码，可以改为：

```
    val sink = File("hello.txt").sink(false)
    val buffer = Buffer()
    //在Buffer中写入utf8编码的字符串
    buffer.writeUtf8("你好世界")
    sink.write(buffer,buffer.size)
    sink.close()
```
没有问题的，在hello.txt文件中，成功写入了“你好世界”字样。这样子，与通过BufferedSink的write()方法有什么区别呢？

我们看到Sink.buffer()方法，该方法得到一个RealBufferedSink对象。

```
fun Sink.buffer(): BufferedSink = RealBufferedSink(this)
```

在构造RealBufferedSink时，传入Sink自身，可以获得RealbufferedSink对象，RealbufferedSink实现了BufferedSink接口，即这里是**装饰者模式**，由于Sink只有简单的几个方法，并不能满足我们的使用，通过**BufferedSink装饰Sink**，可以在不修改基础定义的情况下，扩展方法，即对扩展开放，对修改关闭

```
internal class RealBufferedSink(
  @JvmField val sink: Sink
) : BufferedSink {
  @JvmField val bufferField = Buffer()
  @JvmField var closed: Boolean = false

  @Suppress("OVERRIDE_BY_INLINE") // Prevent internal code from calling the getter.
  override val buffer: Buffer
    inline get() = bufferField

  override fun buffer() = bufferField

  override fun writeUtf8(string: String): BufferedSink {
    check(!closed) { "closed" }
    buffer.writeUtf8(string)
    return emitCompleteSegments()
  }

  override fun emitCompleteSegments(): BufferedSink {
    check(!closed) { "closed" }
    val byteCount = buffer.completeSegmentByteCount()
    if (byteCount > 0L) sink.write(buffer, byteCount)
    return this
  }
//省略一大堆write方法以及其他方法

```
由于篇幅有限，文章就只展示RealBufferedSink的一部分代码，我们来看常用的`writeUtf8(string: String)`方法，该方法只有很简单的三行：
1. 第一行检查Sink是否关闭。
2. 第二行使用了`buffer.writeUtf8(string)`，该方法将UTF8编码的字符串存入Buffer缓存中 [Okio分析一ByteString与Buffer](https://note.youdao.com/ynoteshare1/index.html?id=0ea55bbf075ee1a145ce401a6f6f5fc0&type=note)。
3. 最后一行，调用了 `emitCompleteSegments()`，在该方法中，通过` buffer.completeSegmentByteCount()`方法获取要Buffer中要写的字节数，最终， `sink.write(buffer, byteCount)`写出数据，即我们上文分析的OutputStreamSink通过OutputStream来写数据。

### Source

先看Source的代码：


```
interface Source : Closeable {

  @Throws(IOException::class)
  fun read(sink: Buffer, byteCount: Long): Long

  fun timeout(): Timeout

  @Throws(IOException::class)
  override fun close()
}

```
Source也是一个接口，它用于接收从网络上，存储器或者内存中的数据，它定义了三个函数：
1. read(sink: Buffer, byteCount: Long)，将读取的数据存于缓存中，返回-1表示读取完毕，返回>0的值表示读取了多少个字节。
2. timeout(), 返回souce的超时信息
3. close()，关闭source，并释放资源

#### BufferedSource

![image](https://note.youdao.com/yws/api/personal/file/8B5680860F2B4438AB3CE22381C9D5E1?method=download&shareKey=3701e9f35e6d6cb8185913dc7e6110ba)

BufferedSource内部也持有一个Buffer对象，用于保存读取到的数据，并且它还提供了丰富的接口，用于读取数据


#### Source的使用

```
    val bufferedSource = File("hello.txt").source().buffer()
    while (true){
        val readUtf8Line = bufferedSource.readUtf8Line() ?: break
        println(readUtf8Line)
    }
    bufferedSource.close()
```
我们可以使用Source读取文件数据，上面的代码，就是我们读取使用Sink写入到hello.txt文件中的内容。我们可以看到代码执行后会按行输出文件中的内容

1. 首先我们创建了一个名为hello.txt的File对象，并通过Kotlin的扩展方法source()，得到一个Source对象
2. 然后通过Source的扩展方法buffer(),我们可以获得BufferedSource对象。
3. BufferedSource提供了丰富的接口，用于读数据，上面的代码中，使用到了readUtf8Line(),可以按行读取Utf8字符串。
4. 最后，我们需要调用close()使BufferedSource释放资源。

#### Source是如何实现的？

```
    @Throws(FileNotFoundException::class)
    fun File.source(): Source = inputStream().source()
    
    @kotlin.internal.InlineOnly
    public inline fun File.inputStream(): FileInputStream {
        return FileInputStream(this)
    }

    /** Returns a source that reads from `in`. */
    fun InputStream.source(): Source = InputStreamSource(this, Timeout())
    
    private class InputStreamSource(
      private val input: InputStream,
      private val timeout: Timeout
    ) : Source {
    
      override fun read(sink: Buffer, byteCount: Long): Long {
        if (byteCount == 0L) return 0
        require(byteCount >= 0) { "byteCount < 0: $byteCount" }
        try {
          timeout.throwIfReached()
          //从Buffer中，得到一个还有可写入空间的Segment
          val tail = sink.writableSegment(1)
           //比较byteCount与当前这个Segment的最大空间
          val maxToCopy = minOf(byteCount, Segment.SIZE - tail.limit).toInt()
          //使用InputStream.read()方法，将数据存入Segment持有的Byte数组中
          val bytesRead = input.read(tail.data, tail.limit, maxToCopy)
          //如果input中的数据已经读取完毕，返回-1
          if (bytesRead == -1) return -1
          //input中的数据没有读取完毕，增加Segment中的limit与和sink中的size属性
          tail.limit += bytesRead
          sink.size += bytesRead
          return bytesRead.toLong()
        } catch (e: AssertionError) {
          if (e.isAndroidGetsocknameError) throw IOException(e)
          throw e
        }
      }
  
    }
```

通过`File.source()`，利用`Kotlin.io`中对于File的扩展方法，得到`FileInputStream`对象，然后通过`InputStream.source()`得到一个`InputStreamSource`对象。

InputStreamSource实现了Source，持有一个InputStream对象和一个Timeout对象，与OutputStreamSink相对的，InputStreamSource实现的是read()方法，**内部实际通过InputStream读取数据**。而在该实现中，我们可以发现，如果文件很大，即`byteCount`超过了 `Segment.SIZE - tail.limit`，那么该方法并不能一次性读取完所有的数据存入Buffer，**那么如果我们直接使用InputStreamSource，我们需要判断这一点！**

当然，我们一般使用的是BufferedSource而不是直接使用InputStreamSource，通过Source.buffer()这个扩展方法，得到一个BufferedSource对象，即RealBufferedSource。

```
fun Source.buffer(): BufferedSource = RealBufferedSource(this)
```
与Sink和Buffered一样，Source与BufferedSource也是装饰者模式的实现，BufferedSource装饰Source来扩展Source的方法。

```
internal class RealBufferedSource(
  @JvmField val source: Source
) : BufferedSource {
  @JvmField val bufferField = Buffer()
  @JvmField var closed: Boolean = false

  @Suppress("OVERRIDE_BY_INLINE") // Prevent internal code from calling the getter.
  override val buffer: Buffer
    inline get() = bufferField

  override fun buffer() = bufferField


  override fun readUtf8(): String {
    buffer.writeAll(source)
    return buffer.readUtf8()
  }
  
  //省略了一大堆代码

  override fun isOpen() = !closed

  override fun close() {
    if (closed) return
    closed = true
    source.close()
    buffer.clear()
  }

  override fun timeout() = source.timeout()

  override fun toString() = "buffer($source)"
}

```

#### Source中的read操作

同样省略了一大堆代码，我们来分析RealBufferedSource中的readUtf8()，首先我们可以看到，RealBufferedSource持有一个Source，这个Source就是InputStreamSource。
在`readUtf8()`中，实际上调用了`buffer.writeAll(source)`，我们来看这个方法：

```
  override fun writeAll(source: Source): Long {
    var totalBytesRead = 0L
    while (true) {
      val readCount = source.read(this, Segment.SIZE.toLong())
      if (readCount == -1L) break
      totalBytesRead += readCount
    }
    return totalBytesRead
  }
```

可以看到，`writeAll(source: Source)`循环使用了`InputStreamSource.read()`来将InputStream中的数据读取到Buffer中，直到InputStream中得数据读取完毕。然后，通过`buffer.readUtf8()`，将Buffer中保存的数据，转化成Utf8编码的String返回！

#### 小总结

如图，在Sink与Source的使用中，实际上最终是通过Java的OutputStream与InputStream实现的，而他们的实现采用了装饰者模式。

![image](https://note.youdao.com/yws/api/personal/file/BF5AE50E2CC54D5FA6C14906C9FCB72F?method=download&shareKey=33beb8ded3e806fbd45e32372c9427ce)

### Gzip压缩传输

Okio自带GZIP压缩以及解压功能，具体实现由GzipSource与GzipSink完成：

- GzipSink 实现Sink接口，是带有压缩功能的Sink，会将要写出的数据压缩之后再写出。
- GzipSource 实现了Source接口，是带有解压功能的Source，会将压缩后的数据解压后输入。

#### gzip的使用


```
    val sink = File("gzip.txt").sink(false).gzip().buffer().use {
        it.writeUtf8("是他，就是他，我们的朋友，小哪吒！")
    }
```
上面代码中，通过GzipSink，我们可以将压缩后的数据写入名为"gzip.txt"的文件中，如果你运行了这段代码，可以发现，文件中会是一堆乱码，当然，这是正确的，因为数据经过了压缩！

当然，我们可以使用GzipSource将压缩后的数据解压：


```
    val source = File("gzip.txt").source().gzip().buffer().use {
        println(it.readUtf8())
    }

输出：是他，就是他，我们的朋友，小哪吒！
```
可以知道，GzipSink帮助我们压缩数据，GzipSource帮助我们解压数据，而它们的使用，也十分方便，只需要在原来的基础上增加一步：

![image](https://note.youdao.com/yws/api/personal/file/F04DF14928B544E9A2AD10660C86D6A1?method=download&shareKey=0b76468191fd4308cb330b9704c31874)

#### GzipSink

先来看GzipSink的实现：


```
class GzipSink(sink: Sink) : Sink {
  /** Sink into which the GZIP format is written. */
  private val sink = RealBufferedSink(sink)
  val deflater = Deflater(DEFAULT_COMPRESSION, true /* No wrap */)
  private val deflaterSink = DeflaterSink(this.sink, deflater)
  private var closed = false
  /** Checksum calculated for the compressed body. */
  private val crc = CRC32()

  init {
    // 直接写Gzip格式的头
    this.sink.buffer.apply {
      writeShort(0x1f8b) // Two-byte Gzip ID.
      writeByte(0x08) // 8 == Deflate compression method.
      writeByte(0x00) // No flags.
      writeInt(0x00) // No modification time.
      writeByte(0x00) // No extra flags.
      writeByte(0x00) // No OS.
    }
  }
  @Throws(IOException::class)
  override fun write(source: Buffer, byteCount: Long){...}

  @Throws(IOException::class)
  override fun flush() = deflaterSink.flush()
  override fun timeout(): Timeout = sink.timeout()
  //关闭Sink
  @Throws(IOException::class)
  override fun close() {... }
  //写入数据的尾，用于校验数据
  private fun writeFooter() {...}
}
inline fun Sink.gzip() = GzipSink(this)
```

GzipSink实现Sink接口，并且在构造器中，需要传入一个Sink，我们通过上文知道，我们传入的Sink就是`InputStreamSink`，并且GzipSink直接通过该sink生成一个`RealBufferedSink`。除了sink，还有GzipSink还有其它几个成员：
1. `deflater`，Java的压缩类，用于数据压缩
2. `deflaterSink`，在deflaterSink进行数据的压缩
3. `crc`，用于数据的校验，CRC32可以根据字节数组，生成一个长整型，不同字节数组，生成的数值不同，可以由此判断数据是否完整。

当我们使用`RealBufferedSink.write`等方法时，我们知道内部实际调用的是RealBufferedSink持有的Sink，而此时该Sink为GzipSink，所以我们的重点在于GzipSink中的`write()`方法：


```
  @Throws(IOException::class)
  override fun write(source: Buffer, byteCount: Long) {
    require(byteCount >= 0L) { "byteCount < 0: $byteCount" }
    if (byteCount == 0L) return
    //更新crc生成的值，每次写数据都要更新值
    updateCrc(source, byteCount)
    deflaterSink.write(source, byteCount)
  }

  /** Updates the CRC with the given bytes. */
  private fun updateCrc(buffer: Buffer, byteCount: Long) {
    var head = buffer.head!!
    var remaining = byteCount
    while (remaining > 0) {
      val segmentLength = minOf(remaining, head.limit - head.pos).toInt()
      crc.update(head.data, head.pos, segmentLength)
      remaining -= segmentLength
      head = head.next!!
    }
  }

```
GzipSink中`write(source: Buffer, byteCount: Long)`，比较简单，因为它将压缩的工作都交给了DeflaterSink来做：

```
  @Throws(IOException::class)
  override fun write(source: Buffer, byteCount: Long) {
    checkOffsetAndCount(source.size, 0, byteCount)
    var remaining = byteCount
    while (remaining > 0) {
      //当还有字节没有压缩时，继续压缩数据
      val head = source.head!!
      //获取头Segment可压缩的字节数，即可读区域大小
      val toDeflate = minOf(remaining, head.limit - head.pos).toInt()
      //设置压缩器中的数据
      deflater.setInput(head.data, head.pos, toDeflate)
      //压缩数据
      deflate(false)
      
      source.size -= toDeflate
      head.pos += toDeflate
      if (head.pos == head.limit) {
        source.head = head.pop()
        SegmentPool.recycle(head)
      }
      remaining -= toDeflate
    }
  }
  
   private fun deflate(syncFlush: Boolean) {
    val buffer = sink.buffer
    while (true) {
      //获取末尾的Segment，没有则从SegmentPool中取，并加入末尾
      val s = buffer.writableSegment(1)
      //压缩数据并写入Segment中，返回压缩后的字节长度
      val deflated = if (syncFlush) {
        deflater.deflate(s.data, s.limit, Segment.SIZE - s.limit, Deflater.SYNC_FLUSH)
      } else {
        deflater.deflate(s.data, s.limit, Segment.SIZE - s.limit)
      }

      if (deflated > 0) {
        s.limit += deflated
        buffer.size += deflated
        //直接输出压缩后的数据
        sink.emitCompleteSegments()
      } else if (deflater.needsInput()) {
        if (s.pos == s.limit) {
          // We allocated a tail segment, but didn't end up needing it. Recycle!
          buffer.head = s.pop()
          SegmentPool.recycle(s)
        }
        //当没有压缩器中没有数据时，跳出循环
        return
      }
    }
  }
  
```
可以看出DeflaterSink通过Deflater来进行压缩的工作，并且一段数据压缩好后，通过`sink.emitCompleteSegments()`直接输出数据。

在GzipSink的`close()`中，将crc的值与原数据长度写入数据的末尾，用于校验：

```
  @Throws(IOException::class)
  override fun close() {
    if (closed) return
    var thrown: Throwable? = null
    try {
      deflaterSink.finishDeflate()
      writeFooter()
    } catch (e: Throwable) {
      thrown = e
    }
    //..deflater与sink的关闭释放
  }
  private fun writeFooter() {
    sink.writeIntLe(crc.value.toInt()) 
    sink.writeIntLe(deflater.bytesRead.toInt())
  }
```
注意：这里的footer数据，在`sink.close()`时，会输出，并不会不输出！

#### GzipSource


```
class GzipSource(source: Source) : Source {

  private var section = SECTION_HEADER
  private val source = RealBufferedSource(source)
  private val inflater = Inflater(true)//解压数据的工具
  //在InflaterSource解压数据
  private val inflaterSource = InflaterSource(this.source, inflater)

  private val crc = CRC32()

  @Throws(IOException::class)
  override fun read(sink: Buffer, byteCount: Long): Long {
    require(byteCount >= 0L) { "byteCount < 0: $byteCount" }
    if (byteCount == 0L) return 0L

    if (section == SECTION_HEADER) {
      //校验头信息并读取数据
      consumeHeader()
      section = SECTION_BODY
    }
    if (section == SECTION_BODY) {
      val offset = sink.size
      //通过inflaterSource解压数据
      val result = inflaterSource.read(sink, byteCount)
      if (result != -1L) {
        updateCrc(sink, offset, result)
        return result
      }
      section = SECTION_TRAILER
    }

    if (section == SECTION_TRAILER) {
      consumeTrailer()
      section = SECTION_DONE

      if (!source.exhausted()) {
        throw IOException("gzip finished without exhausting source")
      }
    }

    return -1
  }

```
GzipSource将传入的InputStreamSource生成一个RealBufferedSource，并且它是通过InflaterSource来进行解压数据的。

上文中我们知道，当我们使用`RealBufferedSource.readUtf8()`时，会在`Buffer.writeAll()`中调用持有的Source（这里即**GzipSource**）的`read(sink: Buffer, byteCount: Long)`，我们看该方法。

首先当第一次进来时，`section == SECTION_HEADER`，将会进入`consumeHeader()`去检验我们压缩数据时填入的头信息，即GzipSink初始化的时设置的值，并且下载数据：


```
 @Throws(IOException::class)
  private fun consumeHeader() {
    source.require(10)
    //省略后面的校验代码
  }
  
  //RealBufferedSource中
  override fun require(byteCount: Long) {
    if (!request(byteCount)) throw EOFException()
  }

  override fun request(byteCount: Long): Boolean {
    require(byteCount >= 0) { "byteCount < 0: $byteCount" }
    check(!closed) { "closed" }
    while (buffer.size < byteCount) {
      if (source.read(buffer, Segment.SIZE.toLong()) == -1L) return false
    }
    return true
  }


```
`consumeHeader()`调用`source.require(10)`，该方法会通过RealBufferedSource中持有的Source去读取数据，由于Gzip中的RealBufferedSource是由我们一开始传入InputStreamSource生成的，即这里依然是通过InputStream去读数据。并且参数为10表示读取数据长度必须大于10，即保证压缩的头信息有10位。

当校验完成头信息，并且下载数据完成后，开始通过inflaterSource.read(sink, byteCount)解压并读取数据：


```
 @Throws(IOException::class)
  override fun read(sink: Buffer, byteCount: Long): Long {
    require(byteCount >= 0) { "byteCount < 0: $byteCount" }
    check(!closed) { "closed" }
    if (byteCount == 0L) return 0

    //开始循环读取数据
    while (true) {
      //填充需要解压的数据
      val sourceExhausted = refill()

      try {
        val tail = sink.writableSegment(1)
        val toRead = minOf(byteCount, Segment.SIZE - tail.limit).toInt()
        //解压数据，并且写入Buffer中
        val bytesInflated = inflater.inflate(tail.data, tail.limit, toRead)
        if (bytesInflated > 0) {
          tail.limit += bytesInflated
          sink.size += bytesInflated
          return bytesInflated.toLong()
        }
        if (inflater.finished() || inflater.needsDictionary()) {
          releaseInflatedBytes()
          if (tail.pos == tail.limit) {
            // We allocated a tail segment, but didn't end up needing it. Recycle!
            sink.head = tail.pop()
            SegmentPool.recycle(tail)
          }
          return -1L
        }
        if (sourceExhausted) throw EOFException("source exhausted prematurely")
      } catch (e: DataFormatException) {
        throw IOException(e)
      }
    }
  }
  
    @Throws(IOException::class)
  fun refill(): Boolean {
    if (!inflater.needsInput()) return false

    releaseInflatedBytes()
    check(inflater.remaining == 0) { "?" } // TODO: possible?

    // If there are compressed bytes in the source, assign them to the inflater.
    if (source.exhausted()) return true

    // Assign buffer bytes to the inflater.
    val head = source.buffer.head!!
    bufferBytesHeldByInflater = head.limit - head.pos
    inflater.setInput(head.data, head.pos, bufferBytesHeldByInflater)
    return false
  }

  /** When the inflater has processed compressed data, remove it from the buffer.  */
  private fun releaseInflatedBytes() {
    if (bufferBytesHeldByInflater == 0) return
    val toRelease = bufferBytesHeldByInflater - inflater.remaining
    bufferBytesHeldByInflater -= toRelease
    source.skip(toRelease.toLong())
  }

```

当解压完数据，校验CRC验证数据的完整性：

```
  private fun consumeTrailer() {
    checkEqual("CRC", source.readIntLe(), crc.value.toInt())
    checkEqual("ISIZE", source.readIntLe(), inflater.bytesWritten.toInt())
  }
```

### 总结

Sink与Source底层仍然是通过InputStrem与OutputStream实现了，但是Okio使用了自己的缓存类Buffer而不是Java.nio的，封装了自身的缓存逻辑！

Okio提供了Gzip的实现，一般在网络传输中，使用压缩来节省流量。