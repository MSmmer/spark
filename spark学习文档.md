#SPARK学习文档

##Spark编程指南(Python版)
##Spark RDD API

====

##Spark编程指南(Python版)

===
###概述
从高层次上来看，每一个Spark应用都包含一个驱动程序，用于执行用户的main函数以及在集群上运行各种并行操作。Spark提供的主要抽象是弹性分布式数据集（RDD），这是一个包含诸多元素、被划分到不同节点上进行并行处理的数据集合。RDD通过打开HDFS（或其他hadoop支持的文件系统）上的一个文件、在驱动程序中打开一个已有的Scala集合或由其他RDD转换操作得到。用户可以要求Spark将RDD持久化到内存中，这样就可以有效地在并行操作中复用。另外，在节点发生错误时RDD可以自动恢复。

Spark提供的另一个抽象是可以在并行操作中使用的共享变量。在默认情况下，当Spark将一个函数转化成许多任务在不同的节点上运行的时候，对于所有在函数中使用的变量，每一个任务都会得到一个副本。有时，某一个变量需要在任务之间或任务与驱动程序之间共享。Spark支持两种共享变量：广播变量，用来将一个值缓存到所有节点的内存中；累加器，只能用于累加，比如计数器和求和。

这篇指南将展示这些特性在Spark支持的语言中是如何使用的（本文只翻译了Python部分）。如果你打开了Spark的交互命令行——bin/spark-shell的Scala命令行或bin/pyspark的Python命令行都可以——那么这篇文章你学习起来将是很容易的。
###连接Spark
Spark1.3.0只支持Python2.6或更高的版本（但不支持Python3）。它使用了标准的CPython解释器，所以诸如NumPy一类的C库也是可以使用的。

通过Spark目录下的bin/spark-submit脚本你可以在Python中运行Spark应用。这个脚本会载入Spark的Java/Scala库然后让你将应用提交到集群中。你可以执行bin/pyspark来打开Python的交互命令行。

如果你希望访问HDFS上的数据，你需要为你使用的HDFS版本建立一个PySpark连接。常见的HDFS版本标签都已经列在了这个第三方发行版页面。

最后，你需要将一些Spark的类import到你的程序中。加入如下这行：

```
from pyspark import SparkContext, SparkConf
```
###初始化Spark
在一个Spark程序中要做的第一件事就是创建一个SparkContext对象来告诉Spark如何连接一个集群。为了创建SparkContext，你首先需要创建一个SparkConf对象，这个对象会包含你的应用的一些相关信息。

```
conf = SparkConf().setAppName(appName).setMaster(master)
sc = SparkContext(conf=conf)
```
appName参数是在集群UI上显示的你的应用的名称。master是一个Spark、Mesos或YARN集群的URL,如果你在本地运行那么这个参数应该是特殊的”local”字符串。在实际使用中，当你在集群中运行你的程序，你一般不会把master参数写死在代码中，而是通过用spark-submit运行程序来获得这个参数。但是，在本地测试以及单元测试时，你仍需要自行传入”local”来运行Spark程序。
###使用命令行
在PySpark命令行中，一个特殊的集成在解释器里的SparkContext变量已经建立好了，变量名叫做sc。创建你自己的SparkContext不会起作用。你可以通过使用—master命令行参数来设置这个上下文连接的master主机，你也可以通过—py-files参数传递一个用逗号隔开的列表来将Python的.zip、.egg或.py文件添加到运行时路径中。你还可以通过—package参数传递一个用逗号隔开的maven列表来给这个命令行会话添加依赖（比如Spark的包）。任何额外的包含依赖包的仓库（比如SonaType）都可以通过传给—repositorys参数来添加进去。

Spark包的所有Python依赖（列在这个包的requirements.txt文件中）在必要时都必须通过pip手动安装。
比如，使用四核来运行bin/pyspark应当输入这个命令：

```
$ ./bin/pyspark --master local[4]
```
又比如，把code.py文件添加到搜索路径中（为了能够import在程序中），应当使用这条命令：

```
$ ./bin/pyspark --master local[4] --py-files code.py
```
想要了解命令行选项的完整信息请执行pyspark --help命令。在这些场景下，pyspark会触发一个更通用的spark-submit脚本

在IPython这个加强的Python解释器中运行PySpark也是可行的。PySpark可以在1.0.0或更高版本的IPython上运行。为了使用IPython，必须在运行bin/pyspark时将PYSPARK_DRIVER_PYTHON变量设置为ipython，就像这样：

```
$ PYSPARK_DRIVER_PYTHON=ipython ./bin/pyspark
```
你还可以通过设置PYSPARK_DRIVER_PYTHON_OPTS来自省定制ipython。比如，在运行IPython Notebook
时开启PyLab图形支持应该使用这条命令：

