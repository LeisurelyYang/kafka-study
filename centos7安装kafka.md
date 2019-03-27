## Kafka安装教程

### 1.官网下载kafka
[kafka下载地址](https://www.apache.org/dyn/closer.cgi?path=/kafka/2.2.0/kafka_2.12-2.2.0.tgz)

### 2.安装
1. 执行解压命令 > tar -xzf kafka_2.12-2.2.0.tgz
2. 执行移动命令 > mv kafka_2.12-2.2.0 /usr/kafka
3. 关闭防火墙  > systemctl status firewalld.service,查看防火墙状态 > systemctl status firewalld.service,开机禁用防火墙 > systemctl disable firewalld.service

### 3.配置
* 打开config/server.properties配置文件
* 把31行的注释去掉，listeners=PLAINTEXT://:9092
* 把36行的注释去掉，把advertised.listeners值改为PLAINTEXT://host_ip:9092（我的服务器ip是192.168.1.33）
 ![avatar](https://github.com/LeisurelyYang/kafka-study/blob/master/file/server-config.jpg)
 
### 4.启动
1. 启动ZooKeeper >bin/zookeeper-server-start.sh config/zookeeper.properties
2. 启动kafka服务 >bin/kafka-server-start.sh config/server.properties
3. 创建topic >bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test,查看创建的主题 >bin/kafka-topics.sh --list --bootstrap-server localhost:9092
4. 启动内置的生产消息端 >bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
5. 启动内置的消费消息端 >bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning

### 5.生成者代码
```
package main

import (
	"fmt"
	"github.com/Shopify/sarama"
	"log"
	"os"
	"os/signal"
	"sync"
	"time"
)

var Address = []string{"192.168.1.33:9092"}
func main() {
	//syncProducer(Address)
	asyncProducer1(Address)
}
//同步消息模式
func syncProducer(address []string)  {
	config := sarama.NewConfig()
	config.Producer.Return.Successes = true
	config.Producer.Timeout = 5 * time.Second
	p, err := sarama.NewSyncProducer(address, config)
	if err != nil {
		log.Printf("sarama.NewSyncProducer err, message=%s \n", err)
		return
	}
	defer p.Close()
	topic := "test"
	srcValue := "sync: this is a message. index=%d"
	for i:=0; i<10; i++ {
		value := fmt.Sprintf(srcValue, i)
		msg := &sarama.ProducerMessage{
			Topic:topic,
			Value:sarama.ByteEncoder(value),
		}
		part, offset, err := p.SendMessage(msg)
		if err != nil {
			log.Printf("send message(%s) err=%s \n", value, err)
		}else {
			fmt.Fprintf(os.Stdout, value + "发送成功，partition=%d, offset=%d \n", part, offset)
		}
		time.Sleep(2*time.Second)
	}
}

//异步消费者(Goroutines)：用不同的goroutine异步读取Successes和Errors channel
func asyncProducer1(address []string)  {
	config := sarama.NewConfig()
	config.Producer.Return.Successes = true
	//config.Producer.Partitioner = 默认为message的hash
	p, err := sarama.NewAsyncProducer(address, config)
	if err != nil {
		log.Printf("sarama.NewSyncProducer err, message=%s \n", err)
		return
	}

	//Trap SIGINT to trigger a graceful shutdown.
	signals := make(chan os.Signal, 1)
	signal.Notify(signals, os.Interrupt)

	var wg sync.WaitGroup
	var enqueued, successes, errors int
	wg.Add(2) //2 goroutine

	// 发送成功message计数
	go func() {
		defer wg.Done()
		for range p.Successes() {
			successes++
		}
	}()

	// 发送失败计数
	go func() {
		defer wg.Done()
		for err := range p.Errors() {
			log.Printf("%s 发送失败，err：%s\n", err.Msg, err.Err)
			errors++
		}
	}()

	// 循环发送信息
	asrcValue := "async-goroutine: this is a message. index=%d"
	var i int
Loop:
	for {
		i++
		value := fmt.Sprintf(asrcValue, i)
		msg := &sarama.ProducerMessage{
			Topic:"test",
			Value:sarama.ByteEncoder(value),
		}
		select {
		case p.Input() <- msg: // 发送消息
			enqueued++
			fmt.Fprintln(os.Stdout, value)
		case <-signals: // 中断信号
			p.AsyncClose()
			break Loop
		}
		time.Sleep(2 * time.Second)
	}
	wg.Wait()

	fmt.Fprintf(os.Stdout, "发送数=%d，发送成功数=%d，发送失败数=%d \n", enqueued, successes, errors)

}
```

### 6.消费者代码
```
package main

import (
	"fmt"
	"github.com/Shopify/sarama"
	"github.com/bsm/sarama-cluster"
	"log"
	"os"
	"os/signal"
	"sync"
)

var Address = []string{"192.168.1.33:9092"}
func main() {
	topic := []string{"test"}
	var wg = &sync.WaitGroup{}
	wg.Add(3)
	//广播式消费：消费者1
	go clusterConsumer(wg, Address, topic, "group-1")
	//广播式消费：消费者2
	go clusterConsumer(wg, Address, topic, "group-1")

	go clusterConsumer(wg, Address, topic, "group-2")

	wg.Wait()
}

// 支持brokers cluster的消费者
func clusterConsumer(wg *sync.WaitGroup,brokers, topics []string, groupId string)  {
	defer wg.Done()
	config := cluster.NewConfig()
	config.Consumer.Return.Errors = true
	config.Group.Return.Notifications = true
	config.Consumer.Offsets.Initial = sarama.OffsetNewest

	// init consumer
	consumer, err := cluster.NewConsumer(brokers, groupId, topics, config)
	if err != nil {
		log.Printf("%s: sarama.NewSyncProducer err, message=%s \n", groupId, err)
		return
	}
	defer consumer.Close()

	// trap SIGINT to trigger a shutdown
	signals := make(chan os.Signal, 1)
	signal.Notify(signals, os.Interrupt)

	// consume errors
	go func() {
		for err := range consumer.Errors() {
			log.Printf("%s:Error: %s\n", groupId, err.Error())
		}
	}()

	// consume notifications
	go func() {
		for ntf := range consumer.Notifications() {
			log.Printf("%s:Rebalanced: %+v \n", groupId, ntf)
		}
	}()

	// consume messages, watch signals
	var successes int
Loop:
	for {
		select {
		case msg, ok := <-consumer.Messages():
			if ok {
				fmt.Fprintf(os.Stdout, "%s:%s/%d/%d\t%s\t%s\n", groupId, msg.Topic, msg.Partition, msg.Offset, msg.Key, msg.Value)
				consumer.MarkOffset(msg, "")  // mark message as processed
				successes++
			}
		case <-signals:
			break Loop
		}
	}
	fmt.Fprintf(os.Stdout, "%s consume %d messages \n", groupId, successes)
}

```
