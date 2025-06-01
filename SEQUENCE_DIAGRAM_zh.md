# Logstash Forwarder Java - 序列图 (日志行处理流程)

此图说明了当新日志行写入受监控文件、被处理并成功发送到Logstash时的典型事件流程。

**参与者:**

*   `用户/系统` (User/System): 写入日志文件的外部实体。
*   `被监控文件` (MonitoredFile): 磁盘上的实际日志文件。
*   `FileAlterationObserver`: (Apache Commons IO 组件) 检测文件系统变更。
*   `FileWatcher`: 管理文件监控、状态并协调读取。
*   `FileState`: 保存特定文件的当前读取偏移量 (指针) 和其他元数据。
*   `FileReader`: 从文件读取数据，处理多行和过滤。
*   `事件缓冲/批次` (EventSpool/Batch): 发送前事件的临时存储。
*   `Forwarder`: 主应用程序类，协调发送和状态更新。
*   `LumberjackClient`: (`ProtocolAdapter`) 处理Lumberjack协议通信。
*   `Registrar`: 管理`FileState`指针 (sincedb) 的持久化。
*   `Logstash服务器` (LogstashServer): 远程Logstash实例。

**流程:**

```
用户/系统        被监控文件 FileAlterationObserver FileWatcher     FileState       FileReader      事件缓冲/批次 Forwarder     LumberjackClient Registrar Logstash服务器
    |                  |               |                 |             |               |                 |         |                |         |        |
    | --写入日志-----> |               |                 |             |               |                 |         |                |         |        |
    | (例如, "新的日志条目")           |                 |             |               |                 |         |                |         |        |
    |                  |               |                 |             |               |                 |         |                |         |        |
    |                  | --通知变更 (异步) -->         |             |               |                 |         |                |         |        |
    |                  |               |                 |             |               |                 |         |                |         |        |
    |                  |               |  onFileChange() --> |             |               |                 |         |                |         |        |
    |                  |               |                 |             |               |                 |         |                |         |        |
    |                  |               |                 | -- processModifications() --> |             |         |                |         |        |
    |                  |               |                 |   (使用 FileState.identity 识别被监控文件有新数据) |             |         |                |         |        |
    |                  |               |                 |             |               |                 |         |                |         |        |
    |                  |               |                 | -- 获取当前偏移量 --> | (偏移量: X) |                 |         |                |         |        |
    |                  |               |                 |             | <---------- |                 |         |                |         |        |
    |                  |               |                 |             |               |                 |         |                |         |        |
    |                  |               |                 | -- 请求读取(FileState) --> |             |         |                |         |        |
    |                  |               |                 |             |               | -- readLines() -> |         |                |         |        |
    |                  |               |                 |             |               |  (从被监控文件的偏移量X开始读取) |         |                |         |        |
    |                  |               |                 |             |               | <---------------  |         |                |         |        |
    |                  |               |                 |             |               |  (应用多行/过滤逻辑) |         |                |         |        |
    |                  |               |                 |             |               |                 |         |                |         |        |
    |                  |               |                 |             |               | -- createEvent() --> |      |                |         |        |
    |                  |               |                 |             |               |                 |         |                |         |        |
    |                  |               |                 |             |               | -- addEvent(事件) --> |         |                |         |        |
    |                  |               |                 |             |               |                 | <---------- |                |         |        |
    |                  |               |                 |             |               |                 |         |                |         |        |
    |                  |               |                 |             | <— 更新FileState的新偏移量(Y)(内存中) |         |                |         |        |
    |                  |               |                 |             | <------------------------------------------ |         |                |         |        |
    |                  |               |                 |             |               |                 |         |                |         |        |
    |                  |               |                 | (批次就绪: 缓冲满或刷新时间到) |           |         |                |         |        |
    |                  |               |                 | ----------------------------------------------------------------> | -- sendEvents(批次) --> |       |        |
    |                  |               |                 |             |               |                 |         |                | (窗口大小: N) |        |
    |                  |               |                 |             |               |                 |         |                | ---------------------> |        |
    |                  |               |                 |             |               |                 |         |                |                      |        |
    |                  |               |                 |             |               |                 |         |                | (压缩数据帧)       |        |
    |                  |               |                 |             |               |                 |         |                | ---------------------> |        |
    |                  |               |                 |             |               |                 |         |                |                      |        |
    |                  |               |                 |             |               |                 |         |                |                      | <-- ACK (序列号: N) -- |
    |                  |               |                 |             |               |                 |         | <------------------  |                      |        |
    |                  |               |                 |             |               |                 |         |                |                      |        |
    |                  |               |                 | (ACK 成功) |              |                 |         |                |                      |        |
    |                  |               |                 | ----------------------------------------------------------------> | -- eventsAcknowledged(批次) --> |       |        |
    |                  |               |                 |             |               |                 |         |                | (更新 FileState.pointer 为 Y) |        |
    |                  |               |                 |             |               |                 |         | <-------------------- |       |        |
    |                  |               |                 |             |               |                 |         |                |                      |        |
    |                  |               |                 |             |               |                 |         |                | -- persistState(FileState) --> |        |
    |                  |               |                 |             |               |                 |         |                |                      | <------- |        |
    |                  |               |                 |             |               |                 |         |                |                      |        |
```

**流程说明:**

*   **异步性:** `FileAlterationObserver` 独立运行并异步通知 `FileWatcher`。`Forwarder` 中的主处理循环随后可能同步处理这些通知。
*   **批处理:** 事件通常被缓冲。当缓冲池 (批次) 已满或达到空闲超时时，会调用 `sendEvents`。此细节在“批次就绪”下被略微抽象。
*   **错误处理:** 此图显示了正常情况下的流程。错误处理 (例如，Logstash不可用、ACK超时、磁盘错误) 将涉及额外的步骤，如重试、记录日志以及在重试失败时可能丢弃数据。
*   **`Forwarder` 角色:** `Forwarder` 类通常协调 `FileWatcher`/`FileReader` 的结果与 `LumberjackClient` 之间的交互，并通过 `Registrar` 触发状态持久化。如果 `Forwarder` 将这些职责大量委托出去，则其中一些调用可能是从 `FileWatcher` 或 `FileReader` 直接到 `LumberjackClient` 或 `Registrar`。为简单起见，`Forwarder` 被显示为发送和在ACK后触发状态更新的主要参与者。
*   **`FileState.identity`:** `FileWatcher` 使用它来唯一地识别文件，即使在重命名/轮转事件中也是如此，这对于将活动与正确的 `FileState` 及其偏移量关联起来至关重要。
*   **内存中与持久化状态:** `FileState.pointer` 在 `FileReader` 成功读取和处理后首先在内存中更新。只有在Logstash确认接收后，`Registrar` 才会将其持久化。

此文本表示应能很好地概述操作顺序。