```
$ PYSPARK_DRIVER_PYTHON=ipython PYSPARK_DRIVER_PYTHON_OPTS="notebook --pylab inline" ./bin/pyspark
```
###弹性分布式数据集（RDD）
Spark是以RDD概念为中心运行的。RDD是一个容错的、可以被并行操作的元素集合。创建一个RDD有两个方法：在你的驱动程序中并行化一个已经存在的集合；从外部存储系统中引用一个数据集，这个存储系统可以是一个共享文件系统，比如HDFS、HBase或任意提供了Hadoop输入格式的数据来源。

并行化集合

并行化集合是通过在驱动程序中一个现有的迭代器或集合上调用SparkContext的parallelize方法建立的。为了创建一个能够并行操作的分布数据集，集合中的元素都会被拷贝。比如，以下语句创建了一个包含1到5的并行化

集合：

```
data = [1, 2, 3, 4, 5]
distData = sc.parallelize(data)
```
分布数据集（distData）被建立起来之后，就可以进行并行操作了。比如，我们可以调用disData.reduce(lambda a, b: a+b)来对元素进行叠加。在后文中我们会描述分布数据集上支持的操作。
并行集合的一个重要参数是将数据集划分成分片的数量。对每一个分片，Spark会在集群中运行一个对应的任务。典型情况下，集群中的每一个CPU将对应运行2-4个分片。一般情况下，Spark会根据当前集群的情况自行设定分片数量。但是，你也可以通过将第二个参数传递给parallelize方法(比如sc.parallelize(data, 10))来手动确定分片数量。

注意：有些代码中会使用切片（slice，分片的同义词）这个术语来保持向下兼容性。
###外部数据集
PySpark可以通过Hadoop支持的外部数据源（包括本地文件系统、HDFS、 Cassandra、HBase、亚马逊S3等等）建立分布数据集。Spark支持文本文件、序列文件以及其他任何Hadoop输入格式文件。

通过文本文件创建RDD要使用SparkContext的textFile方法。这个方法会使用一个文件的URI（或本地文件路径，hdfs://、s3n://这样的URI等等）然后读入这个文件建立一个文本行的集合。以下是一个例子：

```
>>> distFile = sc.textFile("data.txt")
```
建立完成后distFile上就可以调用数据集操作了。比如，我们可以调用map和reduce操作来叠加所有文本行的长度，代码如下：

```
distFile.map(lambda s: len(s)).reduce(lambda a, b: a + b)
```
在Spark中读入文件时有几点要注意：

如果使用了本地文件路径时，要保证在worker节点上这个文件也能够通过这个路径访问。这点可以通过将这个文
件拷贝到所有worker上或者使用网络挂载的共享文件系统来解决。

包括textFile在内的所有基于文件的Spark读入方法，都支持将文件夹、压缩文件、包含通配符的路径作为参数。比如，以下代码都是合法的：

```
textFile("/my/directory")
textFile("/my/directory/*.txt")
textFile("/my/directory/*.gz")
```
textFile方法也可以传入第二个可选参数来控制文件的分片数量。默认情况下，Spark会为文件的每一个块（在HDFS中块的大小默认是64MB）创建一个分片。但是你也可以通过传入一个更大的值来要求Spark建立更多的分片。注意，分片的数量绝不能小于文件块的数量。

除了文本文件之外，Spark的Python API还支持多种其他数据格式：

SparkContext.wholeTextFiles能够读入包含多个小文本文件的目录，然后为每一个文件返回一个（文件名，内容)对。这是与textFile方法为每一个文本行返回一条记录相对应的。

RDD.saveAsPickleFile和SparkContext.pickleFile支持将RDD以串行化的Python对象格式存储起来。串行化的过程中会以默认10个一批的数量批量处理。

序列文件和其他Hadoop输入输出格式。

注意

这个特性目前仍处于试验阶段，被标记为Experimental，目前只适用于高级用户。这个特性在未来可能会被基于Spark SQL的读写支持所取代，因为Spark SQL是更好的方式。

可写类型支持

PySpark序列文件支持利用Java作为中介载入一个键值对RDD，将可写类型转化成Java的基本类型，然后使用Pyrolite将java结果对象串行化。当将一个键值对RDD储存到一个序列文件中时PySpark将会运行上述过程的相反过程。首先将Python对象反串行化成Java对象，然后转化成可写类型。以下可写类型会自动转换：

