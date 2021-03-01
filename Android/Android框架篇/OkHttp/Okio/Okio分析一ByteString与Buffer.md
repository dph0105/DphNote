Okio是围绕**ByteString**和**Buffer**这两种类型构建的，本篇文章将介绍这两个类

### ByteString

**ByteString**是不可变的字节序列。**ByteString**与**String**类似，区别在于，**String**是字符**char**的串，而**ByteString**是字节**Byte**的串。

ByteString提供了丰富的方法，来给使用者进行编码或者解码操作，它使得将二进制数据作为一个变量值变得容易操作。


![image](https://note.youdao.com/yws/api/personal/file/29202C940A7A4AD7AFEBBB9220992340?method=download&shareKey=72eb1caa6a0750ba2b14da871d8cdb67)

可以看到，ByteString提供了丰富的方法去操作字节以及编码。我们选择其中一些方法进行分析


```
actual open class ByteString
// Trusted internal constructor doesn't clone data.
internal actual constructor(
  internal actual val data: ByteArray
) : Serializable, Comparable<ByteString> {
  @Transient internal actual var hashCode: Int = 0 // Lazily computed; 0 if unknown.
  @Transient internal actual var utf8: String? = null // Lazily computed.
  actual val size
    @JvmName("size") get() = getSize()

  actual open fun utf8(): String = commonUtf8()

  //直接通过String的方法进行编码转换
  open fun string(charset: Charset) = String(data, charset)

}

internal inline fun ByteString.commonUtf8(): String {
  var result = utf8
  if (result == null) {
    // We don't care if we double-allocate in racy code.
    result = internalArray().toUtf8String()
    utf8 = result
  }
  return result
}

```

为什么说ByteString是不可变的呢？我们可以看到ByteString持有了四个变量：ByteArray(data)、hashcode值，utf8编码的String以及字节数组大小size。**其中data和size被声明为只读变量val(相当于java中的final)**，并且data被internal修饰，size只提供了get()函数，都无法在外部修改他们的值。所以ByteString被创建后，它所持有的ByteArray与size就不可被修改。

ByteString的utf8变量用于缓存Byte数组被编码为UTF8格式的String，这样，当我们需要在Byte数组与UTF8格式的字符串进行转换时，就不需要进行任何工作了。我们可以看到代码中的`ByteString.commonUtf8()`函数，只有当utf8为null时，才会进行字节的编码转换工作，否则直接返回utf8字符串，以空间换时间。

##### ByteString的使用

当我们需要创建一个ByteArray时，可以通过它的伴生对象来创建：

```
    ByteString.EMPTY
    ByteString.of(49,50,51)
    val str = "123"
    val byteStr_utf8 = str.encodeUtf8()
    val byteStr_ascii = str.encode(Charsets.US_ASCII)
```

`ByteString.EMPTY`创建一个空的ByteString，`ByteString.of(49,50,51)`是创建一个字节为`[49,50,51]`这样的ByteString。ByteString的伴生对象还声明一些扩展方法，如`String.encodeUtf8()`、`String.encode(charset：Charset)`，这些扩展方法很方便，但是我们在使用时，需要`import okio.ByteString.Companion.encodeUtf8`

##### ByteString是否真的不可变？

ByteString是否真的不可变呢？我们知道Java中String也被称为是不可变的，但是我们通过反射的方法，依然可以改变，ByteString也是如此：

```
    val str = "123"
    val byteStr = str.encodeUtf8()

    val field = ByteString::class.java.getDeclaredField("data")
    field.isAccessible = true
    val data = field.get(byteStr) as ByteArray
    data[0] = 97
    println(byteStr.string(Charsets.UTF_8))
    
    输出结果为a23
```
上面的代码中，我们使用`ByteString.string(charset: Charset)`而不是直接使用`ByteString.utf8()`，是因为byteStr是直接通过`encodeUtf8()`获得的，保留了原本的uft8格式的String的缓存，直接通过`ByteString.utf8()`打印出来的还是“123”噢

不仅通过反射，由于ByteString提供了IO流访问持有的Byte数组的方法，在IO流中，我们也能改变它的内容：

```
    val str = "123"
    val byteStr = str.encodeUtf8()
    byteStr.write(object : OutputStream() {
        override fun write(b: Int) {
        }
        override fun write(b: ByteArray) {
            b[0] = 98
        }
    })
    println(byteStr.string(Charsets.UTF_8))
    
    输出结果为b23
```

### Buffer

在了解**Okio中的Buffer**之前，我们简单了解一下**java.nio中的Buffer**。


#### java.nio中的Buffer

**java.nio中的Buffer**被定义为面向缓冲区编程，它的Buffer,便承担了缓冲区的作用。

```
public abstract class Buffer {

    // Invariants: mark <= position <= limit <= capacity
    private int mark = -1;
    private int position = 0;
    private int limit;
    //省略一些代码
}
```
先看一下Buffer中的三个重要的变量：

- capacity：容量，必须初始化的值（因为底层是数组）
- limit：上界，缓冲区的临界区，即最多可读到哪个位置
- position：下标，当前读取到的位置(例如当前读出第3个元素，则读完后，position为4)

当然，作为缓冲区，Buffer是用数组来储存数据的，如一般使用的ByteBuffer：

```
public abstract class ByteBuffer extends Buffer implements Comparable<ByteBuffer>{
    final byte[] hb;
    //省略代码。。。。。
}
```
![image](https://note.youdao.com/yws/api/personal/file/5CE82198CE5E47DC9FF200B2456EE32B?method=download&shareKey=4a5b2ebf25c12c6e390c96f5ba8dc344)

如图，我们创建的是容量capacity为128的一个Buffer，此时，代表着Buffer中，缓存的数据大小为7，那么下一个要写入的位置，即limit为7。红色部分代表已经被读取的数据，绿色部分代表还没有被读取的数据，即position为3，而7-128的位置可以继续缓存数据。


#### Okio中的Buffer

介绍完java.nio中的Buffer，我们继续回到我们的**Okio中的Buffer**，先来看官方对自家的Buffer的介绍：

**Buffer**是一个可变的字节序列。像Arraylist一样，你不需要预先设置缓冲区的大小。你可以将缓冲区读写为一个队列：将数据写到队尾，然后从队头读取。

**Buffer是作为Segment的链表实现的**。当您将数据从一个缓冲区移动到另一个缓冲区时，它会重新分配片段的持有关系，而不是跨片段复制数据。这对多线程特别有用：与网络交互的子线程可以与工作线程交换数据，而无需任何复制或多余的操作。

Buffer有三个特点：

**它将数据从一个缓冲区移到另一个缓冲区的速度很快**。此类不会将字节从内存中的一个位置复制到另一位置，而只是更改基础字节数组的所有权。

**此缓冲区随数据增长**。就像ArrayList一样，每个缓冲区从小处开始。它仅消耗所需的内存。

**此缓冲区池合并它的字节数组**。在Java中分配字节数组时，运行时必须在将请求的数组返回给您之前将其清空。即使您无论如何都要在该空间上书写。此类通过合并字节数组避免了清空和GC。

根据介绍，我们发现，似乎我们Okio中Buffer的实现有点不太一样噢，我们来看代码吧！

```
class Buffer : BufferedSource, BufferedSink, Cloneable, ByteChannel {

  @JvmField internal var head: Segment? = null

  @get:JvmName("size")
  var size: Long = 0L
    internal set
  //...省略一些代码
 }

```

可以看到，Buffer中只有两个变量，一个是**head**，代表Segment循环双向链表的表头。另一个是**size**，表示Buffer内的未读数据大小。Buffer将缓存分成一个个很小的片段，每个片段就是Segment，我们写数据或者读数据都是操作的Segment中维护的一个个数组。

我们先看Segment的实现：

```
internal class Segment {
  @JvmField val data: ByteArray

  /* 此段中要读取的应用程序数据字节的下一个字节。 */
  @JvmField var pos: Int = 0

  /* 准备写入的可用数据的第一个字节。 */
  @JvmField var limit: Int = 0

  /* 如果其他Segment或ByteString也持有它的字节数组（即是分享的），则为True。 */
  @JvmField var shared: Boolean = false

  /** 如果这个Segment单独拥有它的字节数组，并且可以在limit后添加内容，则为true  */
  @JvmField var owner: Boolean = false

  /** Segmetn链表中后一个节点  */
  @JvmField var next: Segment? = null

  /** Segment链表中前一个节点  */
  @JvmField var prev: Segment? = null

  constructor() {
    this.data = ByteArray(SIZE)
    this.owner = true
    this.shared = false
  }

  constructor(data: ByteArray, pos: Int, limit: Int, shared: Boolean, owner: Boolean) {
    this.data = data
    this.pos = pos
    this.limit = limit
    this.shared = shared
    this.owner = owner
  }

  companion object {
    /** The size of all segments in bytes.  */
    const val SIZE = 8192

    /** Segments will be shared when doing so avoids `arraycopy()` of this many bytes.  */
    const val SHARE_MINIMUM = 1024
  }
}

```
可以看到，Segment内部是维护一个Byte数组，Segment有两个构造方法，默认的构造创造了一个大小为8k（8192）的Byte数组，用作缓存数据。我们来看Segment的几个变量。`prev`和`next`分别表示链表中的前后节点，`shared`和`owner`则代表着Segment缓存的字节数组的所有权。而`pos`和`limit`，它们的作用与java.nio中Buffer的`position`和`limit`作用一样，如图：

![image](https://note.youdao.com/yws/api/personal/file/6E2E720E64C248EAB9FC28DAA6B5D1CD?method=download&shareKey=18e3755187286ff6db4e9e08aa489a9d)

上图代表一个默认创建的Segment持有一个大小8192的字节数组。有颜色的区域表示已写入数据的区域，红色表示已读数据，绿色表示未读区域。pos表示可以读取数据的下一个字节的位置，即3。白色则表示未写入数据的区域，limit表示下一个可写入数据的位置，即为7。

下面我们来介绍一下Segment的几个方法：

**pop()与push()**
```
/**
  fun pop(): Segment? {
    val result = if (next !== this) next else null
    prev!!.next = next
    next!!.prev = prev
    next = null
    prev = null
    return result
  }

  fun push(segment: Segment): Segment {
    segment.prev = this
    segment.next = next
    next!!.prev = segment
    next = segment
    return segment
  }

```
`pop()`与`push()`是链表的基本方法，`pop()`将当前Segment弹出，并返回下一个Segment；`push()`将传入的Segment加到当前Segment后，并返回加入的Segment。


**sharedCopy()与unsharedCopy()**

```
  fun sharedCopy(): Segment {
    shared = true
    return Segment(data, pos, limit, true, false)
  }

  fun unsharedCopy() = Segment(data.copyOf(), pos, limit, false, true)
```
`sharedCopy()`与`unsharedCopy()`都是复制一个相同数据的Segment，区别在于`sharedCopy()`复制的Segment，与它自身，持有同一个Byte数组，即`shared = true`，只能读不能写。而`unsharedCopy()`并不共同持有一个Byte数组，可读可写。

**split(byteCount: Int)**
```
  fun split(byteCount: Int): Segment {
    require(byteCount > 0 && byteCount <= limit - pos) { "byteCount out of range" }
    val prefix: Segment
    if (byteCount >= SHARE_MINIMUM) {
      prefix = sharedCopy()
    } else {
      prefix = SegmentPool.take()
      data.copyInto(prefix.data, startIndex = pos, endIndex = pos + byteCount)
    }

    prefix.limit = prefix.pos + byteCount
    pos += byteCount
    prev!!.push(prefix)
    return prefix
  }
```
`split(byteCount: Int)`涵数将Segment分割成两个Segment，前一个Segment包含[pos,pos+byteCount]部分的数据，后一个Segment包含[pos+byteCount,limit]部分的数据。
在涵数中，我们可以看到，split并不是简单的将Segment维护的Byte数组分割成两份，而是根据要分割的内容的大小，做不同的操作，`SHARE_MINIMUM`的值我们之前可以看到为1024。

- 当`byteCount >= SHARE_MINIMUM`，Segment的分割实际上是复制操作，两个Segment持有同一个Byte数组，并改变那两个Segment的pos与limit：

![image](https://note.youdao.com/yws/api/personal/file/641111277ADB44C8A191DEB210569358?method=download&shareKey=a0531dc7981bdc68cd348682394f27c5)

- 当`byteCount < SHARE_MINIMUM`时，则是SegmentPool中取出一个新的Segment（SegmentPool是Segment缓存池，我们之后再讲，这里的`take()`方法，我们假设取到的是一个新的Segment）,原本的Segment将pos至pos+byteCount的内容，复制给新的Segment：
![image](https://note.youdao.com/yws/api/personal/file/FF1F5A897A7F46289DE894D903D7A057?method=download&shareKey=0d9c5a20ed5ac596d008cfccb959c26a)

可以看到，这里Okio是对**避免复制数据**与**不能创建太多短的分享的Segment**的一种平衡。

**writeTo(sink: Segment, byteCount: Int)**

将一个Segment中的pos到pos+byteCoutn的数据写到另一个Segment中去
```
  fun writeTo(sink: Segment, byteCount: Int) {
    //检查目标Segment是否可写
    check(sink.owner) { "only owner can write" }
    if (sink.limit + byteCount > SIZE) {
      //这里表示写入的数据将会大于Segment中Byte数组的容量，
      //那么尝试将目标Segment的pos之前的数据丢掉
      
      //如果目标Segment被分享了，不能改变噢
      if (sink.shared) throw IllegalArgumentException()
      //如果将limit-pos+byteCount这样还是超过容量了，那么也失败！
      if (sink.limit + byteCount - sink.pos > SIZE) throw IllegalArgumentException()
      sink.data.copyInto(sink.data, startIndex = sink.pos, endIndex = sink.limit)
      sink.limit -= sink.pos
      sink.pos = 0
    }
    
    //到这里，表示空间放得下byteCount个数据，可以开始复制了
    data.copyInto(sink.data, destinationOffset = sink.limit, startIndex = pos, endIndex = pos + byteCount)
    //改变目标Segment的limit
    sink.limit += byteCount
    //改变当前Segment的pos
    pos += byteCount
  }

```

**compact()**

compact()顾名思义，是使Segment合并
```
  fun compact() {
    //前一个Segment与当前不是同一个则可以合并
    check(prev !== this) { "cannot compact" }
    //判断前一个Segment是否可写
    if (!prev!!.owner) return
    //byteCount为当前Segment未读的数据
    val byteCount = limit - pos
    //前一个Segment的可用空间，如果被分享了，则空间大小为limit到Size的这一段
    //如果没有被分享，那么0-pos的部分也可以写入，writeTo会将未读数据迁移到开头
    val availableByteCount = SIZE - prev!!.limit + if (prev!!.shared) 0 else prev!!.pos
    if (byteCount > availableByteCount) return writable space.
    writeTo(prev!!, byteCount)
    //链表弹出当前的Segment
    pop()
    //SegmentPool回收当前Segment
    SegmentPool.recycle(this)
  }
```
合并Segment有什么好处呢？Segmetn内部是字节数组，在Java中分配字节数组时，运行时必须在将请求的数组返回给您之前将其清空，合并Segment可以减少运行时分配内存，并且被合并后，多出来的Segment，被回收到Pool中，等待下次使用，减少GC。

#### SegmentPool

SegmentPool是Segment的缓存池，它内部的实现是一条Segment的单向链表，Segment的大小为64kb。

```
internal object SegmentPool {
  /** The maximum number of bytes to pool.  */
  // TODO: Is 64 KiB a good maximum size? Do we ever have that many idle segments?
  const val MAX_SIZE = 64 * 1024L // 64 KiB.

  /** Singly-linked list of segments.  */
  var next: Segment? = null

  /** Total bytes in this pool.  */
  var byteCount = 0L

  fun take(): Segment {
    synchronized(this) {
      next?.let { result ->
        next = result.next
        result.next = null
        byteCount -= Segment.SIZE
        return result
      }
    }
    return Segment() // Pool is empty. Don't zero-fill while holding a lock.
  }

  fun recycle(segment: Segment) {
    require(segment.next == null && segment.prev == null)
    if (segment.shared) return // This segment cannot be recycled.

    synchronized(this) {
      if (byteCount + Segment.SIZE > MAX_SIZE) return // Pool is full.
      byteCount += Segment.SIZE
      segment.next = next
      segment.limit = 0
      segment.pos = segment.limit
      next = segment
    }
  }
}
```
我们可以看到SegmentPool只有两个方法:

- `take()`表示从池中取出Segment，若池中没有，则创建新的Segment。**Buffer并不会直接创建Segment，只会从SegmentPool中取Segment。**
- `recycle()`则回收Segment入池，作为链表的表头。被回收的Segment需要没有上下链的关系，即next与prev都为null，当大小超过SegmentPool的容量时，就不会回收。回收时，Segment的limit与pos都设为0。**这里我们可以知道，当一个被读取完数据的Segment被回收时，Segment内部的Byte数组应当是依然有数据的，但是由于pos与limit为0，这些数据无法访问，并且可以被替换，相当于空的数组**

SegmentPool比较简单，内部维护一个单向Segment链表，方便回收，减少GC。我们前面提到过在Segment的`split()`操作时，小数据的分割，会从SegmentPool中通过`take()`得到Segment。那么，除了这里，SegmetnPool什么时候回收Segment和提供新的Segment呢？

点击SegmentPool中take()方法的调用，我们发现，只有一处地方调用了该方法：

**Buffer.writableSegment**

```
internal actual fun writableSegment(minimumCapacity: Int): Segment =
    commonWritableSegment(minimumCapacity)

internal inline fun Buffer.commonWritableSegment(minimumCapacity: Int): Segment {
  require(minimumCapacity >= 1 && minimumCapacity <= Segment.SIZE) { "unexpected capacity" }

  if (head == null) {
    //当没有Segment时，从SegmentPool中取出第一个Segment作为head
    val result = SegmentPool.take() // Acquire a first segment.
    head = result
    //将head的prev和next都指向自身
    result.prev = result
    result.next = result
    return result
  }
  //取尾部的Segment
  var tail = head!!.prev
  if (tail!!.limit + minimumCapacity > Segment.SIZE || !tail.owner) {
    //若尾部的Segment空间不够 或者不是可写的时
    //从SegmentPool中取新的Segment加入到链表中
    tail = tail.push(SegmentPool.take()) 
  }
  return tail
}

```

看上面的代码，`writableSegment(minimumCapacity: Int)`的作用是获得一个可写的空间至少为minimumCapacity的Segment，该方法调用了Buffer的扩展方法`commonWritableSegment(minimumCapacity: Int)`。我们可以看到当Buffer中head为null时（即Buffer中还没有数据时，没有Segment），调用了`SegmentPool.take()`，并将获得的Segment的prev和next都指向了自身。这里我们看出，**Segment不仅是双向链表，还是循环的双向链表**。而当head不为null时，可以由head!!.prev获得链表的尾部Segment，该Segment的满足条件时，直接返回，若不满足条件就从SegmentPool取出新的Segment加入到链表的尾部，并返回。

看到这里，我们其实依然不清楚，到底何时，Buffer会从SegmentPool中获取Segment以及什么时候回收Segment，让我们先回到Buffer中来。

#### 回到Buffer

Buffer作为一个缓存区，当然是既可以往里面写入缓存数据，也可以读取里面的缓存数据，我们先看Buffer提供的写操作：

![image](https://note.youdao.com/yws/api/personal/file/E2584EE4FF8C4D51AAB339148BB1FBBD?method=download&shareKey=a66f1c13252045da36f9a66fc88af6b4)

我们可以看到，写入Buffer的操作有许多种，包括写入ByteString、另一个Buffer、Byte数组、String等等数据到Buffer中

相信大家还记得上文提过的Buffer的特点：**它将数据从一个缓冲区移到另一个缓冲区的速度很快**，所以我们选择分析一下`write(source: Buffer, byteCount: Long)`，来看看到底时如何快的。

```
internal inline fun Buffer.commonWrite(source: Buffer, byteCount: Long) {
  var byteCount = byteCount

   require(source !== this) { "source == this" }
  checkOffsetAndCount(source.size, 0, byteCount)

  while (byteCount > 0L) {

    if (byteCount < source.head!!.limit - source.head!!.pos) {
      //limit-pos为Segment的未读区域，当byteCount小于目标Buffer的头Segment的未读区域大小时
      val tail = if (head != null) head!!.prev else null
      //得到当前Buffer中Segment链表的尾部
      if (tail != null && tail.owner &&
        byteCount + tail.limit - (if (tail.shared) 0 else tail.pos) <= Segment.SIZE) {
        //尾部Segment不为null，并且可写，并且可写区域大小大于等于byteCount时
        //使用Segment的writeTo，将数据写入尾部Segment中
        source.head!!.writeTo(tail, byteCount.toInt())
        //目标Buffer的size减少byteCount
        source.size -= byteCount
        //当前Buffer的size增加byteCount
        size += byteCount
        //写入完成，返回
        return
      } else {
        //当尾部Segment不满足条件时，直接分割目标Buffer的头Segment
        //为转移Segment做准备
        source.head = source.head!!.split(byteCount.toInt())
      }
    }

    //下面这部分代码时将目标Buffer的头Segment移除，并加到当前Buffer的尾部
    val segmentToMove = source.head
    val movedByteCount = (segmentToMove!!.limit - segmentToMove.pos).toLong()
    //将目标Buffer的头Segment从链表移除，并使目标Buffer的head指向新的下一个Segment
    source.head = segmentToMove.pop()
    if (head == null) {
      //如果当前Buffer没有Segment，直接转移segmentToMove的所有权
      head = segmentToMove
      segmentToMove.prev = segmentToMove
      segmentToMove.next = segmentToMove.prev
    } else {
      //如果当前Buffer有，将segmentToMove添加到尾部
      var tail = head!!.prev
      tail = tail!!.push(segmentToMove)
      //尝试合并尾部的Segment，节省空间
      tail.compact()
    }
    source.size -= movedByteCount
    size += movedByteCount
    byteCount -= movedByteCount
  }
}

```
可以看到，当我们转移Buffer中的数据到另一个Buffer时，Okio做的是将一个Buffer中的头部数据写入到另一个Buffer的尾部。但是，这里并不是一味的复制数据，当另一个Buffer的尾部Segment能容纳数据时，则直接复制数据；否则，则是通过转移Segment的方式。这样对数据的操作更加的高效！

我们再看一下比较常用的`writeUtf8(string: String)`
```
  override fun writeUtf8(string: String) = writeUtf8(string, 0, string.length)

  override fun writeUtf8(string: String, beginIndex: Int, endIndex: Int): Buffer {
    require(beginIndex >= 0) { "beginIndex < 0: $beginIndex" }
    require(endIndex >= beginIndex) { "endIndex < beginIndex: $endIndex < $beginIndex" }
    require(endIndex <= string.length) { "endIndex > string.length: $endIndex > ${string.length}" }

    var i = beginIndex
    while (i < endIndex) {
      var c = string[i].toInt()

      when {
        c < 0x80 -> {
          val tail = writableSegment(1)
          val data = tail.data
          //segmentOffset这个变量是为了表示limit和i之间的关系
          //即limit = segmentOffset + i
          val segmentOffset = tail.limit - i
          //Segment.SIZE - segmentOffset = Segment.SIZE - limit + i
          //Segment.SIZE - limit表示Segment的可写入空间大小
          //但是我们是从i开始的，所以需要 + i
          //若runLimit为endIndex,表示String的长度小于Segment的可写入空间大小
          val runLimit = minOf(endIndex, Segment.SIZE - segmentOffset)
          //segmentOffset + i++ 即为  tail.limit++
          data[segmentOffset + i++] = c.toByte() // 0xxxxxxx
          //从i到Segment.SIZE - limit + i
          while (i < runLimit) {
            c = string[i].toInt()
            if (c >= 0x80) break
            //segmentOffset + i++ 即为  tail.limit++
            data[segmentOffset + i++] = c.toByte() // 0xxxxxxx
          }
          // = i + tail.limit  - (之前的i) - tail.limit
          // = (当前的i) - （之前的i） = i的改变大小
          val runSize = i + segmentOffset - tail.limit
          tail.limit += runSize
          size += runSize.toLong()
        }
        c < 0x800 -> {
          val tail = writableSegment(2)
          tail.data[tail.limit    ] = (c shr 6          or 0xc0).toByte() 
          tail.data[tail.limit + 1] = (c       and 0x3f or 0x80).toByte() 
          tail.limit += 2
          size += 2L
          i++
        }
        c < 0xd800 || c > 0xdfff -> {
          val tail = writableSegment(3)
          tail.data[tail.limit    ] = (c shr 12          or 0xe0).toByte() 
          tail.data[tail.limit + 1] = (c shr  6 and 0x3f or 0x80).toByte() 
          tail.data[tail.limit + 2] = (c        and 0x3f or 0x80).toByte() 
          tail.limit += 3
          size += 3L
          i++
        }
        else -> {
          val low = (if (i + 1 < endIndex) string[i + 1].toInt() else 0)
          if (c > 0xdbff || low !in 0xdc00..0xdfff) {
            writeByte('?'.toInt())
            i++
          } else {
            val codePoint = 0x010000 + (c and 0x03ff shl 10 or (low and 0x03ff))
            val tail = writableSegment(4)
            tail.data[tail.limit    ] = (codePoint shr 18          or 0xf0).toByte() // 11110xxx
            tail.data[tail.limit + 1] = (codePoint shr 12 and 0x3f or 0x80).toByte() // 10xxxxxx
            tail.data[tail.limit + 2] = (codePoint shr  6 and 0x3f or 0x80).toByte() // 10xxyyyy
            tail.data[tail.limit + 3] = (codePoint        and 0x3f or 0x80).toByte() // 10yyyyyy
  
            tail.limit += 4
            size += 4L
            i += 2
          }
        }
      }
    }
    return this
  }
```
根据UTF8编码，一个字符由1-4位字节表示，可以看出，`writeUtf8(string: String)`在循环中，将不同位数的字符，分开处理，通过`writableSegment(int: Int)`获取Buffer中末尾的Segment，若没有Segment或者Segment的可写区域大小不满足，则重新创建，然后写入数据。

有趣的是Okio处理1位字节的字符时，并没有像其他的一样，而是在条件分支中，继续循环执行写入操作，直到条件不满足，或者写入完成。这样的写法，虽然让人难以理解，但是却对性能有着较好的提高，在官方的注释中写道大约提升了约4倍的性能！

我们再来看Buffer的读操作：

![image](https://note.youdao.com/yws/api/personal/file/4975623162B84AA395860DC971F90D9F?method=download&shareKey=0731434497b7762df32013d4213bfd3d)

Buffer提供了丰富的read涵数，这里我们选择看一下`readUtf8()`

```
  override fun readUtf8() = readString(size, Charsets.UTF_8)

  @Throws(EOFException::class)
  override fun readUtf8(byteCount: Long) = readString(byteCount, Charsets.UTF_8)

  override fun readString(charset: Charset) = readString(size, charset)

  @Throws(EOFException::class)
  override fun readString(byteCount: Long, charset: Charset): String {
    require(byteCount >= 0 && byteCount <= Integer.MAX_VALUE) { "byteCount: $byteCount" }
    if (size < byteCount) throw EOFException()
    if (byteCount == 0L) return ""
    //走到这里head不可能为null噢
    val s = head!!
    if (s.pos + byteCount > s.limit) {
      //如果Segment的可读区域大小小于要读取的大小，那么使用readByteArray
      return String(readByteArray(byteCount), charset)
    }
    //直接使用String的构造方法，得到响应编码的字符串
    val result = String(s.data, s.pos, byteCount.toInt(), charset)
    s.pos += byteCount.toInt()
    size -= byteCount
    
    if (s.pos == s.limit) {
      //若Segment的数据读取完毕，弹出Segment并回收到SegmentPool中
      head = s.pop()
      SegmentPool.recycle(s)
    }

    return result
  }

```

Buffer的读取操作比较简单，当Buffer中每一个Segment读取完毕时，会回收该Segment。

#### 总结

1. ByteString内部是字节数组，ByteString是不可变的，但是通过非常规手段（反射和IO流），也是可以改变的。
2. Buffer内部是Segment的循环双向链表，Segment内部是字节数组。
3. SegmentPool管理Segment的创建与回收，Buffer需要Segment时，都从SegmentPool的`take()`方法获得，并且在读取完数据后，通过`recycle()`方法回收。
4. Buffer之间数据的转移，一般是通过对Segment的所有权的转移，尽量降低数据的复制与清空，减少性能消耗。Buffer通过Segment的合并`compact()`，来减少资源的消耗，防止大量空闲的Segment产生。
5. 与Java.NIO比较，Okio的Buffer利用Segment，将数据分成一块一块的，提高了数据操作的灵活性，通过SegmentPool管理Segment更加有效的利用了资源。