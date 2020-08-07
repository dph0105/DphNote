Timeout是Okio提供的对一项任务的超时机制，这个类提供两种互补的机制来定义超时策略：
1. **Timeouts**指定等待单个操作完成的最大时间。超时是通常用于检测网络分区等问题。例如，如果远程对等端10秒内没有返回任何数据，我们可以假设对等端不可用。
2. **Deadline**指定在一个作业上花费的最大时间，由一个或多个操作组成。使用设定工作投入时间上限的最后期限。例如，一个注重电池用量的应用可能会限制预加载内容的时间。

### Timeout

我们来看Timeout的代码：

```
open class Timeout {

  private var hasDeadline = false
  private var deadlineNanoTime = 0L
  private var timeoutNanos = 0L

  open fun timeout(timeout: Long, unit: TimeUnit): Timeout {
    require(timeout >= 0) { "timeout < 0: $timeout" }
    timeoutNanos = unit.toNanos(timeout)
    return this
  }

  open fun timeoutNanos(): Long = timeoutNanos

  open fun hasDeadline(): Boolean = hasDeadline

  open fun deadlineNanoTime(): Long {
    check(hasDeadline) { "No deadline" }
    return deadlineNanoTime
  }

  open fun deadlineNanoTime(deadlineNanoTime: Long): Timeout {
    this.hasDeadline = true
    this.deadlineNanoTime = deadlineNanoTime
    return this
  }

  fun deadline(duration: Long, unit: TimeUnit): Timeout {
    require(duration > 0) { "duration <= 0: $duration" }
    return deadlineNanoTime(System.nanoTime() + unit.toNanos(duration))
  }

  open fun clearTimeout(): Timeout {
    timeoutNanos = 0
    return this
  }

  open fun clearDeadline(): Timeout {
    hasDeadline = false
    return this
  }

  @Throws(IOException::class)
  open fun throwIfReached() {
   //...
  }


  @Throws(InterruptedIOException::class)
  fun waitUntilNotified(monitor: Any) {
    //...
  }

  
  inline fun intersectWith(other: Timeout, block: () -> Unit) {
   //...
  }
}

```
Timeout维护三个变量：
1. `hasDeadline：Boolean`,表示当定义了deadline时为true，表示超时机制中设置了最后期限。
2. `deadlineNanoTime：Long`，表示设置的任务的最后期限
3. `timeoutNanos：Long`，表示超时时间

针对这三个变量，**Timeout**提供了一系列的设置和获取的简单函数，如：`timeoutNanos()`， `deadlineNanoTime(deadlineNanoTime: Long)`， `clearTimeout()`等等，大家可以自己看。我们这里主要分析**Timeout**提供的三个主要函数。

#### 首先看`throwIfReached()`


```
  @Throws(IOException::class)
  open fun throwIfReached() {
    if (Thread.interrupted()) {
      Thread.currentThread().interrupt() // Retain interrupted status.
      throw InterruptedIOException("interrupted")
    }

    if (hasDeadline && deadlineNanoTime - System.nanoTime() <= 0) {
      throw InterruptedIOException("deadline reached")
    }
  }
```

该方法的作用当到达DeadLine时或者当前线程中断时，抛出中断异常。该方法的实现很简单，我们就不细讲了，但是大家有没有对该方法有点熟悉？是的，在我们讲Sink与Source时，曾经遇到过该方法：


```
private class OutputStreamSink(
  private val out: OutputStream,
  private val timeout: Timeout
) : Sink {
  override fun write(source: Buffer, byteCount: Long) {
    checkOffsetAndCount(source.size, 0, byteCount)
    var remaining = byteCount
    while (remaining > 0) {
      timeout.throwIfReached()
      //使用outputstream来进行写操作
      //......
    }
  }
    //......
}

 private class InputStreamSource(
      private val input: InputStream,
      private val timeout: Timeout
    ) : Source {
    
      override fun read(sink: Buffer, byteCount: Long): Long {
        if (byteCount == 0L) return 0
        require(byteCount >= 0) { "byteCount < 0: $byteCount" }
        try {
          timeout.throwIfReached()
          //......
        } catch (e: AssertionError) {
          if (e.isAndroidGetsocknameError) throw IOException(e)
          throw e
        }
      }
  
    }

```