```
| 可写类型 | Python类型 |
| ———————- | ————- |
| Text | unicode str|
| IntWritable | int |
| FloatWritable | float |
| DoubleWritable | float |
| BooleanWritable | bool |
| BytesWritable | bytearray |
| NullWritable | None |
| MapWritable | dict |
```
数组是不能自动转换的。用户需要在读写时指定ArrayWritable的子类型.在读入的时候，默认的转换器会把自定义的ArrayWritable子类型转化成Java的Object[]，之后串行化成Python的元组。为了获得Python的array.array类型来使用主要类型的数组，用户需要自行指定转换器。

保存和读取序列文件
和文本文件类似，序列文件可以通过指定路径来保存与读取。键值类型都可以自行指定，但是对于标准可写类型可以不指定。

```
>>> rdd = sc.parallelize(range(1, 4)).map(lambda x: (x, "a" * x ))
>>> rdd.saveAsSequenceFile("path/to/file")
>>> sorted(sc.sequenceFile("path/to/file").collect())
[(1, u'a'), (2, u'aa'), (3, u'aaa')]
```
保存和读取其他Hadoop输入输出格式

PySpark同样支持写入和读出其他Hadoop输入输出格式，包括’新’和’旧’两种Hadoop MapReduce API。如果有必要，一个Hadoop配置可以以Python字典的形式传入。以下是一个例子，使用了Elasticsearch ESInputFormat：

```
$ SPARK_CLASSPATH=/path/to/elasticsearch-hadoop.jar ./bin/pyspark
>>> conf = {"es.resource" : "index/type"}   # assume Elasticsearch is running on localhost defaults
>>> rdd = sc.newAPIHadoopRDD("org.elasticsearch.hadoop.mr.EsInputFormat",\
    "org.apache.hadoop.io.NullWritable", "org.elasticsearch.hadoop.mr.LinkedMapWritable", conf=conf)
>>> rdd.first()         # the result is a MapWritable that is converted to a Python dict
(u'Elasticsearch ID',
 {u'field1': True,
  u'field2': u'Some Text',
  u'field3': 12345})
```
注意，如果这个读入格式仅仅依赖于一个Hadoop配置和/或输入路径，而且键值类型都可以根据前面的表格直接转换，那么刚才提到的这种方法非常合适。

如果你有一些自定义的序列化二进制数据（比如从Cassandra/HBase中读取数据），那么你需要首先在Scala/Java端将这些数据转化成可以被Pyrolite的串行化器处理的数据类型。一个转换器特质已经提供好了。简单地拓展这个特质同时在convert方法中实现你自己的转换代码即可。记住，要确保这个类以及访问你的输入格式所需的依赖都被打到了Spark作业包中，并且确保这个包已经包含到了PySpark的classpath中。

这里有一些通过自定义转换器来使用Cassandra/HBase输入输出格式的Python样例和转换器样例。

###向Spark传递函数
Spark的API严重依赖于向驱动程序传递函数作为参数。有三种推荐的方法来传递函数作为参数。
Lambda表达式,简单的函数可以直接写成一个lambda表达式（lambda表达式不支持多语句函数和无返回值的语句）。
对于代码很长的函数，在Spark的函数调用中在本地用def定义。
模块中的顶级函数。
比如，传递一个无法转化为lambda表达式长函数，可以像以下代码这样：

```
"MyScript.py"""
if __name__ == "__main__":
    def myFunc(s):
        words = s.split(" ")
        return len(words)

    sc = SparkContext(...)
    sc.textFile("file.txt").map(myFunc)
```
值得指出的是，也可以传递类实例中方法的引用（与单例对象相反），这种传递方法会将整个对象传递过去。比如，考虑以下代码：

```
class MyClass(object):
    def func(self, s):
        return s
    def doStuff(self, rdd):
        return rdd.map(self.func)
```
在这里，如果我们创建了一个新的MyClass对象，然后对它调用doStuff方法，map会用到这个对象中func方法的引用，所以整个对象都需要传递到集群中。
还有另一种相似的写法，访问外层对象的数据域会传递整个对象的引用：

```
class MyClass(object):
    def __init__(self):
        self.field = "Hello"
    def doStuff(self, rdd):
        return rdd.map(lambda s: self.field + x)
```
此类问题最简单的避免方法就是，使用一个本地变量缓存一份这个数据域的拷贝，直接访问这个数据域：

```
def doStuff(self, rdd):
    field = self.field
    return rdd.map(lambda s: field + x)
```
###使用键值对
虽然大部分Spark的RDD操作都支持所有种类的对象，但是有少部分特殊的操作只能作用于键值对类型的RDD。这类操作中最常见的就是分布的shuffle操作，比如将元素通过键来分组或聚集计算。

在Python中，这类操作一般都会使用Python内建的元组类型，比如(1, 2)。它们会先简单地创建类似这样的元组，然后调用你想要的操作。

