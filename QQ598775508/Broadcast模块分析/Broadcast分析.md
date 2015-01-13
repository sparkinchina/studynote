#[Broadcast模块分析](http://www.tuicool.com/articles/yUfqay)#

## 一. BroadcastManager
   BroadcastManager类中包含有一个BroadFactory对象的引用，大部分操作通过BroadFactory中的方法来实现。BroadcastFactory是一个Trait，有两个直接子类TorrentBroadcastFactory、HttpBroadcastFactory。这两个子类实现了对HttpBroadcast、TorrentBroadcast的封装，而后面两个又同时集成了Broadcast抽象类。
### 1. BroadcastManager的初始化
SparkContext初始化时会创建SparkEnv对象env，这个过程中会调用BroadcastManager的构造方法返回一个对象作为env的成员变量存在：

       val broadcastManager = new BroadcastManager(isDriver, conf, securityManager)

构造BlockcastManager对象时会调用initialize方法，主要根据配置初始化broadcastFactory变量，并且调用其initialize方法：

       val broadcastFactoryClass =
          conf.get("spark.broadcast.factory", "org.apache.spark.broadcast.TorrentBroadcastFactory")

        broadcastFactory =
          Class.forName(broadcastFactoryClass).newInstance.asInstanceOf[BroadcastFactory]

        // Initialize appropriate BroadcastFactory and BroadcastObject
        broadcastFactory.initialize(isDriver, conf, securityManager)

两个工厂类的initialize方法都是对其相应实体类的initialize方法的调用,如HttpBroadcast类：

      override def initialize(isDriver: Boolean, conf: SparkConf, securityMgr: SecurityManager) {
        HttpBroadcast.initialize(isDriver, conf, securityMgr)
      }

下面分开两个类来看：
#### HttpBroadcast的initialize方法：
       def initialize(isDriver: Boolean, conf: SparkConf, securityMgr: SecurityManager) {
         synchronized {
           if (!initialized) {
             bufferSize = conf.getInt("spark.buffer.size", 65536)
             compress = conf.getBoolean("spark.broadcast.compress", true)
             securityManager = securityMgr
             if (isDriver) {
     //          根据conf在driver创建一个httpServer，并启动
               createServer(conf)
               conf.set("spark.httpBroadcast.uri",  serverUri)
             }
             serverUri = conf.get("spark.httpBroadcast.uri")
     //        创建一个对象，定时清理过时的元数据
             cleaner = new MetadataCleaner(MetadataCleanerType.HTTP_BROADCAST, cleanup, conf)
             compressionCodec = CompressionCodec.createCodec(conf)
             initialized = true
           }
         }
       }

#### TorrentBroadcast的initialize方法：
Spark1.2以后，TorrentBroadcast中并没有显式定义的initialize函数了。这也可以看出BTTorrent和Http的区别，Torrent的处理方式就是p2p、去中心化；而Http方式就是中心化服务，需要启动服务来接收请求。

## 二. 创建broadcast变量
调用SparkContext中的 `def broadcast[T: ClassTag](value: T): Broadcast[T]`方法来初始化一个广播变量，实现如下：

      def broadcast[T: ClassTag](value: T): Broadcast[T] = {
        val bc = env.broadcastManager.newBroadcast[T](value, isLocal)
        val callSite = getCallSite
        logInfo("Created broadcast " + bc.id + " from " + callSite.shortForm)
        cleaner.foreach(_.registerBroadcastForCleanup(bc))
        bc
      }

### 1. HttpBroadcastFactory的newBroadcast
      override def newBroadcast[T: ClassTag](value_ : T, isLocal: Boolean, id: Long) =
        new HttpBroadcast[T](value_, isLocal, id)
可以看出是直接创建了一个HttpBroadcast对象，但是这个对象的构造函数里面是做了一些处理的：

          //将变量id和值放入blockManager，但并不通知master
         HttpBroadcast.synchronized {
           SparkEnv.get.blockManager.putSingle(
             blockId, value_, StorageLevel.MEMORY_AND_DISK, tellMaster = false)
         }

         if (!isLocal) {
       //    将对象值按照指定的压缩、序列化写入指定的文件,这个文件所在的目录即是HttpServer的资源目录
           HttpBroadcast.write(id, value_)
         }
