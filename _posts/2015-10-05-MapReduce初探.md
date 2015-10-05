---
layout: post
title: "MapReduce初探"
date: 2015-10-05
categories: hadoop
excerpt: MapReduce工作流程
---

* content
{:toc}

## 题记

> 今天正好闲下来了，开始写我工作后的第一篇博客啦

前一段时间一直忙于找工作，工作的事解决了之后开始熟悉公司的项目代码以及所负责业务，所以在这一段时间内，Git博客并没有规律性的更新(在公司的这段时间，每天的总结都会记录到公司的wiki上，所以也就没有多余的时间来写博客了)

> 在公司的这几天接触到的新技术并不多，但是以前没有真正的实战机会去使用，所以之前对于比如像(Maven，Git，Dubbo，Hadoop，项目的基础构建等)只是单纯的了解，并没有很深的去理解和实战

比如Maven，之前一直抱着[Maven实战](http://book.douban.com/subject/5345682/)这本书来学(这本书很不错)，但是到公司之后，从仓库中clone一个项目(项目一般都是maven构建)需要本地调试的话，在本地打包mvn clean package会报错，因为pom.xml中所引用的jar包以及parent都是公司的nexus仓库中的，所以需要找到公司的settings.xml在本地调试编译打包(当然这是一个很简单的问题，但是刚到公司像这样的问题挺困扰我的)，再比如一般项目分为dev，beta，prod所有的环境下的数据库配置，hadoop文件系统的配置等其他配置都不相同，不可能通过每一个环境都需要重新编译pom.xml来替换配置(Maven中通过profile文件构建不同环境下的部署包)

> 参考[利用Profile构建不同环境的部署包](http://www.cnblogs.com/yjmyzz/p/3941043.html)

所以总结一点：

> 实战，实战，实战，重要的事情说三遍!！！

## MapReduce

> 这里讲的是MapReduce V1.0(关于yarn之后再更新)

### 什么是MapReduce?
> * 理论上讲MapReduce就是一个并行计算框架(对比JDK中的fork/join框架)
> * 实际上讲就是现在有一个统计任务(统计若干个文件单词的个数)，MapReduce的处理过程就是分割成若干个子任务，计算统计出各个子任务的结果，然后混合统计各个子任务的结果，最后根据各个子任务的值得出最后的统计结果

### 图

<h4>MapReduce：客户端-->提交Job工作流程图</h4>
![submit](http://images.cnitblog.com/blog/306623/201306/23175400-d7ef91ad75ad48099a525c097eb48bb6.jpg)
![submit1](http://images.cnitblog.com/blog/306623/201306/23175340-b918c709af954125962ac14125ad4823.jpg)

### 实例分析

<h4>MapReduce：WordCount计算过程</h4>
![cacl](http://images.cnitblog.com/blog/306623/201306/23175200-0ccb72e4bd7b4e48a2eea8673f361741.x-png)

就图来看，我们要统计file*.txt文件下的单词数(file1.txt file2.txt.....)
传统的思想我们通过IO读取文件进行逻辑处理，串行操作(当然可以通过多线程的方式来进行处理，这里提一下并发与并行)

通过IO读取指定的文件进行分析：

> * 声明一个容器存放单词Word(HashMap<String, Integer>)
> * 读取文件file*.txt
> * 读取文件中的一行readLine()，进行分割得到单词Word开始统计加一
> * 遇到相同单词总数加一
> * 这样貌似是正常的操作，但是当文件一大(比如达到1GB)，操作会十分低效
> * 这并不是唯一的思路(在Java中可以有多种方式来优化这个问题)

现在引入MapReduce并行框架
通过MapReduce来统计文件单词数量是如何操作的呢？
先看代码再来分析(Hadoop的经典例子)：


    //客户端测试方法
    public static void main(String[] args) throws Exception {


	    //加载配置信息，Input文件,Output目录
        Configuration conf = new Configuration();
        String[] otherArgs = (new GenericOptionsParser(conf, args)).getRemainingArgs();
        if(otherArgs.length < 2) {
            System.err.println("Usage: wordcount <in> [<in>...] <out>");
            System.exit(2);
        }

	    //构建一个Job
        Job job = Job.getInstance(conf, "word count");
	    //设置Job配置属性：Mapper Reducer, OutputKeyClass, OutputValueClass
        job.setJarByClass(WordCount.class);
        job.setMapperClass(WordCount.TokenizerMapper.class);
        job.setCombinerClass(WordCount.IntSumReducer.class);
        job.setReducerClass(WordCount.IntSumReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);


	    //设置输入文件属性
        for(int i = 0; i < otherArgs.length - 1; ++i) {
            FileInputFormat.addInputPath(job, new Path(otherArgs[i]));
        }


	    //设置输出目录
        FileOutputFormat.setOutputPath(job, new Path(otherArgs[otherArgs.length - 1]));
        //执行Job
	    System.exit(job.waitForCompletion(true)?0:1);
    }


    //统计单词数量Reducer
    public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
        private IntWritable result = new IntWritable();

        public IntSumReducer() {
        }
	    //核心reduce方式完成业务逻辑
        public void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
            int sum = 0;

            IntWritable val;
            for(Iterator i$ = values.iterator(); i$.hasNext(); sum += val.get()) {
                val = (IntWritable)i$.next();
            }

            this.result.set(sum);
            context.write(key, this.result);
        }
    }

    //分割单词写入文件系统的Mapper(分割任务)
    public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {
        private static final IntWritable one = new IntWritable(1);
        private Text word = new Text();

        public TokenizerMapper() {
        }


	    //核心map方式(从输入文件中解析分割单词并初始化数量1)
        public void map(Object key, Text value, org.apache.hadoop.mapreduce.Mapper.Context context) throws IOException, InterruptedException {
            StringTokenizer itr = new StringTokenizer(value.toString());

            while(itr.hasMoreTokens()) {
                this.word.set(itr.nextToken());
                context.write(this.word, one);
            }

        }
    }

先看TokenizerMapper.class
代码比较简单：

> * 从接收file*.txt读取后的字符串(一行)比如“Hello World”
> * 开始分割字符串得到单词并初始化单词数量
> * "Hello" "World" context("Hello", 1)/"World", 1
> * context就相当于一个大管家(文件系统)

然后看IntSumReducer.class
接收单词key的 键值对：

> * "Hello" 1
> * "Hello" 1
> * "Hello" 1
> * (Text key, Iterable<IntWriterable> values, Context context) 

直接迭代统计，将结果写入大管家中
貌似只是这样并不能让我们清晰的了解单词统计的功能是怎样完成的？
我们继续看Mapper Reducer中的方法：

    protected void setup(Mapper.Context context) throws IOException, InterruptedException {
    }

    protected void map(KEYIN key, VALUEIN value, Mapper.Context context) throws IOException, InterruptedException {
        context.write(key, value);
    }

    protected void cleanup(Mapper.Context context) throws IOException, InterruptedException {
    }

    public void run(Mapper.Context context) throws IOException, InterruptedException {
        this.setup(context);

        try {
            while(context.nextKeyValue()) {
                this.map(context.getCurrentKey(), context.getCurrentValue(), context);
            }
        } finally {
            this.cleanup(context);
        }


    }
    
     protected void setup(Reducer.Context context) throws IOException, InterruptedException {
    }

    protected void reduce(KEYIN key, Iterable<VALUEIN> values, Reducer.Context context) throws IOException, InterruptedException {
        Iterator i$ = values.iterator();

        while(i$.hasNext()) {
            Object value = i$.next();
            context.write(key, value);
        }

    }

    protected void cleanup(Reducer.Context context) throws IOException, InterruptedException {
    }

    public void run(Reducer.Context context) throws IOException, InterruptedException {
        this.setup(context);

        try {
            while(context.nextKey()) {
                this.reduce(context.getCurrentKey(), context.getValues(), context);
                Iterator iter = context.getValues().iterator();
                if(iter instanceof ValueIterator) {
                    ((ValueIterator)iter).resetBackupStore();
                }
            }
        } finally {
            this.cleanup(context);
        }

    }

    public abstract class Context implements ReduceContext<KEYIN, VALUEIN, KEYOUT, VALUEOUT> {
        public Context() {
        }
    }
    
> Mapper Reducer都是通过run方法来进行调用，而这个是通过JobClient.runJob(jobconf)来提交submit()job任务之后来进行调度run()

注意里面的setup() cleanup()方法 相当于Servlet中init() destroy()方法

> * 当MapReduce调度的时候执行setup()且执行一次
> * 当MapReduce结束的时候执行cleanup()执行一次
> * 所以在setup()完成初始化数据的操作，cleanup()中释放资源操作

因为源码中执行流程如下：

      setup();

        while(true?)

        	mapper()/reducer()

      cleanup();
      
### 问题(转自[夏天的森林](http://www.cnblogs.com/sharpxiajun/p/3151395.html))

<h4>那现在有三个问题？</h4>

1.file*.txt文件是怎么读取并且Mapper接收的是一行一行字符串这个是怎么操作？

这个就是对输入文件进行分片操作：

输入分片（input split）：

> * 在进行map计算之前，mapreduce会根据输入文件计算输入分片（input split），每个输入分片（input split）针对一个map任务，输入分片（input split）存储的并非数据本身，而是一个分片长度和一个记录数据的位置的数组
> * 输入分片（input split）往往和hdfs的block（块）关系很密切，假如我们设定hdfs的块的大小是64mb，如果我们输入有三个文件，大小分别是3mb、65mb和127mb，那么mapreduce会把3mb文件分为一个输入分片（input split），65mb则是两个输入分片（input split）而127mb也是两个输入分片（input split），换句话说我们如果在map计算前做输入分片调整，例如合并小文件，那么就会有5个map任务将执行，而且每个map执行的数据大小不均，这个也是mapreduce优化计算的一个关键点

2.Mapper输出如"Hello" 1，Reducer为什么可以接收［"Hello" 1, "Hello" 1, .... ］呢？

shuffle阶段：

将map的输出作为reduce的输入的过程就是shuffle了，这个是mapreduce优化的重点地方

> * Shuffle一开始就是map阶段做输出操作，一般mapreduce计算的都是海量数据，map输出时候不可能把所有文件都放到内存操作，因此map写入磁盘的过程十分的复杂，更何况map输出时候要对结果进行排序，内存开销是很大的
> * map在做输出时候会在内存里开启一个环形内存缓冲区，这个缓冲区专门用来输出的，默认大小是100mb，并且在配置文件里为这个缓冲区设定了一个阀值，默认是0.80（这个大小和阀值都是可以在配置文件里进行配置的），同时map还会为输出操作启动一个守护线程，如果缓冲区的内存达到了阀值的80%时候，这个守护线程就会把内容写到磁盘上，这个过程叫spill
> * 另外的20%内存可以继续写入要写进磁盘的数据，写入磁盘和写入内存操作是互不干扰的，如果缓存区被撑满了，那么map就会阻塞写入内存的操作，让写入磁盘操作完成后再继续执行写入内存操作，前面我讲到写入磁盘前会有个排序操作，这个是在写入磁盘操作时候进行，不是在写入内存时候进行的
> * 如果我们定义了combiner函数，那么排序前还会执行combiner操作。每次spill操作也就是写入磁盘操作时候就会写一个溢出文件，也就是说在做map输出有几次spill就会产生多少个溢出文件，等map输出全部做完后，map会合并这些输出文件。这个过程里还会有一个Partitioner操作，对于这个操作很多人都很迷糊，其实Partitioner操作和map阶段的输入分片（Input split）很像，一个Partitioner对应一个reduce作业
> * 如果我们mapreduce操作只有一个reduce操作，那么Partitioner就只有一个，如果我们有多个reduce操作，那么Partitioner对应的就会有多个，Partitioner因此就是reduce的输入分片，这个程序员可以编程控制，主要是根据实际key和value的值，根据实际业务类型或者为了更好的reduce负载均衡要求进行，这是提高reduce效率的一个关键所在
> * 到了reduce阶段就是合并map输出文件了，Partitioner会找到对应的map输出文件，然后进行复制操作，复制操作时reduce会开启几个复制线程，这些线程默认个数是5个，程序员也可以在配置文件更改复制线程的个数，这个复制过程和map写入磁盘过程类似，也有阀值和内存大小，阀值一样可以在配置文件里配置，而内存大小是直接使用reduce的tasktracker的内存大小，复制时候reduce还会进行排序操作和合并文件操作，这些操作完了就会进行reduce计算了。

3.最后一个问题MapReduce发生了什么(如何运行的)？

> * 首先是客户端要编写好mapreduce程序，配置好mapreduce的作业也就是job，接下来就是提交job了
> * 提交job是提交到JobTracker上的，这个时候JobTracker就会构建这个job，具体就是分配一个新的job任务的ID值，接下来它会做检查操作，这个检查就是确定输出目录是否存在，如果存在那么job就不能正常运行下去，JobTracker会抛出错误给客户端，接下来还要检查输入目录是否存在，如果不存在同样抛出错误
> * 如果存在JobTracker会根据输入计算输入分片（Input Split），如果分片计算不出来也会抛出错误，至于输入分片我后面会做讲解的，这些都做好了JobTracker就会配置Job需要的资源了
> * 分配好资源后，JobTracker就会初始化作业，初始化主要做的是将Job放入一个内部的队列，让配置好的作业调度器能调度到这个作业，作业调度器会初始化这个job，初始化就是创建一个正在运行的job对象（封装任务和记录信息），以便JobTracker跟踪job的状态和进程。初始化完毕后
> * 作业调度器会获取输入分片信息（input split），每个分片创建一个map任务。接下来就是任务分配了，这个时候tasktracker会运行一个简单的循环机制定期发送心跳给jobtracker，心跳间隔是5秒，程序员可以配置这个时间，心跳就是jobtracker和tasktracker沟通的桥梁，通过心跳，jobtracker可以监控tasktracker是否存活，也可以获取tasktracker处理的状态和问题，同时tasktracker也可以通过心跳里的返回值获取jobtracker给它的操作指令
> * 任务分配好后就是执行任务了。在任务执行时候jobtracker可以通过心跳机制监控tasktracker的状态和进度，同时也能计算出整个job的状态和进度，而tasktracker也可以本地监控自己的状态和进度。当jobtracker获得了最后一个完成指定任务的tasktracker操作成功的通知时候，jobtracker会把整个job状态置为成功，然后当客户端查询job运行状态时候（注意：这个是异步操作），客户端会查到job完成的通知的。如果job中途失败，mapreduce也会有相应机制处理，一般而言如果不是程序员程序本身有bug，mapreduce错误处理机制都能保证提交的job能正常完成

MapReduced的一些问题: 

> * 1.在大数据量的情况下可以设置setCombinerClass(Reducer.class)来减少Reducer输入的数据太多
> * 2.MapReduce中的单点问题
> * 3.MapReduce Hbase
> * 4.如果使用eclipse调试MapReduce程序失败：可能是因为setJarByClass(Self.class) 这个方法会默认从classpath找对应的Class,这样肯定会报ClassNotFound异常

解决方案：[eclipse调试MapReduce失败](http://f.dataguru.cn/thread-403-1-1.html)

### MapReduce博文书籍推荐

请参考书籍：

> * [Hadoop权威指南](http://book.douban.com/subject/6523762/)
> * [Hadoop技术内幕V1](http://book.douban.com/subject/24375031/)
> * [Hadoop技术内幕V2 yarn](http://book.douban.com/subject/25774649/)

请参考博文：

> * [hadoop学习笔记：mapreduce框架详解(夏天的森林)](http://www.cnblogs.com/sharpxiajun/p/3151395.html)
> * [hadoop资料汇总](http://f.dataguru.cn/thread-403-1-1.html)
> * [hadoop中使用MapReduce编程实例转](http://eric-gcm.iteye.com/blog/1807468)
> * [MapReduce原理](http://blog.jobbole.com/84089/)