比如，一下代码对键值对调用了reduceByKey操作,来统计每一文本行在文本文件中出现的次数：

```
lines = sc.textFile("data.txt")
pairs = lines.map(lambda s: (s, 1))
counts = pairs.reduceByKey(lambda a, b: a + b)
```
我们还可以使用counts.sortByKey()，比如，当我们想将这些键值对按照字母表顺序排序，然后调用counts.collect()方法来将结果以对象列表的形式返回。

###转化操作
下面的表格列出了Spark支持的常用转化操作。欲知细节，请查阅RDD API文档（Scala, Java, Python）和键值对RDD函数文档（Scala, Java）。

```
转化操作 | 作用
————| ——
map(func) | 返回一个新的分布数据集，由原数据集元素经func处理后的结果组成
filter(func) | 返回一个新的数据集，由传给func返回True的原数据集元素组成
flatMap(func) | 与map类似，但是每个传入元素可能有0或多个返回值，func可以返回一个序列而不是一个值
mapParitions(func) | 类似map，但是RDD的每个分片都会分开独立运行，所以func的参数和返回值必须都是迭代器
mapParitionsWithIndex(func) | 类似mapParitions，但是func有两个参数，第一个是分片的序号，第二个是迭代器。返回值还是迭代器
sample(withReplacement, fraction, seed) | 使用提供的随机数种子取样，然后替换或不替换
union(otherDataset) | 返回新的数据集，包括原数据集和参数数据集的所有元素
intersection(otherDataset) | 返回新数据集，是两个集的交集
distinct([numTasks]) | 返回新的集，包括原集中的不重复元素
groupByKey([numTasks]) | 当用于键值对RDD时返回(键，值迭代器)对的数据集
aggregateByKey(zeroValue)(seqOp, combOp, [numTasks]) | 用于键值对RDD时返回（K，U）对集，对每一个Key的value进行聚集计算
sortByKey([ascending], [numTasks])用于键值对RDD时会返回RDD按键的顺序排序，升降序由第一个参数决定
join(otherDataset, [numTasks]) | 用于键值对(K, V)和(K, W)RDD时返回(K, (V, W))对RDD
cogroup(otherDataset, [numTasks]) | 用于两个键值对RDD时返回
(K, (V迭代器， W迭代器))RDD
cartesian(otherDataset) | 用于T和U类型RDD时返回(T, U)对类型键值对RDD
pipe(command, [envVars]) | 通过shell命令管道处理每个RDD分片
coalesce(numPartitions) | 把RDD的分片数量降低到参数大小
repartition(numPartitions) | 重新打乱RDD中元素顺序并重新分片，数量由参数决定
repartitionAndSortWithinPartitions(partitioner) | 按照参数给定的分片器重新分片，同时每个分片内部按照键排序
```
###启动操作
下面的表格列出了Spark支持的部分常用启动操作。欲知细节，请查阅RDD API文档（Scala, Java, Python）和键值对RDD函数文档（Scala, Java）。

```
启动操作 | 作用
————| ——
reduce(func) | 使用func进行聚集计算,func的参数是两个，返回值一个，两次func运行应当是完全解耦的，这样才能正确地并行运算
collect() | 向驱动程序返回数据集的元素组成的数组
count() | 返回数据集元素的数量
first() | 返回数据集的第一个元素
take(n) | 返回前n个元素组成的数组
takeSample(withReplacement, num, [seed]) | 返回一个由原数据集中任意num个元素的suzuki，并且替换之
takeOrder(n, [ordering]) | 返回排序后的前n个元素
saveAsTextFile(path) | 将数据集的元素写成文本文件
saveAsSequenceFile(path) | 将数据集的元素写成序列文件，这个API只能用于Java和Scala程序
saveAsObjectFile(path) | 将数据集的元素使用Java的序列化特性写到文件中，这个API只能用于Java和Scala程序
countByCount() | 只能用于键值对RDD，返回一个(K, int) hashmap，返回每个key的出现次数
foreach(func) | 对数据集的每个元素执行func, 通常用于完成一些带有副作用的函数，比如更新累加器（见下文）或与外部存储交互等
```
###RDD持久化
Spark的一个重要功能就是在将数据集持久化（或缓存）到内存中以便在多个操作中重复使用。当我们持久化一个RDD是，每一个节点将这个RDD的每一个分片计算并保存到内存中以便在下次对这个数据集（或者这个数据集衍生的数据集）的计算中可以复用。这使得接下来的计算过程速度能够加快（经常能加快超过十倍的速度）。缓存是加快迭代算法和快速交互过程速度的关键工具。

