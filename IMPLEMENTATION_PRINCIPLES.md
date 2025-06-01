# Logstash Forwarder Java - Key Implementation Principles

This document outlines the core implementation principles of the `logstash-forwarder-java` application, based on an analysis of its source code.

## 1. Stateful Forwarding

The application ensures that log data is not lost and duplicates are not sent by remembering the last read position in each monitored file.

*   **Sincedb and `Registrar`:** A state file (commonly `.logstash-forwarder` or similar, managed by the `Registrar` class) stores the last successfully transmitted byte offset for each file being watched.
*   **`FileState` Objects:** Internally, `FileState` objects (or a similar structure) hold the current offset and potentially other metadata (like inode or file signature) for each file.
*   **Loading State:** On startup, the `Registrar` loads this sincedb file. When a known file is encountered, its offset is initialized from the stored state.
*   **Updating State:** After successfully sending a batch of log events to Logstash and receiving an acknowledgment, the `Registrar` updates the sincedb file with the new offsets for the acknowledged files. This typically happens in `Forwarder.publishCurrentState()`.

## 2. File Monitoring and Change Detection

The application actively monitors files for new data and handles common log file management scenarios like rotation.

*   **`FileWatcher` and `FileAlterationObserver`:** The `FileWatcher` class is central to monitoring. It utilizes Apache Commons IO's `FileAlterationObserver` and `FileAlterationMonitor` to receive notifications about file system events (creation, modification, deletion) for the configured paths. The observer typically runs in a separate background thread.
*   **Processing Modifications (`FileWatcher.processModifications`):**
    *   **New Files:** When a new file matching the configured patterns is created, it's added to the watch list, and its initial offset is typically set to 0 (or the end of the file if configured to only send new lines).
    *   **Modified Files:** When a file is modified (data appended), `FileWatcher` signals that new lines are available. `FileReader` then reads from the last known offset.
    *   **File Deletion/Rotation/Renaming/Truncation:** Handling these is more complex:
        *   **`FileSigner`:** A `FileSigner` class is used to generate a signature (hash) of the initial part of a file. This helps in detecting if a file has been truncated and replaced (same name, new content) or if a new file is actually a rotated/renamed version of a previously watched file.
        *   If a watched file disappears, the system might look for new files with the same signature to continue reading from the correct offset if it's a rename/rotation.
        *   If a file's size becomes smaller than the last known offset (truncation), it's typically treated as a new file, and reading restarts from offset 0. The old state for that inode might be cleared or archived.
        *   The `FileWatcher`'s `processModifications` method contains the logic to compare current file states with previous states (inode, size, signature) to make these decisions.

## 3. Log Processing

The forwarder can process raw log lines, combine multiline events, and apply filters before sending.

*   **`FileReader`:** This class is responsible for reading lines from a file, starting from a given offset. It interacts with the `FileState` to know where to begin reading.
*   **Multiline Handling (`Multiline` class):**
    *   If multiline configuration is provided (via `FilesSection.Multiline`), `FileReader` (or a class it uses) buffers lines that match a specified pattern (e.g., lines starting with whitespace) and appends them to the previous line that *doesn't* match the pattern.
    *   This allows stack traces or other multi-line log entries to be sent as a single event.
    *   It involves pattern matching (regex) and managing a temporary buffer for pending lines.
*   **Filtering (`Filter` class):**
    *   If filter configurations are provided (via `FilesSection.Filter`), events can be included or excluded based on pattern matching against the log content.
    *   This filtering likely happens in `FileReader` or `Event` class after a line/event is read but before it's added to the outgoing spool.

## 4. Lumberjack Protocol v1 Usage

Communication with the Logstash server is handled using the Lumberjack protocol (version 1, as v2 uses JSON).

