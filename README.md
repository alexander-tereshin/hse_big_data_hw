# Инструкция по развертыванию Hadoop кластера

## Содержание

1. [Настройка подключения к нодам](#настройка-подключения-к-нодам)
2. [Установка необходимых файлов](#установка-необходимых-файлов)
3. [Настройка переменных окружения для Hadoop и Java](#настройка-переменных-окружения-для-hadoop-и-java)
4. [Настройка конфигурационных файлов](#настройка-конфигурационных-файлов)
5. [Копирование конфигурационных файлов на Data Nodes](#копирование-конфигурационных-файлов-на-data-nodes)
6. [Форматирование NameNode и запуск сервисов Hadoop](#форматирование-namenode-и-запуск-сервисов-hadoop)
7. [Настройка Nginx для NameNode, YARN и History Server](#настройка-nginx-для-namenode-yarn-и-history-server)
8. [Доступ к компонентам Hadoop](#доступ-к-компонентам-hadoop)

## Настройка подключения к нодам

Итеративно переходим на каждую ноду (Jump Node, Name Node, Data Nodes) и:

Редактируем файл с хостами:

```bash
sudo vim /etc/hosts
```

Прописываем ip адреса и названия хостов, комментируя остальные строки:

```bash
# The following lines are desirable for IPv6 capable hosts
#::1     ip6-localhost ip6-loopback
#fe00::0 ip6-localnet
#ff00::0 ip6-mcastprefix
#ff02::1 ip6-allnodes
#ff02::2 ip6-allrouters
192.168.1.139   team-34-nn
192.168.1.140   team-34-dn-00
192.168.1.141   team-34-dn-01
```

Создаем пользователя hadoop без root прав

```bash
sudo adduser hadoop
```

### Настройка SSH ключей для безпарольного доступа к нодам

На каждой ноде за исключением jump node переключаемся на пользователя hadoop и генерируем ssh ключ:

```bash
su - hadoop
ssh-keygen -t ed25519
```

После генерации SSH ключа на каждой ноде, копируем публичные ключи и вставляем в любой текстовый файл ключи из Name ноды и всех Data нод

```bash
cat ~/.ssh/id_ed25519.pub
```

Переходим на Jump Node и редактируем файл с авторизионными ключами у пользователя Hadoop:

```bash
su - hadoop
vim ~/.ssh/authorized_keys
```

Вставим все публичные ключи от Name ноды и Data нод. Пример для Name Node и двух Data Nodes:

```bash
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINFza/YAhvN9qyYpmqsrvjlxZXvrCSyBtIlRTfyHMh4u hadoop@team-34-nn

ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIATXZ1WPg+85HY+XAQeP1OcczGNiR+KZkHqhVik22M9O hadoop@team-34-dn-00

ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPk4fsFT4qTM+2Km/SOk8yPAGOeV/K8HlGVHrQheUBSN hadoop@team-34-dn-01
```

Передача файла `authorized_keys` на все ноды через `scp`:

```bash
scp ~/.ssh/authorized_keys hadoop@team-34-nn:~/.ssh/
scp ~/.ssh/authorized_keys hadoop@team-34-dn-00:~/.ssh/
scp ~/.ssh/authorized_keys hadoop@team-34-dn-01:~/.ssh/
```

Теперь пробуем подключиться с каждой ноды на другие с помощью SSH для проверки соединения:

```bash
ssh team-34-nn
ssh team-34-dn-00
ssh team-34-dn-01
```

## Установка необходимых файлов

Переходим на Name ноду и скачиваем архив с Hadoop с официального сайта. Это готовый для использования архив, содержащий все необходимые файлы и библиотеки для запуска Hadoop, включая HDFS, YARN, и MapReduce.

```bash
ssh hadoop@team-34-nn
```

```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.0/hadoop-3.4.0.tar.gz
```

копируем на оставшиеся ноды:

```bash
scp hadoop-3.4.0.tar.gz hadoop@team-34-dn-00:/home/hadoop/
scp hadoop-3.4.0.tar.gz hadoop@team-34-dn-01:/home/hadoop/
```

**Далее итеративно для каждой ноды (Name и Data нод)**:

Разархивируем его:

```bash
tar -xvzf hadoop-3.4.0.tar.gz
```

Перемещаем распакованные файлы Hadoop в `/usr/local/hadoop`:

```bash
sudo mv hadoop-3.4.0 /usr/local/hadoop
```

Создаем директорию для хранения системных логов Hadoop. Логи помогут отслеживать состояние системы и возможные ошибки.
Для этого создаем отдельную директорию и изменяем права доступа для директории Hadoop для того, чтобы пользователь `hadoop` имел полный доступ к директории.

```bash
sudo mkdir /usr/local/hadoop/logs
sudo chown -R hadoop:hadoop /usr/local/hadoop
```

## Настройка переменных окружения для Hadoop и Java

Переходим снова на Name ноду

```bash
ssh hadoop@team-34-nn
```

Открываем файл `~/.bashrc` для редактирования

```bash
vim ~/.bashrc
```

Добавляем в конец файла следующие строки для настройки окружения Hadoop:

```bash
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
```

После редактирования файла применяем изменения:

```bash
source ~/.bashrc
```

Проверка пути к Java

```bash
$ which java

/usr/bin/java
```

Определяем фактический путь к Java

```bash
$readlink -f /usr/bin/java

/usr/lib/jvm/java-11-openjdk-amd64/bin/java
```

Откроем файл конфигурации Hadoop для указания пути к Java:

```bash
sudo vim $HADOOP_HOME/etc/hadoop/hadoop-env.sh
```

Добавим переменные окружения для Java и Hadoop Classpath в файле `hadoop-env.sh`

```bash
export JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64"
export HADOOP_CLASSPATH+=" $HADOOP_HOME/lib/*.jar"
```

Эти строки указывают на местоположение установленной Java и расширяют `HADOOP_CLASSPATH`, чтобы включить все `.jar` файлы из библиотеки Hadoop.

Убедимся, что все настроено правильно, запустив одну из команд Hadoop

```bash
hadoop version
```

Это должно отобразить версию Hadoop

## Настройка конфигурационных файлов

### Редактирование конфигурационного файла _core-site.xml_

Далее мы редактируем конфигурационный файл _core-site.xml_, чтобы указать URL для NameNode:

```bash
sudo vim $HADOOP_HOME/etc/hadoop/core-site.xml
```

Затем добавляем несколько строк в файл и сохраняем его:

```xml
<configuration>
    <property>
        <name>fs.default.name</name>
        <value>hdfs://team-34-nn:9000</value>
        <description>URI по умолчанию для файловой системы</description>
    </property>
</configuration>
```

Кроме того, создаем директорию для хранения метаданных нод и изменяем владельца на пользователя _hadoop_:

```bash
sudo mkdir -p /home/hadoop/hdfs/{namenode,datanode}
sudo chown -R hadoop:hadoop /home/hadoop/hdfs
```

Эта команда успешно создает директорию для хранения метаданных нод и изменяет владельца этой директории.

### Редактирование конфигурационного файла _hdfs-site.xml_

Далее мы также редактируем конфигурационный файл _hdfs-site.xml_, чтобы определить коэффициент репликации

```bash
sudo nano $HADOOP_HOME/etc/hadoop/hdfs-site.xml
```

Затем добавляем несколько строк в файл и сохраняем его:

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
</configuration>
```

### Редактирование конфигурационного файла _mapred-site.xml_

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_HOME/share/hadoop/mapreduce/*:$HADOOP_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
```

### Редактирование конфигурационного файла _yarn-site.xml_

```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME, HADOOP_COMMON_HOME, HADOOP_HDFS_HOME, HADOOP_CONF_DIR...</value>
    </property>
</configuration>
```

### Редактирование файла _workers_

Редактируем файл _workers_ для указания имен нод:

```bash
sudo vim $HADOOP_HOME/etc/hadoop/workers
```

Добавляем в файл следующие строки:

```bash
team-34-nn
team-34-dn-00
team-34-dn-01
```

Теперь все необходимые конфигурации выполнены

## Копирование конфигурационных файлов на Data Nodes

1. **Копируем файл `.bashrc`:**

   ```bash
   scp ~/.bashrc hadoop@team-34-dn-00:/home/hadoop/
   scp ~/.bashrc hadoop@team-34-dn-01:/home/hadoop/
   ```

2. **Копируем файл `hadoop-env.sh`:**

   ```bash
   scp $HADOOP_HOME/etc/hadoop/hadoop-env.sh hadoop@team-34-dn-00:$HADOOP_HOME/etc/hadoop/
   scp $HADOOP_HOME/etc/hadoop/hadoop-env.sh hadoop@team-34-dn-01:$HADOOP_HOME/etc/hadoop/
   ```

3. **Копируем файл `core-site.xml`:**

   ```bash
   scp $HADOOP_HOME/etc/hadoop/core-site.xml hadoop@team-34-dn-00:$HADOOP_HOME/etc/hadoop/
   scp $HADOOP_HOME/etc/hadoop/core-site.xml hadoop@team-34-dn-01:$HADOOP_HOME/etc/hadoop/
   ```

4. **Копируем файл `hdfs-site.xml`:**

   ```bash
   scp $HADOOP_HOME/etc/hadoop/hdfs-site.xml hadoop@team-34-dn-00:$HADOOP_HOME/etc/hadoop/
   scp $HADOOP_HOME/etc/hadoop/hdfs-site.xml hadoop@team-34-dn-01:$HADOOP_HOME/etc/hadoop/
   ```

5. **Копируем файл `workers`:**

   ```bash
   scp $HADOOP_HOME/etc/hadoop/workers hadoop@team-34-dn-00:$HADOOP_HOME/etc/hadoop/
   scp $HADOOP_HOME/etc/hadoop/workers hadoop@team-34-dn-01:$HADOOP_HOME/etc/hadoop/
   ```

6. **Копируем файл `mapred-site.xml`:**

   ```bash
   scp $HADOOP_HOME/etc/hadoop/mapred-site.xml hadoop@team-34-dn-00:$HADOOP_HOME/etc/hadoop/
   scp $HADOOP_HOME/etc/hadoop/mapred-site.xml hadoop@team-34-dn-01:$HADOOP_HOME/etc/hadoop/
   ```

7. **Копируем файл `yarn-site.xml`:**

   ```bash
   scp $HADOOP_HOME/etc/hadoop/yarn-site.xml hadoop@team-34-dn-00:$HADOOP_HOME/etc/hadoop/
   scp $HADOOP_HOME/etc/hadoop/yarn-site.xml hadoop@team-34-dn-01:$HADOOP_HOME/etc/hadoop/
   ```

Теперь все необходимые конфигурационные файлы скопированы на Data Nodes

## Форматирование NameNode и запуск сервисов Hadoop

1. **Форматирование NameNode:**

   ```bash
   /usr/local/hadoop/bin/hdfs namenode -format
   ```

   В этой команде мы форматируем NameNode, что означает, что мы создаем новый файловый каталог для хранения метаданных Hadoop Distributed File System (HDFS). Форматирование необходимо выполнить только один раз при первоначальной настройке кластера. Этот шаг подготавливает HDFS к использованию, создавая необходимые директории и файлы для хранения данных и информации о файлах.

2. **Запуск HDFS:**

   ```bash
   /usr/local/hadoop/sbin/start-dfs.sh
   ```

   Эта команда запускает демон HDFS, включая NameNode и DataNode. После выполнения этой команды HDFS начинает работать, и пользователи могут загружать и получать доступ к данным, хранящимся в распределенной файловой системе.

3. **Запуск YARN:**

   ```bash
   /usr/local/hadoop/sbin/start-yarn.sh
   ```

   Эта команда запускает систему управления ресурсами YARN (Yet Another Resource Negotiator). YARN управляет ресурсами в кластере и отвечает за распределение задач и управление выполнением приложений MapReduce. Запуск YARN позволяет нам запускать и управлять приложениями на Hadoop.

4. **Запуск History Server:**

   После запуска YARN запустим **Hadoop History Server**, который отвечает за предоставление доступа к истории завершённых MapReduce заданий через веб-интерфейс. Это позволит вам просматривать логи и статистику по завершённым заданиям.

   ```bash
   mapred --daemon start historyserver
   ```

5. **Проверка состояния процессов:**

   ```bash
   jps
   ```

   Эта команда выводит список всех Java процессов, работающих в системе. Она позволяет нам проверить, успешно ли запущены демоны Hadoop, такие как NameNode, DataNode, ResourceManager и NodeManager. Если все сервисы запущены правильно, мы увидим их в списке, что означает, что наш кластер готов к использованию.

Таким образом, мы подготовили и запустили необходимые компоненты кластера Hadoop, включая HDFS и YARN, и проверили их состояние. Теперь мы можем начинать работать с Hadoop и загружать данные в HDFS или запускать приложения MapReduce.

## Настройка Nginx для NameNode, YARN и History Server

Настроим Nginx на Jump Node для проксирования запросов к компонентам Hadoop, включая NameNode, YARN и History Server. Это позволит нам удобно управлять доступом к ним через один интерфейс.

Переходим на Jump Node

   ```bash
   ssh hadoop@<IP-адрес-Jump-Node>
   ```

### 1. Настройка Nginx для NameNode

1. **Создаем копию конфигурации для NameNode:**

   ```bash
   sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/nn
   ```

2. **Редактируем файл конфигурации для NameNode:**

   ```bash
   sudo vim /etc/nginx/sites-available/nn
   ```

3. **Вставляем следующую конфигурацию:**

   ```nginx
   server {
       listen 9870 default_server;
       proxy_pass http://team-34-nn:9807;
   }
   ```

4. **Создаем символическую ссылку для активации конфигурации:**

   ```bash
   sudo ln -s /etc/nginx/sites-available/nn /etc/nginx/sites-enabled/nn
   ```

#### 2. Настройка Nginx для YARN

1. **Скопируем конфигурацию для YARN:**

   ```bash
   sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/yarn
   ```

2. **Редактируем файл конфигурации для YARN:**

   ```bash
   sudo vim /etc/nginx/sites-available/yarn
   ```

3. **Вставляем следующую конфигурацию:**

   ```nginx
   server {
       listen 8088 default_server;
       proxy_pass http://team-34-nn:8088;
   }
   ```

4. **Создаем символическую ссылку для YARN:**

   ```bash
   sudo ln -s /etc/nginx/sites-available/yarn /etc/nginx/sites-enabled/yarn
   ```

#### 3. Настройка Nginx для History Server

1. **Создаем копию конфигурации для History Server:**

   ```bash
   sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/history
   ```

2. **Редактируем файл конфигурации для History Server:**

   ```bash
   sudo vim /etc/nginx/sites-available/history
   ```

3. **Вставляем следующую конфигурацию:**

   ```nginx
   server {
       listen 19888 default_server;
       proxy_pass http://team-34-nn:19888;
   }
   ```

4. **Создаем символическую ссылку для активации конфигурации:**

   ```bash
   sudo ln -s /etc/nginx/sites-available/history /etc/nginx/sites-enabled/history
   ```

#### 4. Перезапуск Nginx

Теперь, когда все конфигурации настроены, необходимо перезапустить Nginx, чтобы применить изменения:

```bash
sudo systemctl restart nginx
```

## Доступ к компонентам Hadoop

Теперь Nginx настроен на проксирование запросов как к NameNode, так и к YARN и History Server. Мы можем обращаться к следующим адресам:

- **NameNode:** `http://<IP-адрес-Jump-Node>:9870`
- **YARN:** `http://<IP-адрес-Jump-Node>:8088`
- **History Server:** `http://<IP-адрес-Jump-Node>:19888`
