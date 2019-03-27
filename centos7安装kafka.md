## Kafka安装教程

### 1.官网下载kafka
[kafka下载地址](https://www.apache.org/dyn/closer.cgi?path=/kafka/2.2.0/kafka_2.12-2.2.0.tgz)

### 2.安装
1. 执行解压命令 > tar -xzf kafka_2.12-2.2.0.tgz
2. 执行移动命令 > mv kafka_2.12-2.2.0 /usr/kafka
3. 关闭防火墙  > systemctl status firewalld.service,查看防火墙状态 > systemctl status firewalld.service,开机禁用防火墙 > systemctl disable firewalld.service

### 3.配置
