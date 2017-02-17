---
title: Spring Boot 项目如何配置 log4j2
layout: post
---

Log4j 是 Apache 的一个开源项目，使用 log4j，我们可以定义日志的输出格式，日志级别，输出目的等。log4j2 是 log4j的升级版，主要特点是异步日志输出，性能较 log4j 有大幅度提升，特别适合高并发场景下。 log4j 使用目前很火的 disruptor 无锁并发队列，世道日志事件处理非常迅速。并且，增加了 nosql 的 appender，如：kafka, flume 等。

Spring Boot 升级 log4j2 需要移除原先的 log4j1.x的 maven 依赖。可以使用 mvn dependency:tree 找出所有的 log4j1.x 的依赖包，显示 exclusion:

    <dependency>
             <groupId>org.apache.zookeeper</groupId>
             <artifactId>zookeeper</artifactId>
             <version>3.4.6</version>
             <exclusions>
                 <exclusion>
                     <groupId>log4j</groupId>
                     <artifactId>log4j</artifactId>
                 </exclusion>
                 <exclusion>
                     <groupId>org.slf4j</groupId>
                     <artifactId>slf4j-log4j12</artifactId>
                 </exclusion>
             </exclusions>
        </dependency>

同时，加入 log4j2的依赖，可以直接使用 spring-boot 提供的 starter：

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-logging</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-log4j2</artifactId>
    </dependency> 

在升级过程中，由于exclusion某些包里的 lo4j1.x ，导致运行时报 Class Not Found，可以使用 lo4j-over-slf4j 代替：

    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>log4j-over-slf4j</artifactId>
      <version>1.7.13</version>
    </dependency>

lo4j2 支持xml, yaml, json, properties等配置方式，网上能找到的几乎都是xml配置。但是，xml配置太过繁琐，没有yaml方便。以下使用yaml配置。首先，需要引用 jackson-dataformat-yaml 包：

    <dependency>
        <groupId>com.fasterxml.jackson.dataformat</groupId>
        <artifactId>jackson-dataformat-yaml</artifactId>
        <version>2.6.0-rc4</version>
    </dependency>

log4j2.yaml 配置文件如下：

    Configuration:
      properties:
        property:
          - name: logPath
            value: /home/shared/log/
          - name: filename
            value: project-name.log
          - name: pattern
            value: "%d{yyyy-MM-dd HH:mm:ss} [%p] [%t] [%c] %m%n"
      status: "info"
      Appenders:
        RollingRandomAccessFile:
          - name: "FileAppender"
            fileName: "${logPath}${filename}"
            filePattern: "${logPath}${filename}-%d{yyyy-MM-dd}"
            PatternLayout:
              pattern: "${pattern}"
            Policies:
              TimeBasedTriggeringPolicy: {}
          - name: "AnalysisFileAppender"
            fileName: "${logPath}analysis-${filename}"
            filePattern: "${logPath}analysis-${filename}-%d{yyyy-MM-dd}"
            PatternLayout:
              pattern: "${pattern}"
            Policies:
              TimeBasedTriggeringPolicy: {}
          - name: "HTTPRequestFileAppender"
            fileName: "${logPath}http-request-${filename}"
            filePattern: "${logPath}http-request-${filename}-%d{yyyy-MM-dd}"
            PatternLayout:
              pattern: "${pattern}"
            Policies:
              TimeBasedTriggeringPolicy: {}
       Raven:
         name: "Sentry"
         dsn: "xxxxxxxxxxxxxx"
      Loggers:
        Logger:
          - name: RequestLogger
            level: info
            additivity: false
            AppenderRef:
              - ref: HTTPRequestFileAppender
          - name: AnalysisLogger
            level: info
            additivity: false
            AppenderRef:
              - ref: AnalysisFileAppender
        Root:
         - level: "info"
           AppenderRef:
             - ref: "FileAppender"
         - level: "error"
           AppenderRef:
             - ref: "Sentry"

log4j2 支持在 application.yaml 中设置 logging.config，以实现不同环境下使用不同的配置文件：

    ---
    # 测试服务环境
    spring:
        profiles: test
    logging.access.dir: /home/shared/log
    logging.config: classpath:log4j2.test.yaml
    ---
    # 线上服务环境
    spring:
        profiles: online
    logging.access.dir: /home/shared/log
    logging.config: classpath:log4j2.online.yaml

