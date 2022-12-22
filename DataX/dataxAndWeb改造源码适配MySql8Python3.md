# DataX/DataX-Web 编译源码安装

| 版本 | 修改人 | 修改时间       |
| ---- | ------ | -------------- |
| 1.0  | 付有杰 | 2022年12月22日 |

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
    - ![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=NGI1ZDZiNjVhYTAwOGQ1ZDc1YTE5NWY4MGE0ODBiMTVfNGRKaWN5aHdsV2laRWtvcTZHNkxlSWtXWGdlVlpWWEhfVG9rZW46Ym94Y25RUUo3MnpNQ3k2MjBiS2U1SkFESW1oXzE2NzE2OTQ3OTg6MTY3MTY5ODM5OF9WNA)

- 第二是修改mysqlreader模块和mysqlwriter模块的pom文件
  - ![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=YWQ1YWM2MTc4ZWExNGZjMTRkNWViZDBhNDFkMWYyZDJfWFdEYTJUTm04aGI0RGVoYXN5SUVURzVXempjb0VOV0JfVG9rZW46Ym94Y252UzlqbDZvMWNhOWJHV3lEdmxWdHZiXzE2NzE2OTQ3OTg6MTY3MTY5ODM5OF9WNA)
- 修改plugin-rdbms-util模块的DataBaseType的MySQL驱动className
  - 按需修改即可，我这里用的全文替换。
  - ![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTBhMDg2NWUzZjhhNGNiMDRkYzc1MTc5NDkzNGUyOGVfcXQ1U21NSFVqTkpWaGxMUnNpcjdLT1RoZlRnM2lqRWdfVG9rZW46Ym94Y24zOHd2WGlkdmtiNXc1bkdpQnJ0OUxmXzE2NzE2OTQ3OTg6MTY3MTY5ODM5OF9WNA)
- 修改JDBC连接
  - 将converttoNull修改为CONVERT_TO_NULL(如果使用的驱动在8.0.13以上，则可以忽略此处。)
  - 后缀追加&serverTimeZone=Asia/Shanghai(datax会给JDBC URL追加后缀，所以自己写数据库连接的时候属性尽量写全。)
  - > ==Connector/J now translates the legacy value of convertToNull for the connection property zeroDateTimeBehavior to CONVERT_TO_NULL. This allows applications or frameworks that use the legacy value (for example, NetBeans) to work with Connector/J 8.0. (Bug #28246270, Bug #91421)==

  - ![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=NjM1NmY5MjBmZjQyZmIyMjJlYTUzOWQxN2I0MzM1OGNfM3NaZm9mb2pUZ0tXWUgycnVoNGQ2REVHWVRKYm1tM05fVG9rZW46Ym94Y254VFVFSXE5YmVuSjV0TVVpenpCeEdmXzE2NzE2OTQ3OTg6MTY3MTY5ODM5OF9WNA)

### **编译打包**

```Shell
mvn -U clean package assembly:assembly -Dmaven.test.skip=true
执行命令在项目的target目录会生成 datax*.tar.gz的压缩包
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

![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=OGFkN2ZlNGE4MjMzMTdjYmRkY2VlMzJhNDlhMDBhMDVfN2JyNlZNcmhEQ2gzS2xHV2FXMjBXZjJiZFN0NmdzbmtfVG9rZW46Ym94Y251dkVoNGNSNFA2UFdIOERQZW5vUlplXzE2NzE2OTQ3OTg6MTY3MTY5ODM5OF9WNA)

**修改application.yml**

> datax-admin/src/main/resources/application.yml

修改MySQL的驱动

![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=MmJhOTI4NjlmMjQzZjMzNmVmZjI4ZTlkZTkyMWRkNzRfWWlEdlFYMkc0aGhOMTBSM0xqcDlUYWdlR3dRcXpUYjJfVG9rZW46Ym94Y253NkJzVkFDNzlCaXdCVlFqYml2RE5lXzE2NzE2OTQ3OTg6MTY3MTY5ODM5OF9WNA)

### 适配Python3

修改 buildDataXExecutorCmd.class

![img](https://xqlwxcekt8.feishu.cn/space/api/box/stream/download/asynccode/?code=M2VkYzVmYjNiM2QwY2YxODA5YTk1ZTY2MDkyZTljNmJfTTc0SWRzRWY2Q2pwYk1Zc1dTbkx6TGlDT1dHWHgyd21fVG9rZW46Ym94Y25QMUIxQVU4QUswZ3RLVUZKcHpZTHZlXzE2NzE2OTQ3OTg6MTY3MTY5ODM5OF9WNA)

**如果本地环境是Python3,那么DataX的运行脚本也要更换.**

**替换datax/bin下面的三个python文件，替换文件在doc/datax-web/datax-python3下**

### 打包

```Shell
mvn clean install 
执行成功后将会在工程的build目录下生成安装包
build/datax-web-{VERSION}.tar.gz
```

## 使用

略(新开文件写)