在`OutputStreamSink`与`InputStreamSource`的具体实现中，读写都会调用该方法来判断任务是否到达截止时间。



```
fun OutputStream.sink(): Sink = OutputStreamSink(this, Timeout())

fun InputStream.source(): Source = InputStreamSource(this, Timeout())
```

但是我们可以看到，传入`OutputStreamSink`与`InputStreamSource`中的timeout是直接构造的一个Timeout对象，即`hasDeadline = false`，因此，在`OutputStreamSink`与`InputStreamSource`中，只要他们的线程不中断，就不会触发超时抛出中断异常。

#### `waitUntilNotified(monitor: Any)`

```
 @Throws(InterruptedIOException::class)
  fun waitUntilNotified(monitor: Any) {
    try {
      val hasDeadline = hasDeadline()
      val timeoutNanos = timeoutNanos()
    
      if (!hasDeadline && timeoutNanos == 0L) {
      //当没有deadline和超时时间时，可以直接wait()
        (monitor as Object).wait() // There is no timeout: wait forever.
        return
      }

      // 有超时时，计算哪个距离限制的时间最近，即deadline和timeout谁先到达
      val start = System.nanoTime()
      val waitNanos = if (hasDeadline && timeoutNanos != 0L) {
        val deadlineNanos = deadlineNanoTime() - start
        minOf(timeoutNanos, deadlineNanos)
      } else if (hasDeadline) {
        deadlineNanoTime() - start
      } else {
        timeoutNanos
      }
      

      //计算花费了多少时间，和最多wait多久，设置的超时时间即最多wait的时间
      var elapsedNanos = 0L
      if (waitNanos > 0L) {
        val waitMillis = waitNanos / 1000000L
        (monitor as Object).wait(waitMillis, (waitNanos - waitMillis * 1000000L).toInt())
        elapsedNanos = System.nanoTime() - start
      }

      //若elapsedNanos >= waitNanos，代表已经超时，抛出中断异常
      if (elapsedNanos >= waitNanos) {
        throw InterruptedIOException("timeout")
      }
    } catch (e: InterruptedException) {
      Thread.currentThread().interrupt() // Retain interrupted status.
      throw InterruptedIOException("interrupted")
    }
  }

```

我们知道在Java的线程中，通过在synchronized方法或者代码块中调用`Object.wait()`方法，可以使线程阻塞，知道其他线程调用`Object.notify()`,才有可能重新运行。`waitUntilNotified(monitor: Any)`是优化了在**具有超时条件**下，wait()方法的调用。首先，`waitUntilNotified(monitor: Any)`同样需要**在synchronized方法或者代码块中调用**。进入该方法，首先判断是否设置了超时条件，若没有，则直接wait(),若有设置超时信息，则要计算wait的时间，以及到wait的过程花费的时间。最后，若花费的时间已经导致超时，则直接抛出中断异常。


#### `intersectWith(other: Timeout, block: () -> Unit)`
```
 inline fun intersectWith(other: Timeout, block: () -> Unit) {
    val originalTimeout = this.timeoutNanos()
    this.timeout(minTimeout(other.timeoutNanos(), this.timeoutNanos()), TimeUnit.NANOSECONDS)

    if (this.hasDeadline()) {
      val originalDeadline = this.deadlineNanoTime()
      if (other.hasDeadline()) {
        this.deadlineNanoTime(Math.min(this.deadlineNanoTime(), other.deadlineNanoTime()))
      }
      try {
        block()
      } finally {
        this.timeout(originalTimeout, TimeUnit.NANOSECONDS)
        if (other.hasDeadline()) {
          this.deadlineNanoTime(originalDeadline)
        }
      }
    } else {
      if (other.hasDeadline()) {
        this.deadlineNanoTime(other.deadlineNanoTime())
      }
      try {
        block()
      } finally {
        this.timeout(originalTimeout, TimeUnit.NANOSECONDS)
        if (other.hasDeadline()) {
          this.clearDeadline()
        }
      }
    }
  }

```
`intersectWith(other: Timeout, block: () -> Unit)`用于临时使用其他Timeout与当前Timeout的超时时间的交集，即哪个时间最近选哪个，并运行传入的block函数，运行完成后，将超时时间改回原始的设置。

