DataX/DataX-Web 编译源码安装

| 版本 | 修改人 | 修改时间       |
| ---- | ------ | -------------- |
| 1.0  | 付有杰 | 2022年12月21日 |

## 解决问题

- 适配MySQL8
- 适配Python3

如果没有以上需求，可忽略。

## DataX适配MySQL8

### 下载源码

https://github.com/alibaba/DataX

- 下载源码本地编译使用公司镜像遇到了某些组件不能下载的问题。所以改了一点maven的配置文件。主要是修改了镜像源。

- ```XML
  <mirrors>
        <mirror>
          <id>alimaven</id>
          <mirrorOf>central</mirrorOf>
          <name>aliyun maven</name>
          <url>http://maven.aliyun.com/nexus/content/repositories/central/</url>
      </mirror>
  
      <!-- 中央仓库1 -->
      <mirror>
          <id>repo1</id>
          <mirrorOf>central</mirrorOf>
          <name>Human Readable Name for this Mirror.</name>
          <url>http://repo1.maven.org/maven2/</url>
      </mirror>
  
      <!-- 中央仓库2 -->
      <mirror>
          <id>repo2</id>
          <mirrorOf>central</mirrorOf>
          <name>Human Readable Name for this Mirror.</name>
          <url>http://repo2.maven.org/maven2/</url>
      </mirror>
      <!-- <mirror>
        <id>intsig-public</id>
        <mirrorOf>*</mirrorOf>
        <name>intsig</name>
        <url>http://maven.intsig.net/repository/intsig-public</url>
      </mirror> -->
    </mirrors>
  ```

- ###  改造DataX源码适配MySQL8

- > DataX目前的读写模块还是特定于MySQL5.7编码，我们生产版本是MySQL8，所以需要做一些调整。

-  **修改POM引用MySQL8的驱动**

- - 第一是**可以**修改DataX-all工程的POM文件
    - 修改驱动版本为**8.0.28**
    - ![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=Mjk1ZDEyZGFhOWVmMDM2YjRkMzI1ZDhiYzdjNGRmNjlfbzRBSVdWOGtkWTlaUklaSWVZZ1lLZ0owRWtYRVR0NHlfVG9rZW46Ym94Y25RUUo3MnpNQ3k2MjBiS2U1SkFESW1oXzE2NzE3NjI5OTM6MTY3MTc2NjU5M19WNA)

- 第二是修改mysqlreader模块和mysqlwriter模块的pom文件
  - ![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=MTkyMGEzYTJjYzRmMWJjM2UwMWJhNDVmNjAxOTZmZDJfY3ZTMVdMY0ZNTjZBeTdIRUNuSmp4c09LQkIwZVVheEJfVG9rZW46Ym94Y252UzlqbDZvMWNhOWJHV3lEdmxWdHZiXzE2NzE3NjI5OTM6MTY3MTc2NjU5M19WNA)
- 修改plugin-rdbms-util模块的DataBaseType的MySQL驱动className
  - 按需修改即可，我这里用的全文替换。
  - ![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=NDMxMzkwYWVmMGNlNmY1M2I2ZDcwMWI3ZWU4NjVhNWJfY21nQmJEeVljMVdONU1pb3dpM3BGalZBbHg0cEVPTkVfVG9rZW46Ym94Y24zOHd2WGlkdmtiNXc1bkdpQnJ0OUxmXzE2NzE3NjI5OTM6MTY3MTc2NjU5M19WNA)
