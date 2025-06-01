# Logstash Forwarder Java - Sequence Diagram (Log Line Processing)

This diagram illustrates the typical flow of events when a new log line is written to a monitored file, processed, and successfully sent to Logstash.

**Participants:**

*   `User/System`: External entity writing to the log file.
*   `MonitoredFile`: The actual log file on the disk.
*   `FileAlterationObserver`: (Apache Commons IO component) Detects file system changes.
*   `FileWatcher`: Manages file monitoring, state, and coordinates reading.
*   `FileState`: Holds the current read offset (pointer) and other metadata for a specific file.
*   `FileReader`: Reads data from files, handles multiline and filtering.
*   `EventSpool/Batch`: Temporary storage for events before sending.
*   `Forwarder`: Main application class, orchestrates sending and state updates.
*   `LumberjackClient`: (`ProtocolAdapter`) Handles Lumberjack protocol communication.
*   `Registrar`: Manages persistence of `FileState` pointers (sincedb).
*   `LogstashServer`: The remote Logstash instance.

**Flow:**

```
User/System         MonitoredFile FileAlterationObserver FileWatcher       FileState       FileReader      EventSpool/Batch Forwarder     LumberjackClient Registrar LogstashServer
    |                    |                 |                   |               |               |                   |           |                  |         |          |
    | -- Writes log -->  |                 |                   |               |               |                   |           |                  |         |          |
    |   (e.g., "new log entry")            |                   |               |               |                   |           |                  |         |          |
    |                    |                 |                   |               |               |                   |           |                  |         |          |
    |                    | -- Notifies change (async) -->    |               |               |                   |           |                  |         |          |
    |                    |                 |                   |               |               |                   |           |                  |         |          |
    |                    |                 |  onFileChange() --> |               |               |                   |           |                  |         |          |
    |                    |                 |                   |               |               |                   |           |                  |         |          |
    |                    |                 |                   | -- processModifications() --> |               |           |                  |         |          |
    |                    |                 |                   |   (Identifies MonitoredFile has new data using FileState.identity) |               |           |                  |         |          |
    |                    |                 |                   |               |               |                   |           |                  |         |          |
    |                    |                 |                   | -- Get current offset --> |   (offset: X) |                   |           |                  |         |          |
    |                    |                 |                   |               | <---------- |                   |           |                  |         |          |
    |                    |                 |                   |               |               |                   |           |                  |         |          |
    |                    |                 |                   | -- Request read(FileState) --> |               |           |                  |         |          |
    |                    |                 |                   |               |               | -- readLines() -> |           |                  |         |          |
    |                    |                 |                   |               |               |   (Reads from MonitoredFile from offset X) |           |                  |         |          |
    |                    |                 |                   |               |               | <---------------  |           |                  |         |          |
    |                    |                 |                   |               |               |   (Applies multiline/filter logic) |           |                  |         |          |
    |                    |                 |                   |               |               |                   |           |                  |         |          |
    |                    |                 |                   |               |               | -- createEvent() --> |        |                  |         |          |
    |                    |                 |                   |               |               |                   |           |                  |         |          |
    |                    |                 |                   |               |               | -- addEvent(event) --> |           |                  |         |          |
    |                    |                 |                   |               |               |                   | <---------- |                  |         |          |
    |                    |                 |                   |               |               |                   |           |                  |         |          |
    |                    |                 |                   |               | <— Update new offset (Y) for FileState (in memory) |           |                  |         |          |
    |                    |                 |                   |               | <-------------------------------------------------- |           |                  |         |          |
    |                    |                 |                   |               |                   |           |                  |         |          |
    |                    |                 |                   | (Batch ready: spool full or flush time) |       |           |                  |         |          |
    |                    |                 |                   | ----------------------------------------------------------------> | -- sendEvents(batch) --> |         |          |
    |                    |                 |                   |               |                   |           |                  |   (WindowSize: N)  |          |
    |                    |                 |                   |               |                   |           |                  | ---------------------> |          |
    |                    |                 |                   |               |                   |           |                  |                      |          |
    |                    |                 |                   |               |                   |           |                  |   (CompressedDataFrame) |          |
    |                    |                 |                   |               |                   |           |                  | ---------------------> |          |
    |                    |                 |                   |               |                   |           |                  |                      |          |
    |                    |                 |                   |               |                   |           |                  |                      | <-- ACK (Seq: N) -- |
    |                    |                 |                   |               |                   |           | <------------------  |                      |          |
    |                    |                 |                   |               |                   |           |                  |                      |          |
    |                    |                 |                   | (ACK Successful) |                |           |                  |                      |          |
    |                    |                 |                   | ----------------------------------------------------------------> | -- eventsAcknowledged(batch) --> |         |          |
    |                    |                 |                   |               |                   |           |                  |   (Updates FileState.pointer to Y) |          |
    |                    |                 |                   |               |                   |           |                  | <-------------------- |         |          |
    |                    |                 |                   |               |                   |           |                  |                      |          |
    |                    |                 |                   |               |                   |           |                  | -- persistState(FileState) --> |          |
    |                    |                 |                   |               |                   |           |                  |                      | <------- |          |
    |                    |                 |                   |               |                   |           |                  |                      |          |
```

**Notes on the Flow:**

*   **Asynchronicity:** `FileAlterationObserver` runs independently and notifies `FileWatcher` asynchronously. The main processing loop in `Forwarder` might then handle these notifications synchronously.
*   **Batching:** Events are typically spooled. The `sendEvents` call happens when the spool (batch) is full or an idle timeout is reached. This detail is abstracted slightly under "Batch ready".
*   **Error Handling:** This diagram shows the happy path. Error handling (e.g., Logstash unavailable, ACK timeout, disk errors) would involve additional steps like retries, logging, and potentially discarding data if retries fail.
*   **`Forwarder` Role:** The `Forwarder` class often orchestrates the interaction between `FileWatcher`/`FileReader` results and the `LumberjackClient`, and also triggers state persistence via `Registrar`. Some of these calls might be direct from `FileWatcher` or `FileReader` to `LumberjackClient` or `Registrar` if `Forwarder` delegates these responsibilities heavily. For simplicity, `Forwarder` is shown as the primary actor for sending and triggering state updates post-ACK.
*   **`FileState.identity`:** Used by `FileWatcher` to uniquely identify files even across rename/rotation events, crucial for associating activity with the correct `FileState` and its offset.
*   **In-memory vs. Persisted State:** The `FileState.pointer` is updated in memory first after successful read and processing by `FileReader`. It's only persisted by `Registrar` after Logstash acknowledges receipt.

This textual representation should provide a good overview of the sequence of operations.
