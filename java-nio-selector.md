# Java NIO Selector\(选择器\)

Selector是Java NIO的一个重要组件，用来检查一个或者多个channel，检查channel是否已经准备好读和写。这样就能让一个线程同时管理多个channel了，例如多个网络连接。

## 为什么使用Selector

使用Selector的主要优势是可以使用单线程来处理多个channel，甚至所有channel。多线程间切换时会多余的消耗操作系统的资源，例如内存和cpu。所以使用较少的线程，系统的整体性能会表现得更好。

下面就是一个线程操作三个channel的示意图：

![](/assets/1.png)

## 创建Selector

直接调用静态方法open\(\)就可以创建一个Selector：

```
Selector selector = Selector.open();
```

## 向Selector注册Channel

为了能让Channel和Selector一起使用，需要向Selector注册Channel，可以通过调用类似`SelectableChannel.register()`的方法来实现，例如下面：

```
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```

当使用Selector时，Channel必须要是非阻塞模式。因为FileChannel不能切换到非阻塞模式，所以FileChannel不能和Selector一起使用。

注意register\(\)方法的第二个参数，该参数表示Selector监听Channel的事件，只有触发指定的Channel事件时，Selector才会去处理该Channel的数据。可以监听四种不同的事件：

* 1.Connect
* 2.Accept
* 3.Read
* 4.Write

当Channel触发了某个事件时，也就意味着它已经准备好处理该事件了。当channel触发了connect事件后，意味着已经连接成功；当触发了accept事件后，意味着一个server socket已经成功的接收了客户端的请求；当channel触发了read事件后，意味着channel已经准备好被读取数据了；当channel触发了write事件后，意味着channel已经准备好被写入数据了。

这四种事件，对应了SelectionKey类中的四个常量：

* 1.SelectionKey.OP\_CONNECT
* 2.SelectionKey.OP\_ACCEPT
* 3.SelectionKey.OP\_READ
* 4.SelectionKey.OP\_WRITE

如果想监听多个事件，可以使用`或`运算，例如：

```
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```

## SelectionKey

从前面的例子中可以看出，当调用Channel的register\(\)方法来把channel注册到Selector上时，会返回一个SelectionKey对象。SelectionKey对象有下面这几个属性：

* Interest set

* Read set

* Channel

* Selector

* Attached object（附加对象，可选）

### Interest set

“Interest set”表示在把channel注册到selector上时，监听的所有事件集合。可以通过SelectionKey对象来获取监听的事件集合，也可以设置新的监听集合。

```
//获取监听的时间集合
int interestSet = selectionKey.interestOps();

//判断是否监听了指定的事件
boolean isInterestedInAccept  = interestSet & SelectionKey.OP_ACCEPT;
boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT;
boolean isInterestedInRead    = interestSet & SelectionKey.OP_READ;
boolean isInterestedInWrite   = interestSet & SelectionKey.OP_WRITE;
```

其实在这里不难发现，SelectionKey.OP\_READ对应的值是\(0x00001\);SelectionKey.OP\_WRITE对应的值是\(0x00100\);SelectionKey.OP\_CONNECT对应的值是\(0x01000\);SelectionKey.OP\_ACCEPT对应的值是\(0x10000\)

所以，在把channel注册到Selector上时，通过`或运算`设置监听多个事件；在拿到`Interest set`的时候，可以通过`与运算`来检验是否监听了某个事件。

### Ready Set

“Ready Set”是指 Channel已经准备好了处理Selector所监听的事件集合。可以这样来获取到ready set：

```
int readySet = selectionKey.readyOps();
```

在拿到readySet后，我们需要判断此时channel现在准备好了处理什么事件，我们也可以通过下面的方法来判断：

```
selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();
```

### Channel + Selector

通过SelectionKey来访问channel和selector比较简单：

```
Channel  channel  = selectionKey.channel();

Selector selector = selectionKey.selector();
```

### Attaching Objects\(附加对象\)

我们可以给SelectionKey附加一个对象，用来标识某个给定的channel。例如我们可以使用channel对应的buffer对象作为附加对象，或者使用一个包含其它信息的对象。下面是使用SelectionKey来添加附加对象：

```
selectionKey.attach(theObject);

Object attachedObj = selectionKey.attachment();
```

可以在向Selector注册channel时添加附加对象：

```
SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);
```

## 通过Selector来选择Channel