*   **`LumberjackClient`:** This class encapsulates the protocol logic.
    *   **Window Size Frames:** Before sending data, the client sends a "window size" frame to the server, indicating how many events (lines) will be in the current batch.
    *   **Data Frames:** Each log entry is sent as part of a "data" frame. The frame typically includes key-value pairs. Essential keys are `line` (the log content) and `offset` (the byte offset of the line in the source file). Custom fields defined in the configuration (`FilesSection.fields`) are also added to each event.
    *   **Compression:** Data frames are compressed using zlib before sending to reduce network bandwidth. This is a standard part of the Lumberjack v1 protocol.
    *   **ACK Frames:** After successfully sending all data frames in the declared window, the client waits for an Acknowledgment (ACK) frame from the server. The ACK frame contains the sequence number of the last event received by the server.
    *   **Sequence Numbering:** Both window size frames and data frames are associated with sequence numbers. The ACK from the server refers to these sequence numbers, confirming receipt up to that point. If the ACK matches the expected sequence number (the window size sent), the batch is considered successfully delivered.

## 5. SSL/TLS for Secure Communication

Secure communication with Logstash is supported via SSL/TLS.

*   **Configuration:** The `NetworkSection` of the configuration file allows specifying an SSL CA (`ssl ca` path to a certificate).
*   **`SSLSocketFactory`:** The `LumberjackClient` uses this configuration to set up an `SSLSocketFactory`.
    *   A `KeyStore` is typically loaded with the provided CA certificate (or default Java truststore if no CA is specified).
    *   An `SSLContext` is initialized with this Keystore, and an `SSLSocketFactory` is obtained from it.
    *   This factory is then used to create `SSLSocket`s for encrypted communication with the Logstash server.

## 6. Configuration Management

The application's behavior is controlled by a JSON configuration file.

*   **`ConfigurationManager`:** This class (or similar, potentially within `Forwarder`) is responsible for loading the JSON configuration file specified via command-line arguments or a default path.
*   **Jackson Library:** The Jackson library is used to parse the JSON string into Java objects.
*   **Configuration Objects (`Configuration`, `NetworkSection`, `FilesSection`):**
    *   `Configuration`: The root object holding network and file configurations.
    *   `NetworkSection`: Contains settings for connecting to Logstash (host, port, SSL CA, timeout).
    *   `FilesSection`: An array defining which files to watch, any specific fields to add to events from those files, multiline settings, and filter settings.
*   **Key Configurable Aspects:**
    *   Logstash server address and port.
    *   Path to SSL CA certificate.
    *   Connection and read timeouts.
    *   List of file paths to monitor (can include wildcards).
    *   Custom fields to associate with logs from specific paths.
    *   Multiline log processing rules (pattern, what, previous/next).
    *   Filtering rules.
    *   Spool size (max events in a batch).
    *   Idle flush time (how long to wait before sending an incomplete batch).

## 7. Error Handling and Reconnection Logic

The application is designed to be resilient to network issues.

*   **`AdapterException`:** Custom exception (`com.indeed.util.varevent.client.AbstractRetryingClient.AdapterException` or similar specific to `LumberjackClient`) likely used to wrap I/O errors or protocol-specific errors encountered during communication with Logstash.
*   **Connection and Send Retries:**
    *   `LumberjackClient` itself might have internal retries for specific operations.
    *   The `Forwarder` class (`Forwarder.connectToServer` and the main publish loop) implements a higher-level retry mechanism. If a connection to Logstash drops or a send operation fails, it will attempt to reconnect.
    *   Reconnection attempts are typically made with an exponential backoff strategy (waiting for a short period, then a longer period, up to a maximum) to avoid overwhelming the server or network.
    *   Failures and retry attempts are logged.

## 8. Concurrency

The application employs some level of concurrency for efficient operation.

*   **File Watching:** As mentioned, Apache Commons IO's `FileAlterationObserver` typically runs in its own dedicated background thread, polling the file system for changes independently of the main processing logic.
*   **Main Processing Loop:** The core logic in `Forwarder.java` (reading from ready files, processing lines, batching, sending to Logstash, waiting for ACK, updating sincedb) appears to be primarily single-threaded for a given Logstash endpoint.
    *   It waits for `FileWatcher` to signal file changes.
    *   Network operations (send, wait for ACK) are blocking but have timeouts.
*   **Multiple Endpoints (Potential):** While not explicitly detailed in the provided files for a single `Forwarder` instance, a more advanced setup could potentially involve multiple `LumberjackClient` instances or `Forwarder` threads if configured to send to multiple Logstash destinations, but the current structure seems focused on one primary endpoint per `Forwarder` instance.

This structure allows the application to monitor files efficiently without blocking the main log processing and sending pipeline, and to handle network operations with resilience.
```
