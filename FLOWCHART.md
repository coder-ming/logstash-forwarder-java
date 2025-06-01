# Logstash Forwarder Java - High-Level Flowchart

```
[START]
  |
  V
Application Startup
  |
  V
Parse Command-Line Options (e.g., -config, -spool-size, -idle-flush-time)
  |
  +---------------------> [ Has -config option? ] --NO--> Use default config path ("logstash-forwarder.conf")
  |                             | YES
  |                             V
  |                       Use specified config path
  |                             |
  |<----------------------------+
  |
  V
Load Configuration File (JSON)
  |
  V
[ Configuration File Valid? ] --NO--> Log Error & Exit
  | YES
  V
Parse Configuration:
  - Network settings (host, port, ssl ca, timeout)
  - Files to watch (paths, fields)
  |
  V
Initialize FileWatcher
  |
  V
Load Sincedb State (e.g., ".logstash-forwarder")
  - For each configured file, get last known byte offset
  |
  V
Initialize LumberjackClient
  |
  V
Attempt to Connect to Logstash Server (using network settings from config)
  |
  +------> [ Connection Successful? ] --NO-------------------------------------+
  |               | YES                                                        |
  |               V                                                            |
  |       Log "Connected to Logstash"                                          |
  |                                                                            |
  |<-------------------------------------- Main Processing Loop <--------------+
  |                                         (Periodically, e.g., every second) |
  |                                                                            |
  |  [ Is stdin configured to be watched? ] --YES--> Check stdin for new lines |
  |    | NO                                           |                       |
  |    V                                              V                       |
  |  Check Monitored Files for Modifications (via FileWatcher)                 |
  |    |                                                                       |
  |    +-----> [ Any files modified or new lines from stdin? ] --NO--> Loop back to check again after delay
  |              | YES                                                       |
  |              V                                                           |
  |            For each modified file (or stdin):                            |
  |              - Read new log lines from last known offset                 |
  |              - Add lines to current event batch (spool)                  |
  |              - Update byte offset for the file                           |
  |                                                                          |
  |  [ Batch size > 0 AND (Batch full (spool size) OR Idle flush time reached)? ] --YES--> Send Batch
  |    | NO                                                                      |
  |    +-------------------------------------------------------------------------+
  |                                                                              |
  | Send Batch to Logstash (via LumberjackClient):                               |
  |   - Prepare Lumberjack payload (window size = batch size, compressed data)   |
  |   - Send data frame                                                          |
  |   |                                                                          |
  |   V                                                                          |
  | [ Data Sent Successfully? ] --NO-------------------------------------------> [ Handle Send Error ]
  |   | YES                                                                        |
  |   V                                                                            |
  | Wait for Acknowledgement (ACK) from Logstash                                 |
  |   |                                                                          |
  |   V                                                                          |
  | [ ACK Received & Matches Window Size? ] --YES--> Update Sincedb State with new offsets for acknowledged files
  |   | NO (Timeout or Mismatch)                                                 |
  |   V                                                                          |
  | [ Handle ACK Error/Timeout ]                                                 |
  |   - Log error                                                                |
  |   - Potentially resend batch or mark as failed (depending on error)          |
  |   - If connection lost, trigger reconnection sequence                        |
  |   |                                                                          |
  |   +--------------------------------------------------------------------------+
  |                                                                              |
  | Loop back to Main Processing Loop ------------------------------------------->
  |
  V
[ Handle Send Error ] / [ Connection Lost during Send/ACK ]
  |
  V
Log Error (e.g., "Connection lost to Logstash")
  |
  V
Attempt Reconnection to Logstash (with backoff/retry mechanism)
  |
  +-----> [ Reconnection Successful? ] --NO--> Keep Retrying (with increasing delay) / Log failures
  |         | YES
  |         V
  |       Log "Reconnected to Logstash"
  |         |
  |         +--------------------------------> Loop back to Main Processing Loop (to resend or continue)
  |
[END] (Typically runs indefinitely until interrupted, e.g., Ctrl+C)
```