当开发人员向Selector注册了一个或多个channel时，此时可以调用Selector的select\(\)方法，该方法会返回已经准备就绪的channel数。例如，监听了channel的reading事件，当该channel触发了reading事件后你调用了Selector的select\(\)方法，那么你将会得到触发了reading事件的channel个数。

下面是select\(\)方法的几个版本

* int select\(\)

注册在Selector上的channel中，至少有一个channel被监听的事件已经准备就绪。否则线程阻塞。

* int select\(long timeout\)

返回值意义和select\(\)方法一样，但是阻塞时长最多不超过timeout毫秒。

* int selectNow\(\)

立刻返回准备就绪的Channel个数，不阻塞。

> select\(\)方法返回的int值是指自上次调用select\(\)方法后，到目前为止准备就绪的channel数。
>
> 例如：第一次调用select\(\)方法时返回了1，表示有一个channel触发了事件，并准备就绪，现在不处理该channel；再次调用select\(\)，并返回1，表示自从上次调用了select后，又有新的channel触发了事件并准备就绪。此时一共有两个channel准备就绪。

### selectedKeys\(\)

调用select\(\)方法并返回了现在又出现多少个channel准备就绪，开发人员可以通过调用方法selectedKeys\(\)来得到“selected key set”，以此访问准备就绪的channel。代码如下：

```
Set<SelectionKey> selectedKeys = selector.selectedKeys();
```

当调用Channel.register\(\)方法来向Selector注册channel时，将会返回一个SelectionKey对象。这个SelectionKey对象相当于channel的注册器，可以通过调用selector的selectedKeySet\(\)方法来获得这个注册器。

下面通过“selected key set”来访问已经准备就绪的channel：

```
Set<SelectionKey> selectedKeys = selector.selectedKeys();

Iterator<SelectionKey> keyIterator = selectedKeys.iterator();

while(keyIterator.hasNext()) {

    SelectionKey key = keyIterator.next();

    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.

    } else if (key.isConnectable()) {
        // a connection was established with a remote server.

    } else if (key.isReadable()) {
        // a channel is ready for reading

    } else if (key.isWritable()) {
        // a channel is ready for writing
    }

    keyIterator.remove();
}
```

> 注意：在每次循环最后都会调用keyIterator.remove\(\)来把当前的SelectionKey对象从Selector中移除掉，每次处理完channel的事件后，都需要手动去移除SelectionKey对象。当channel再次准备就绪时，Selector将会再次向“selected key set”添加SelectionKey。

SelectionKey.channel\(\)方法返回的是一个SelectableChannel抽象类的指针，在使用的时候需要使用强制类型转换的方式把返回的对象转为你所需要的类型类型，例如：ServerSocketChannel或是SocketChannel等等。

## wakeUp\(\)方法

当某个线程执行了Selector.select\(\)方法后，将会进入阻塞，直到有新的channel进入准备就绪的状态。如果现在想要该线程退出阻塞状态，那么可以在另外一个线程中执行Selector的wakeUp\(\)方法。

如果在某个线程中先执行了Selector的wakeUp\(\)方法，然后首次执行Selector的select\(\)方法将不会阻塞。Selector的wakeUp\(\)方法多次执行的效果和执行一次的效果相同。

## close\(\)方法

当停止使用Selector来处理多个channel时，可以调用Selector的close\(\)方法。调用了close\(\)方法后，所有的SelectionKey对象都不可用了。该方法只是关闭了Selector，但channel并没有关闭。

## Selector实例

下面是一个比较完整的例子，包括了打开一个选择器、注册channel，并监控选择器上的事件\(accept，connect，read，write\):

```
Selector selector = Selector.open();

channel.configureBlocking(false);

SelectionKey key = channel.register(selector, SelectionKey.OP_READ);


while(true) {

  int readyChannels = selector.select();

  if(readyChannels == 0) continue;


  Set<SelectionKey> selectedKeys = selector.selectedKeys();

  Iterator<SelectionKey> keyIterator = selectedKeys.iterator();

  while(keyIterator.hasNext()) {

    SelectionKey key = keyIterator.next();

    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.

    } else if (key.isConnectable()) {
        // a connection was established with a remote server.

    } else if (key.isReadable()) {
        // a channel is ready for reading

    } else if (key.isWritable()) {
        // a channel is ready for writing
    }

    keyIterator.remove();
  }
}
```