### 2. TorrentBroadcastFactory的newBroadcast方法
         override def newBroadcast[T: ClassTag](value_ : T, isLocal: Boolean, id: Long) = {
           new TorrentBroadcast[T](value_, id)
         }
同样也是创建了一个TorrentBroadcast，在其构造函数中也做了两件事情：第一步和HttpBoradcast一样；第二件就是对数据进行分块，并以字节的方式放到block manager。

          private def writeBlocks(value: T): Int = {
            // Store a copy of the broadcast variable in the driver so that tasks run on the driver
            // do not create a duplicate copy of the broadcast variable's value.
            SparkEnv.get.blockManager.putSingle(broadcastId, value, StorageLevel.MEMORY_AND_DISK,
              tellMaster = false)
        //    下面方法将对象obj分块（默认块大小为4M）
            val blocks =
              TorrentBroadcast.blockifyObject(value, blockSize, SparkEnv.get.serializer, compressionCodec)
            blocks.zipWithIndex.foreach { case (block, i) =>
        //      将数据块放到block manager上去
              SparkEnv.get.blockManager.putBytes(
                BroadcastBlockId(id, "piece" + i),
                block,
                StorageLevel.MEMORY_AND_DISK_SER,
                tellMaster = true)
            }
            blocks.length
          }

## 三. 读取广播变量的值
通过调用bc.value来取得广播变量的值，其主要实现在反序列化方法readObject中，如HttpBroadcast中的readObject函数代码如下：

          private def readObject(in: ObjectInputStream): Unit = Utils.tryOrIOException {
            in.defaultReadObject()
            HttpBroadcast.synchronized {
        //      首先查看blockManager中是否已有，如有则直接取值，否则调用伴生对象的read方法进行读取
              SparkEnv.get.blockManager.getSingle(blockId) match {
                case Some(x) => value_ = x.asInstanceOf[T]
                case None => {
                  logInfo("Started reading broadcast variable " + id)
                  val start = System.nanoTime
                  value_ = HttpBroadcast.read[T](id)
                  /**
                   * 我们缓存广播变量的值到BlockManager以便后续的任务可以直接使用而不需要重新获取.
                   因为这些数据仅仅在本地使用，并且其他的node不需要获取这个block，所以我们不用通知Master
                   */
                  SparkEnv.get.blockManager.putSingle(
                    blockId, value_, StorageLevel.MEMORY_AND_DISK, tellMaster = false)
                  val time = (System.nanoTime - start) / 1e9
                  logInfo("Reading broadcast variable " + id + " took " + time + " s")
                }
              }
            }
          }
其中`value_ = HttpBroadcast.read[T](id)`的read方法如下：

          /**
           * 使用serverUri和block id对应的文件名直接开启一个HttpConnection将中心服务器上相应的数据取过来，
           * 并使用配置的压缩和序列化机制进行解压和反序列化
           * @param id
           * @tparam T
           * @return
           */
          private def read[T: ClassTag](id: Long): T = {
            logDebug("broadcast read server: " +  serverUri + " id: broadcast-" + id)
            val url = serverUri + "/" + BroadcastBlockId(id).name

            var uc: URLConnection = null
            if (securityManager.isAuthenticationEnabled()) {
              logDebug("broadcast security enabled")
              val newuri = Utils.constructURIForAuthentication(new URI(url), securityManager)
              uc = newuri.toURL.openConnection()
              uc.setConnectTimeout(httpReadTimeout)
              uc.setAllowUserInteraction(false)
            } else {
              logDebug("broadcast not using security")
              uc = new URL(url).openConnection()
              uc.setConnectTimeout(httpReadTimeout)
            }

            val in = {
              uc.setReadTimeout(httpReadTimeout)
              val inputStream = uc.getInputStream
              if (compress) {
                compressionCodec.compressedInputStream(inputStream)
              } else {
                new BufferedInputStream(inputStream, bufferSize)
              }
            }
            val ser = SparkEnv.get.serializer.newInstance()
            val serIn = ser.deserializeStream(in)
            val obj = serIn.readObject[T]()
            serIn.close()
            obj
          }