你可以通过调用persist或cache方法来标记一个想要持久化的RDD。在第一次被计算产生之后，它就会始终停留在节点的内存中。Spark的缓存是具有容错性的——如果RDD的任意一个分片丢失了，Spark就会依照这个RDD产生的转化过程自动重算一遍。

另外，每一个持久化的RDD都有一个可变的存储级别，这个级别使得用户可以改变RDD持久化的储存位置。比如，你可以将数据集持久化到硬盘上，也可以将它以序列化的Java对象形式（节省空间）持久化到内存中，还可以将这个数据集在节点之间复制，或者使用Tachyon将它储存到堆外。这些存储级别都是通过向persist()传递一个StorageLevel对象（Scala, Java, Python）来设置的。存储级别的所有种类请见下表：

注意：在Python中，储存的对象永远是通过Pickle库序列化过的，所以设不设置序列化级别不会产生影响。

Spark还会在shuffle操作（比如reduceByKey）中自动储存中间数据，即使用户没有调用persist。这是为了防止在shuffle过程中某个节点出错而导致的全盘重算。不过如果用户打算复用某些结果RDD，我们仍然建议用户对结果RDD手动调用persist，而不是依赖自动持久化机制。

应该选择哪个存储级别？

Spark的存储级别是为了提供内存使用与CPU效率之间的不同取舍平衡程度。我们建议用户通过考虑以下流程来选择合适的存储级别：

如果你的RDD很适合默认的级别（MEMORY_ONLY）,那么久使用默认级别吧。这是CPU最高效运行的选择，能够让RDD上的操作以最快速度运行。

否则，试试MEMORY_ONLY_SER选项并且选择一个快的序列化库来使对象的空间利用率更高，同时尽量保证访问速度足够快。

不要往硬盘上持久化，除非重算数据集的过程代价确实很昂贵，或者这个过程过滤了巨量的数据。否则，重新计算分片有可能跟读硬盘速度一样快。

如果你希望快速的错误恢复（比如用Spark来处理web应用的请求），使用复制级别。所有的存储级别都提供了重算丢失数据的完整容错机制，但是复制一份副本能省去等待重算的时间。

在大内存或多应用的环境中，处于实验中的OFF_HEAP模式有诸多优点：

这个模式允许多个执行者共享Tachyon中的同一个内存池

这个模式显著降低了垃圾回收的花销。

在某一个执行者个体崩溃之后缓存的数据不会丢失。

删除数据

Spark会自动监视每个节点的缓存使用同时使用LRU算法丢弃旧数据分片。如果你想手动删除某个RDD而不是等待它被自动删除，调用RDD.unpersist()方法。

###共享变量
通常情况下，当一个函数传递给一个在远程集群节点上运行的Spark操作（比如map和reduce）时，Spark会对涉及到的变量的所有副本执行这个函数。这些变量会被复制到每个机器上，而且这个过程不会被反馈给驱动程序。

通常情况下，在任务之间读写共享变量是很低效的。但是，Spark仍然提供了有限的两种共享变量类型用于常见的使用场景：广播变量和累加器。

广播变量

广播变量允许程序员在每台机器上保持一个只读变量的缓存而不是将一个变量的拷贝传递给各个任务。它们可以被使用，比如，给每一个节点传递一份大输入数据集的拷贝是很低效的。Spark试图使用高效的广播算法来分布广播变量，以此来降低通信花销。

可以通过SparkContext.broadcast(v)来从变量v创建一个广播变量。这个广播变量是v的一个包装，同时它的值可以功过调用value方法来获得。以下的代码展示了这一点：

```
>>> broadcastVar = sc.broadcast([1, 2, 3])
<pyspark.broadcast.Broadcast object at 0x102789f10>

>>> broadcastVar.value
[1, 2, 3]
```
在广播变量被创建之后，在所有函数中都应当使用它来代替原来的变量v，这样就可以保证v在节点之间只被传递一次。另外，v变量在被广播之后不应该再被修改了，这样可以确保每一个节点上储存的广播变量的一致性（如果这个变量后来又被传输给一个新的节点）。

累加器

累加器是在一个相关过程中只能被”累加”的变量，对这个变量的操作可以有效地被并行化。它们可以被用于实现计数器（就像在MapReduce过程中）或求和运算。Spark原生支持对数字类型的累加器，程序员也可以为其他新的类型添加支持。累加器被以一个名字创建之后，会在Spark的UI中显示出来。这有助于了解计算的累进过程（注意：目前Python中不支持这个特性）。

可以通过SparkContext.accumulator(v)来从变量v创建一个累加器。在集群中运行的任务随后可以使用add方法或+=操作符（在Scala和Python中）来向这个累加器中累加值。但是，他们不能读取累加器中的值。只有驱动程序可以读取累加器中的值，通过累加器的value方法。

