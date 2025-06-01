## 项目概述：Java实现的Logstash Forwarder

**主要目的：**

本项目是一个基于Java的日志转发器，设计用于将文件中的日志发送到Logstash实例。它是对原有基于Go语言的logstash-forwarder的一个替代品，在Java生态系统中提供了类似的功能。

**主要特性：**

*   **Java版本兼容性：** 根据`pom.xml`文件中的`maven-compiler-plugin`配置，该项目构建为与Java 7及可能更高版本兼容。
*   **轻量级：** 设计强调最小化占用，旨在成为一个轻量级的日志转发解决方案。
*   **协议：** 它利用Lumberjack协议与Logstash实例进行通信。这一点从`LumberjackClient.java`以及`README.md`中的提及可以看出。
*   **文件监控：** 应用程序监控指定文件的变更（新的日志条目）并将其转发。此功能由`FileWatcher.java`处理。
*   **状态持久化：** 它会跟踪每个文件中最后读取的位置，确保在重启或中断期间不会丢失日志条目。这一点由`README.md`中提到的`.logstash-forwarder`文件以及可能由`FileReader.java`管理的功能所暗示。
*   **配置：** 转发器使用JSON文件（例如，`logstash-forwarder.conf`）进行配置，该文件指定了要监控的文件、Logstash服务器详细信息（主机、端口、SSL证书）以及其他参数。
*   **安全性：** 支持SSL以实现与Logstash服务器的安全通信，可通过JSON配置文件进行配置。

**主要使用的技术和库：**

*   **Java：** 应用程序的核心编程语言。
*   **Maven：** 用作构建自动化和项目管理工具，如`pom.xml`中所定义。
*   **Apache Commons IO：** 用于文件相关的工具操作，正如其在`pom.xml`中的依赖关系所示。具体可能使用了`FileUtils`和`IOUtils`。
*   **Jackson：** 用于JSON处理，主要用于解析配置文件。`pom.xml`中存在对`jackson-core`、`jackson-databind`和`jackson-annotations`的依赖。
*   **Log4j：** 用于应用程序日志记录，如`log4j`依赖所示。
*   **JUnit：** 用于单元测试，如`pom.xml`中的`junit`依赖所示。

**起源：**

该项目是作为对原有`logstash-forwarder`（用Go语言编写）的一个Java替代品而创建的。这可能是出于希望拥有一个能够更容易地在以Java为中心的环境中集成或管理的日志转发解决方案的动机。