**这里可以看到，所有需要用到广播变量值的executor都需要去driver上pull广播变量的内容**
### TorrentBroadcast的反序列化
和Http一样，都是先查看blockManager中是否已经缓存，若没有，则调用receiveBroadcast方法。和写数据一样，同样是分成两个部分，首先取元数据信息，再根据元数据信息读取实际的block信息。
Torrent方法首先将广播变量数据分块，并存到BlockManager中；每个节点需要读取广播变量时，是分块读取，对每一块都读取其位置信息，然后随机选一个存有此块数据的节点进行get。每个节点读取后会将包含的快信息报告给BlockManagerMaster，这样本地节点也成为了这个广播网络中的一个peer。
与Http方式形成鲜明对比，这是一个去中心化的网络，只需要保持一个tracker即可，这就是p2p的思想。

## 四. 广播变量的清除
广播变量被创建时，紧接着有这样一句代码：

    cleaner.foreach(_.registerBroadcastForCleanup(bc))
cleaner是一个ContextCleaner对象，会将刚刚创建的广播变量注册到其中，调用栈为：

      def registerBroadcastForCleanup[T](broadcast: Broadcast[T]) {
        registerForCleanup(broadcast, CleanBroadcast(broadcast.id))
      }
      private def registerForCleanup(objectForCleanup: AnyRef, task: CleanupTask) {
        referenceBuffer += new CleanupTaskWeakReference(task, objectForCleanup, referenceQueue)
      }
等出现广播变量被[弱引用](http://blog.csdn.net/lyfi01/article/details/6415726)时则会执行`cleaner.foreach(_.start())` 。start方法中会调用keepCleaning方法，会遍历注册的清理任务（包括RDD、shuffle和broadcast），依次进行清理：

      private def keepCleaning(): Unit = Utils.logUncaughtExceptions {
        while (!stopped) {
          try {
            val reference = Option(referenceQueue.remove(ContextCleaner.REF_QUEUE_POLL_TIMEOUT))
              .map(_.asInstanceOf[CleanupTaskWeakReference])
            reference.map(_.task).foreach { task =>
              logDebug("Got cleaning task " + task)
              referenceBuffer -= reference.get
              task match {
                case CleanRDD(rddId) =>
                  doCleanupRDD(rddId, blocking = blockOnCleanupTasks)
                case CleanShuffle(shuffleId) =>
                  doCleanupShuffle(shuffleId, blocking = blockOnShuffleCleanupTasks)
                case CleanBroadcast(broadcastId) =>
                  doCleanupBroadcast(broadcastId, blocking = blockOnCleanupTasks)
              }
            }
          } catch {
            case e: Exception => logError("Error in cleaning thread", e)
          }
        }
      }
doCleanupBroadcast调用以下语句：

    def doCleanupBroadcast(broadcastId: Long, blocking: Boolean) {
      try {
        logDebug("Cleaning broadcast " + broadcastId)
        broadcastManager.unbroadcast(broadcastId, true, blocking)
        listeners.foreach(_.broadcastCleaned(broadcastId))
        logInfo("Cleaned broadcast " + broadcastId)
      } catch {
        case e: Exception => logError("Error cleaning broadcast " + broadcastId, e)
      }
    }
     def unbroadcast(id: Long, removeFromDriver: Boolean, blocking: Boolean) {
       broadcastFactory.unbroadcast(id, removeFromDriver, blocking)
     }
每个工厂类调用其对应实体类的伴生对象的unbroadcast方法。

### HttpBroadcast中的变量清除
       / 清除Exexutor上所有的与HTTP广播数据有关的持久化数据块。
       * 如果removeFromDriver被设置为true，则同时清除Driver上的持久化数据块，删除相关的广播数据文件
       */
      def unpersist(id: Long, removeFromDriver: Boolean, blocking: Boolean) = synchronized {
        SparkEnv.get.blockManager.master.removeBroadcast(id, removeFromDriver, blocking)
        if (removeFromDriver) {
          val file = getFile(id)
          files.remove(file)
          deleteBroadcastFile(file)
        }
      }
### TorrentBroadcast中的变量清除
      def unpersist(id: Long, removeFromDriver: Boolean, blocking: Boolean) = {
        logDebug(s"Unpersisting TorrentBroadcast $id")
        SparkEnv.get.blockManager.master.removeBroadcast(id, removeFromDriver, blocking)
      }
## 五. 总结
Broadcast可以使用在executor端多次使用某个数据的场景（比如说字典），Http和Torrent两种方式对应传统的CS访问方式和P2P访问方式，当广播变量较大或者使用较频繁时，采用后者可以减少driver端的压力。