以下的代码展示了向一个累加器中累加数组元素的过程：

```
>>> accum = sc.accumulator(0)
Accumulator<id=0, value=0>

>>> sc.parallelize([1, 2, 3, 4]).foreach(lambda x: accum.add(x))
...
10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

scala> accum.value
```

这段代码利用了累加器对int类型的内建支持，程序员可以通过继承AccumulatorParam类来创建自己想要的类型支持。AccumulatorParam的接口提供了两个方法：zero'用于为你的数据类型提供零值；'addInPlace'用于计算两个值得和。比如，假设我们有一个Vector`类表示数学中的向量，我们可以这样写：

```
class VectorAccumulatorParam(AccumulatorParam):
    def zero(self, initialValue):
        return Vector.zeros(initialValue.size)

    def addInPlace(self, v1, v2):
        v1 += v2
        return v1

 # Then, create an Accumulator of this type:
vecAccum = sc.accumulator(Vector(...), VectorAccumulatorParam())
```

累加器的更新操作只会被运行一次，Spark提供了保证，每个任务中对累加器的更新操作都只会被运行一次。比如，重启一个任务不会再次更新累加器。在转化过程中，用户应该留意每个任务的更新操作在任务或作业重新运算时是否被执行了超过一次。
累加器不会该别Spark的惰性求值模型。如果累加器在对RDD的操作中被更新了，它们的值只会在启动操作中作为RDD计算过程中的一部分被更新。所以，在一个懒惰的转化操作中调用累加器的更新，并没法保证会被及时运行。下面的代码段展示了这一点：

```
accum = sc.accumulator(0)
data.map(lambda x => acc.add(x); f(x))
# Here, acc is still 0 because no actions have cause the `map` to be computed.
```

###在集群上部署
这个应用提交指南描述了一个应用被提交到集群上的过程。简而言之，只要你把你的应用打成了JAR包（Java/Scala应用）或.py文件的集合或.zip压缩包(Python应用)，bin/spark-submit脚本会将应用提交到任意支持的集群管理器上。
###单元测试
Spark对单元测试是友好的，可以与任何流行的单元测试框架相容。你只需要在测试中创建一个SparkContext，并如前文所述将master的URL设为local，执行你的程序，最后调用SparkContext.stop()来终止运行。请确保你在finally块或测试框架的tearDown方法中终止了上下文，因为Spark不支持两个上下文在一个程序中同时运行。

从1.0之前版本的Spark迁移

Spark1.0冻结了1.X系列Spark的核心API。现在版本中没有标注”experimental”或是”developer API”的API在未来的版本中仍会被支持。对Python用户来说唯一的变化就是组管理操作，比如groupByKey, cogroup, join, 它们的返回值都从（键，值列表）对变成了（键， 值迭代器）对。

你还可以阅读Spark Streaming, MLlib和GraphX的迁移指南。

还有什么要做的

你可以在Spark的网站上看到更多的Spark样例程序。另外，在examples目录下还有许多样例代码（Scala, Java, Python）。你可以通过将类名称传给Spark的bin/run-example 脚本来运行Java和Scala语言样例，举例说明：

```
./bin/run-example SparkPi
```
对于Python例子，使用spark-submit脚本代替：

```
./bin/spark-submit examples/src/main/python/pi.py
```

为了给你优化代码提供帮助，配置指南和调优指南提供了关于最佳实践的一些信息。确保你的数据储存在以高效的格式储存在内存中，这很重要。为了给你部署应用提供帮助，集群模式概览描述了许多内容，包括分布式操作和支持的集群管理器。

=====
##Spark RDD API

###RDD是什么？

RDD是Spark中的抽象数据结构类型，任何数据在Spark中都被表示为RDD。从编程的角度来看，RDD可以简单看成是一个数组。和普通数组的区别是，RDD中的数据是分区存储的，这样不同分区的数据就可以分布在不同的机器上，同时可以被并行处理。因此，Spark应用程序所做的无非是把需要处理的数据转换为RDD，然后对RDD进行一系列的变换和操作从而得到结果。本文为第一部分，将介绍Spark RDD中与Map和Reduce相关的API中。
###RDD操作
RDD支持两类操作：转化操作，用于从已有的数据集转化产生新的数据集；启动操作，用于在计算结束后向驱动程序返回结果。举个例子，map是一个转化操作，可以将数据集中每一个元素传给一个函数，同时将计算结果作为一个新的RDD返回。另一方面，reduce操作是一个启动操作，能够使用某些函数来聚集计算RDD中所有的元素，并且向驱动程序返回最终结果（同时还有一个并行的reduceByKey操作可以返回一个分布数据集）。
在Spark所有的转化操作都是惰性求值的，就是说它们并不会立刻真的计算出结果。相反，它们仅仅是记录下了转换操作的操作对象（比如：一个文件）。只有当一个启动操作被执行，要向驱动程序返回结果时，转化操作才会真的开始计算。这样的设计使得Spark运行更加高效——比如，我们会发觉由map操作产生的数据集将会在reduce操作中用到，之后仅仅是返回了reduce的最终的结果而不是map产生的庞大数据集。
在默认情况下，每一个由转化操作得到的RDD都会在每次执行启动操作时重新计算生成。但是，你也可以通过调用persist(或cache)方法来将RDD持久化到内存中，这样Spark就可以在下次使用这个数据集时快速获得。Spark同样提供了对将RDD持久化到硬盘上或在多个节点间复制的支持。
基本操作
为了演示RDD的基本操作，请看以下的简单程序:

