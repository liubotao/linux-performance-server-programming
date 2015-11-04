### Kafka高级别消费者

#### 为什么要使用

在某些时候业务从Kafka读取数据不关心处理这些信息的偏移量。他只是想得到数据就可以，所以高级别的消费者满足了大部分从Kafka消费事件的需求   

第一件事你需要知道的是高级别的消费者存储最后从特定的分区读取的偏移量是存储在Zookeeper当中的。当进程开始消费时偏移量被存储在一个提供给Kafka的名称，这个名称是做为一个消费者组的简称 

这个消费者组的名称在Kafka集群里面是全局的，所以你必须非常消息处理这些老的消费逻辑，当你启动新代码的时候你必须先关闭旧的消费进程。当一个新的进程开始处理这个相同的消费者集群，Kafka将会添加这个线程到当前消费这个Topic的的线程池以便做负载均衡，在负载均衡的过程中Kafka会分配可用分区到这些可用线程,或者可能迁移一个分区到另外一个进程。如果你有混合了老的和新的业务逻辑，可能一些消息会到旧的业务中.  

#### 设计高级别的消费者 

当你使用高级别的消费者你必须知道的第一件事消费者必须是多线程的应用,这个线程模型是围绕这个分区数量的在你的topic当中，下面是很详细的几条规则:  

*  如果你提*   供了比你分区多的线程，某些线程就永远不会接受到消息
*  如果你提供的分区比你的线程多，那么某些线程就会从多个分区获取数据
*  如果每个线程对应多个分区，那么你接受信息是不可靠的,甚至这个分区的偏移量会是序列化的,比如，你可能接受5条消息从分区10然后6条信息从分区11,然后5条信息从分区10接下来更多的5条从分区10虽然分区11是有数据可用的
*  添加更多的进程线程和进程会导致Kafka重新分配负载，可能将一个分区的的端到一个线程 

接下来，你的业务会从Kafka接收到一个迭代器,当没有消息的时候他是堵塞的 
下面是一个非常简单消费者 

    public class ConsumerTest implements Runnable {
	
		private KafkaStream m_stream;
		private int m_threadNumber;
	
		public ConsumerTest(KafkaStream a_stream, int a_threadNumber) {
			m_threadNumber = a_threadNumber;
			m_stream = a_stream;
		}
	
		
		@Override
		public void run() {
			ConsumerItertor<byte[], byte[]> it = m_stream.iterator();
			while (it.hasNext()) {
				System.out.println("Thread" + m_threadNumber + ":" + new String(it.next().message()));
				System.out.println("Shutting down Thread:" + m_threadNumber);
			}
		}
	}

这段最精彩的部分就是while(it.hasNext())则个片段.如果你不停止他这段代码会一直从Kafka读取数据.


#### 配置测试应用 

不像简单消费者高级消费者更关心的是很多薄记和错误处理,以及你需要告诉Kafka你把这些信息存储到那里,接下来的这些方法定义了基本的配置当你创建高级消费者的时候 

    private static ConsumerConfig crateConsumerConfig(String a_zookeeper, String a_grouId) {
        Properties properties = new Properties();
        properties.put("zookeeper.connect", a_zookeeper);
        properties.put("group.id", a_grouId);
        properties.put("zookeeper.session.timeout.ms", "400");
        properties.put("zookeeper.sync.time.ms", "200");
        properties.put("auto.commit.interval.ms", "1000");
        return new ConsumerConfig(properties);
    }
    
 "zookeeper.connect"定义了你连接Zookpeeper的集群获取一个实例的地方.Kafak使用Zookpeeper在存消息偏移量当他被一个特定的topic和一个分区组    
 
 "group.id"定义了消费那个那个分组    
 
 "zookeeper.session.timout.ms"表示Kafka发送一个请求到Zoookeeper到它结束的时间   
 
 "zookeeper.sync.time.ms"表示当发生一个错误时候Zookeeper能够允许的落后时间  
 
 "auto.commit.interval.ms"设置是多次时间更新消费者的offsets到Zookeeper
 
 #### 创建线程池
 
 下面这个demo使用Java的concurrent包来创建一个简单的线程池 
 
	public void run(int a_numThreads) {
	
		Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
		topicCountMap.put(topic, new Integer(a_numThreads));
		Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCountMap);
		List<Kafkastream<byte[], byte[]>> streams = consumerMap.get(topic);
		
		executor = Executors.newFixedThreadPool(a_numThreads);
		int threadNumber = 0;
		for (final KafkaSteam stream : streams) {
			executor.submit(new ConsumerTest(stream, threadNumber));
			threadNumber++;
	 	}
    }
    
首先我们创建一个Map告诉Kafka好点线程我们为当前的topics来提供,消费者创建消息流是怎么样传递消息给kafka,返回是topic(注意档期我们仅仅询问Kafka作为一个单独的topic但是我们已经请求多个不同的元素)  

最后我们创建一个线程池然后传递一个新的消费者测试对象到每一个对象作为我们的业务逻辑 


#### 优雅关闭和上传错误

Kakfa不支持每次读取数据都去更新Zookper的偏移量,相反他有一个很短的刷下时间,所以就会造成消费者消费了一个消息但是没有同步给zookeeper

	try {
		Thread.sleep(10000);
	} catch (InterruptedExcpetion ie) {
	
	}
	
	example.shutdown();


