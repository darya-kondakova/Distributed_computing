Если устанавливаете Hadoop на кластер из [первой лабы](../main/1lab/instruction.md), советую размонтировать папку `/home`.  

**на node'ах**
```
sudo umount /home
```

Также закомментируйте соответствующие строки в `/etc/fstab` **на node'ах** и в `/etc/exports` **на head**.  
Далее можете действовать по [видео](https://www.youtube.com/watch?v=zGP0Uqm0SAo) или по этой инструкции.

**на всех узлах**
```
sudo apt install openjdk-8-jdk
```

**на head**
```
sudo wget https://downloads.apache.org/hadoop/common/hadoop-3.3.0/hadoop-3.3.0.tar.gz
scp hadoop-3.3.0.tar.gz darya@node1:/home/darya
scp hadoop-3.3.0.tar.gz darya@node2:/home/darya
```

**на всех узлах**
```
tar -xzvf hadoop-3.3.0.tar.gz
sudo mv hadoop-3.3.0 /usr/local/hadoop
```

```
sudo vi /etc/environment
    PATH="...:/usr/local/hadoop/bin:/usr/local/hadoop/sbin"
    JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64" 
    # используйте команду `update-alternatives --display java`, чтобы узнать, где у вас лежит java
```

Создаём нового пользователя **на всех узлах**, если у вас не автоматизирована синхранизация пользователей.
```
sudo adduser hadoop
sudo usermod -aG hadoop hadoop
sudo chown hadoop:root -R /usr/local/hadoop
sudo chmod g+rwx -R /usr/local/hadoop
sudo adduser hadoop sudo

su - hadoop
```

```
ssh-keygen -t rsa
ssh-copy-id hadoop@head
ssh-copy-id hadoop@node1
ssh-copy-id hadoop@node2
```

```bash
sudo vi .bashrc
    export PDSH_RCMD_TYPE=ssh
```

## HDFS

**на head**
```
sudo vi /usr/local/hadoop/etc/hadoop/core-site.xml
    <configuration>
      <property>
        <name>fs.default.name</name>
        <value>hdfs://head:9000</value>
      </property>
    </configuration>
```

```
sudo vi /usr/local/hadoop/etc/hadoop/hdfs-site.xml
    <configuration>
      # Установить количество копий блоков резервного копирования.
      <property>
        <name>dfs.replication</name>
        <value>2</value>
      </property>
      # Установить каталог хранения Namenode
      <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/usr/local/hadoop/data/nameNode</value>
      </property>
      # Установить директорию хранения Datanode
      <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/usr/local/hadoop/data/dataNode</value>
      </property>
    </configuration>
```

```
sudo vi /usr/local/hadoop/etc/hadoop/workers
    node1
    node2
```

```
scp /usr/local/hadoop/etc/hadoop/* node1:/usr/local/hadoop/etc/hadoop/
scp /usr/local/hadoop/etc/hadoop/* node2:/usr/local/hadoop/etc/hadoop/
```

```
source /etc/environmnet
hdfs namenode -format
```

```
start-dfs.sh
```

Проверяем, всё ли работает.
```console
hadoop@head:~$ jps
4138 Jps
3771 NameNode
4014 SecondaryNameNode
```
```console
hadoop@node1:~$ jps
2031 Jps
1814 DataNode
```
```console
hadoop@node2:~$ jps
2031 Jps
1814 DataNode
```

В браузере открываем страницу http://head:9870.  
Если у вас ничего не отображается ни в разделе 'Overview', ни в 'Datanodes',  
то **на head** выполнить команду `stop-all.sh`, в файле `/etc/hosts` закомментировать строку `127.0.1.1   head`  
и опять запустить `start-dfs.sh`.

## Yarn

**на node'ах**
```
sudo vi /usr/local/hadoop/etc/hadoop/yarn-site.xml
    <configuration>
      <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>head</value>
      </property>
    </configuration>
```

**на head**
```bash
sudo vi .bashrc
    export HADOOP_HOME="/usr/local/hadoop"
    export HADOOP_COMMON_HOME=$HADOOP_HOME
    export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
    export HADOOP_HDFS_HOME=$HADOOP_HOME
    export HADOOP_MAPRED_HOME=$HADOOP_HOME
    export HADOOP_YARN_HOME=$HADOOP_HOME
```
```
start-yarn.sh
```
```
yarn node -list
```
В браузере открываем страницу http://head:8088/cluster.

## WordCount
```
mkdir -p Wordcount/Input
```
Добавляем файлы, в которых будут считаться слова в `Wordcount/Input`:
```
vi Wordcount/Input/input1.txt
    любой текст
vi Wordcount/Input/input2.txt
    любой текст
```
```
hadoop fs -mkdir /WordCount/Input
hadoop fs -put Lab/Input/* /WordCountTutorial/Input
```
```
hadoop jar /usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.0.jar wordcount /WordCount/Input /WordCount/Output
```
Выводим результат:
```
hadoop dfs -cat /WordCount/Output/*
hadoop dfs -cat /WordCount/Output/* | sort -n -k2 # сортируем вывод по возрастанию кол-ва слов
```

## Полезные ссылки

### Hadoop Single Node Cluster  
[How to Install Hadoop Cluster on Ubuntu 16.04](https://medium.com/@alibaba-cloud/how-to-install-hadoop-cluster-on-ubuntu-16-04-bd9f52e5447c)  

[How to run WordCount program using Hadoop on Ubuntu](https://www.youtube.com/watch?v=UD8ysT3DTIQ) # установка hadoop и запуск своей wordcount.java программы  
Есть [интсрукция](https://docs.google.com/document/d/1-BKY9iBpkm2dSbO7OKc33JBa4CZymOCiwl1EWaFqeBQ/edit).  

[Hadoop Single Node Setup](https://www.youtube.com/watch?v=98UCknD8_qA)  
Есть [интсрукция](https://github.com/atozknowledge/bigdata/wiki/Hadoop-Single-Node-Installation).  

### Hadoop Multi-Node Cluster
[How To Set Up a Hadoop 3.2.1 Multi-Node Cluster on Ubuntu 18.04 (2 Nodes)](https://medium.com/@jootorres_11979/how-to-set-up-a-hadoop-3-2-1-multi-node-cluster-on-ubuntu-18-04-2-nodes-567ca44a3b12)