```
lines = sc.textFile("data.txt")
lineLengths = lines.map(lambda s: len(s))
totalLength = lineLengths.reduce(lambda a, b: a + b)
```
第一行定义了一个由外部文件产生的基本RDD。这个数据集不是从内存中载入的也不是由其他操作产生的；lines仅仅是一个指向文件的指针。第二行将lineLengths定义为map操作的结果。再强调一次，由于惰性求值的缘故，lineLengths并不会被立即计算得到。最后，我们运行了reduce操作，这是一个启动操作。从这个操作开始，Spark将计算过程划分成许多任务并在多机上运行，每台机器运行自己部分的map操作和reduce操作，最终将自己部分的运算结果返回给驱动程序。
如果我们希望以后重复使用lineLengths，只需在reduce前加入下面这行代码：

```
lineLengths.persist()
```
这条代码将使得lineLengths在第一次计算生成之后保存在内存中。


###如何创建RDD？

RDD可以从普通数组创建出来，也可以从文件系统或者HDFS中的文件创建出来。

举例：从普通数组创建RDD，里面包含了1到9这9个数字，它们分别在3个分区中。

```
scala> val a = sc.parallelize(1 to 9, 3)
a: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[1] at parallelize at <console>:12
```
举例：读取文件README.md来创建RDD，文件中的每一行就是RDD中的一个元素

```
scala> val b = sc.textFile("README.md")
b: org.apache.spark.rdd.RDD[String] = MappedRDD[3] at textFile at <console>:12
```
虽然还有别的方式可以创建RDD，但在本文中我们主要使用上述两种方式来创建RDD以说明RDD的API。

###map

map是对RDD中的每个元素都执行一个指定的函数来产生一个新的RDD。任何原RDD中的元素在新RDD中都有且只有一个元素与之对应。

举例：把原RDD中每个元素都乘以2来产生一个新的RDD。

```
scala> val a = sc.parallelize(1 to 9, 3)
scala> val b = a.map(x => x*2)
scala> a.collect
res10: Array[Int] = Array(1, 2, 3, 4, 5, 6, 7, 8, 9)
scala> b.collect
res11: Array[Int] = Array(2, 4, 6, 8, 10, 12, 14, 16, 18)
```

###mapPartitions

mapPartitions是map的一个变种。map的输入函数是应用于RDD中每个元素，而mapPartitions的输入函数是应用于每个分区，也就是把每个分区中的内容作为整体来处理的。 
它的函数定义为：

```
def mapPartitions[U: ClassTag](f: Iterator[T] => Iterator[U], preservesPartitioning: Boolean = false): RDD[U]
```
f即为输入函数，它处理每个分区里面的内容。每个分区中的内容将以Iterator[T]传递给输入函数f，f的输出结果是Iterator[U]。最终的RDD由所有分区经过输入函数处理后的结果合并起来的。

举例：

```
scala> val a = sc.parallelize(1 to 9, 3)
scala> def myfunc[T](iter: Iterator[T]) : Iterator[(T, T)] = {
    var res = List[(T, T)]() 
    var pre = iter.next while (iter.hasNext) {
        val cur = iter.next; 
        res .::= (pre, cur) pre = cur;
    } 
    res.iterator
}
scala> a.mapPartitions(myfunc).collect
res0: Array[(Int, Int)] = Array((2,3), (1,2), (5,6), (4,5), (8,9), (7,8))
```
上述例子中的函数myfunc是把分区中一个元素和它的下一个元素组成一个Tuple。因为分区中最后一个元素没有下一个元素了，所以(3,4)和(6,7)不在结果中。 
mapPartitions还有些变种，比如mapPartitionsWithContext，它能把处理过程中的一些状态信息传递给用户指定的输入函数。还有mapPartitionsWithIndex，它能把分区的index传递给用户指定的输入函数。