- 修改JDBC连接
  - 将converttoNull修改为CONVERT_TO_NULL(如果使用的驱动在8.0.13以上，则可以忽略此处。)
  - 后缀追加&serverTimeZone=Asia/Shanghai(datax会给JDBC URL追加后缀，所以自己写数据库连接的时候属性尽量写全。)
  - > ==Connector/J now translates the legacy value of convertToNull for the connection property zeroDateTimeBehavior to CONVERT_TO_NULL. This allows applications or frameworks that use the legacy value (for example, NetBeans) to work with Connector/J 8.0. (Bug #28246270, Bug #91421)==

  - ![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=M2FhYzQ2YzlhNWI5YzUyMDhlMzQyMDcxYjQwZjYyMmJfODhzT2JxWm16ZVhOR2d3U3NZa1FkZEJuV29KTEVkM2hfVG9rZW46Ym94Y254VFVFSXE5YmVuSjV0TVVpenpCeEdmXzE2NzE3NjI5OTM6MTY3MTc2NjU5M19WNA)

### **编译打包安装**

```Shell
mvn -U clean package assembly:assembly -Dmaven.test.skip=true
执行命令在项目的target目录会生成 datax*.tar.gz的压缩包
使用 tar -zxvf 命令解压在某个目录即可
```

### **检查是否成功**

```Shell
python2 运行 python {YOUR_DATAX_HOME}/bin/datax.py {YOUR_DATAX_HOME}/job/job.json
python3 运行 python3 {YOUR_DATAX_HOME}/bin/datax.py {YOUR_DATAX_HOME}/job/job.json

显示类似的结果没有error就是成功了。
...
2015-12-17 11:20:25.263 [job-0] INFO  JobContainer - 
任务启动时刻                    : 2015-12-17 11:20:15
任务结束时刻                    : 2015-12-17 11:20:25
任务总计耗时                    :                 10s
任务平均流量                    :              205B/s
记录写入速度                    :              5rec/s
读出记录总数                    :                  50
读写失败总数                    :                   0
```

## DataX-WEB改造源码

**目标**

- 适配Python3
- 适配MySQL8

### 下载源码

https://github.com/WeiYe-Jing/datax-web

### 适配MySQL8

**修改datax-web pom文件**

修改驱动版本

![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDM4Zjk3NjgzZWY5YTFjMjliNmQzNWY2YjhkMGU0YjJfdVN4R0tBNXRUQTFaaE1vYVNrelh3Wk0yN1YzbnJMTHhfVG9rZW46Ym94Y251dkVoNGNSNFA2UFdIOERQZW5vUlplXzE2NzE3NjI5OTM6MTY3MTc2NjU5M19WNA)

**修改application.yml**

> datax-admin/src/main/resources/application.yml

修改MySQL的驱动

![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=ZWMwYTM1NjE2MmU2NGU0ZjVlMTcxZDMwOTA2MTNlYjNfNENySVBXWGdWNE9peVZrM1hFUVhkd1dmRzlidVJsYUZfVG9rZW46Ym94Y253NkJzVkFDNzlCaXdCVlFqYml2RE5lXzE2NzE3NjI5OTM6MTY3MTc2NjU5M19WNA)

### 适配Python3

修改 buildDataXExecutorCmd.class

![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=ZWI0NzM3ZDg4ZDg4ZTg4MDE1ZDkzOTU1N2U3MWM2YTBfNExzdGNFV2UxY2ZrVXdheG4zMlR4S0piR3lBMkdOWGlfVG9rZW46Ym94Y25QMUIxQVU4QUswZ3RLVUZKcHpZTHZlXzE2NzE3NjI5OTM6MTY3MTc2NjU5M19WNA)

**如果本地环境是Python3,那么DataX的运行脚本也要更换.**

**替换datax/bin下面的三个python文件，替换文件在doc/datax-web/datax-python3下**

### 打包安装

```Shell
mvn clean install 
执行成功后将会在工程的build目录下生成安装包
build/datax-web-{VERSION}.tar.gz
使用 tar -zxvf 解压在某个目录 
安装datax-web需要提前准备好一个MySQL数据库供datax-web.
进入解压后的目录 执行  ./bin/install.sh 一路点yes即可。遇到需要输入数据库密码的地方就输入准备好的MySQL数据库。输错了也不要慌后面可以改。
可在 ./modules/datax-admin/conf/bootstrap.properties 重新配置DataX-WEB的MySQL数据库地址。
可在 ./modules/datax-admin/bin/env.properties 配置邮件服务(可跳过，用于发送邮件告警)
在 ./modules/datax-execute/bin/env.properties 下配置DataX的Py脚本地址
PYTHON_PATH=【我这里是 /home/youjie_fu/datax/bin/datax.py】
```

## 启动DataX-WEB

#### **启动服务**

- **-一键启动所有服务**

```Plaintext
./bin/start-all.sh
```

中途可能发生部分模块启动失败或者卡住，可以退出重复执行，如果需要改变某一模块服务端口号，则：

```Plaintext
vi ./modules/{module_name}/bin/env.properties
```

找到SERVER_PORT配置项，改变它的值即可。 当然也可以单一地启动某一模块服务：

```Plaintext
./bin/start.sh -m {module_name}
```

- **一键取消所有服务**

```Plaintext
./bin/stop-all.sh
```

当然也可以单一地停止某一模块服务：

```Plaintext
./bin/stop.sh -m {module_name}
```

#### **查看服务（注意！注意！）**

在Linux环境下使用JPS命令，查看是否出现DataXAdminApplication和DataXExecutorApplication进程，如果存在这表示项目运行成功

#### **如果项目启动失败，请检查启动日志：modules/datax-admin/bin/console.out或者modules/datax-executor/bin/console.out**

Tips: 脚本使用的都是bash指令集，如若使用sh调用脚本，可能会有未知的错误

#### **运行**

部署完成后，在浏览器中输入 http://ip:port/index.html 就可以访问对应的主界面（ip为datax-admin部署所在服务器ip,port为为datax-admin 指定的运行端口）

输入用户名 admin 密码 123456 就可以直接访问系统

### 使用

略