###mapValues

mapValues顾名思义就是输入函数应用于RDD中Kev-Value的Value，原RDD中的Key保持不变，与新的Value一起组成新的RDD中的元素。因此，该函数只适用于元素为KV对的RDD。

举例：

```
scala> val a = sc.parallelize(List("dog", "tiger", "lion", "cat", "panther", " eagle"), 2)
scala> val b = a.map(x => (x.length, x))
scala> b.mapValues("x" + _ + "x").collect
res5: Array[(Int, String)] = Array((3,xdogx), (5,xtigerx), (4,xlionx),(3,xcatx), (7,xpantherx), (5,xeaglex))
```
###mapWith

mapWith是map的另外一个变种，map只需要一个输入函数，而mapWith有两个输入函数。它的定义如下：

def mapWith[A: ClassTag, U: ](constructA: Int => A, preservesPartitioning: Boolean = false)(f: (T, A) => U): RDD[U]
第一个函数constructA是把RDD的partition index（index从0开始）作为输入，输出为新类型A；
第二个函数f是把二元组(T, A)作为输入（其中T为原RDD中的元素，A为第一个函数的输出），输出类型为U。
举例：把partition index 乘以10，然后加上2作为新的RDD的元素。

```
val x = sc.parallelize(List(1,2,3,4,5,6,7,8,9,10), 3) 
x.mapWith(a => a * 10)((a, b) => (b + 2)).collect 
res4: Array[Int] = Array(2, 2, 2, 12, 12, 12, 22, 22, 22, 22)
flatMap
```
与map类似，区别是原RDD中的元素经map处理后只能生成一个元素，而原RDD中的元素经flatmap处理后可生成多个元素来构建新RDD。 
举例：对原RDD中的每个元素x产生y个元素（从1到y，y为元素x的值）

```
scala> val a = sc.parallelize(1 to 4, 2)
scala> val b = a.flatMap(x => 1 to x)
scala> b.collect
res12: Array[Int] = Array(1, 1, 2, 1, 2, 3, 1, 2, 3, 4)
```
###flatMapWith

flatMapWith与mapWith很类似，都是接收两个函数，一个函数把partitionIndex作为输入，输出是一个新类型A；另外一个函数是以二元组（T,A）作为输入，输出为一个序列，这些序列里面的元素组成了新的RDD。它的定义如下：

```
def flatMapWith[A: ClassTag, U: ClassTag](constructA: Int => A, preservesPartitioning: Boolean = false)(f: (T, A) => Seq[U]): RDD[U]
```
举例：

```
scala> val a = sc.parallelize(List(1,2,3,4,5,6,7,8,9), 3)
scala> a.flatMapWith(x => x, true)((x, y) => List(y, x)).collect
res58: Array[Int] = Array(0, 1, 0, 2, 0, 3, 1, 4, 1, 5, 1, 6, 2, 7, 2,
8, 2, 9)
```
###flatMapValues

flatMapValues类似于mapValues，不同的在于flatMapValues应用于元素为KV对的RDD中Value。每个一元素的Value被输入函数映射为一系列的值，然后这些值再与原RDD中的Key组成一系列新的KV对。

举例

```
scala> val a = sc.parallelize(List((1,2),(3,4),(3,6)))
scala> val b = a.flatMapValues(x=>x.to(5))
scala> b.collect
res3: Array[(Int, Int)] = Array((1,2), (1,3), (1,4), (1,5), (3,4), (3,5))
```
上述例子中原RDD中每个元素的值被转换为一个序列（从其当前值到5），比如第一个KV对(1,2), 其值2被转换为2，3，4，5。然后其再与原KV对中Key组成一系列新的KV对(1,2),(1,3),(1,4),(1,5)。

###reduce

reduce将RDD中元素两两传递给输入函数，同时产生一个新的值，新产生的值与RDD中下一个元素再被传递给输入函数直到最后只有一个值为止。

举例:上述例子对RDD中的元素求和。

```
scala> val c = sc.parallelize(1 to 10)
scala> c.reduce((x, y) => x + y)
res4: Int = 55
```


###reduceByKey

顾名思义，reduceByKey就是对元素为KV对的RDD中Key相同的元素的Value进行reduce，因此，Key相同的多个元素的值被reduce为一个值，然后与原RDD中的Key组成一个新的KV对。

举例:上述例子中，对Key相同的元素的值求和，因此Key为3的两个元素被转为了(3,10)。

```
scala> val a = sc.parallelize(List((1,2),(3,4),(3,6)))
scala> a.reduceByKey((x,y) => x + y).collect
res7: Array[(Int, Int)] = Array((1,2), (3,10))
```





