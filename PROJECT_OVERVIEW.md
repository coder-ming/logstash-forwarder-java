## Project Overview: Logstash Forwarder in Java

**Primary Purpose:**

The project is a Java-based log forwarder designed to send logs from files to a Logstash instance. It serves as an alternative to the original Go-based logstash-forwarder, offering similar functionality within the Java ecosystem.

**Key Features:**

*   **Java Version Compatibility:** The project is built to be compatible with Java 7 and potentially newer versions, as indicated by the `maven-compiler-plugin` configuration in `pom.xml`.
*   **Lightweight:** The design emphasizes a minimal footprint, aiming to be a lightweight solution for log forwarding.
*   **Protocol:** It utilizes the Lumberjack protocol for communication with the Logstash instance. This is evident from `LumberjackClient.java` and mentions in the `README.md`.
*   **File Watching:** The application monitors specified files for changes (new log entries) and forwards them. This is handled by `FileWatcher.java`.
*   **State Persistence:** It keeps track of the last read position in each file, ensuring that log entries are not missed during restarts or interruptions. This is suggested by the `.logstash-forwarder` file mentioned in `README.md` and likely managed by `FileReader.java`.
*   **Configuration:** The forwarder is configured using a JSON file (e.g., `logstash-forwarder.conf`), which specifies the files to watch, the Logstash server details (host, port, SSL certificate), and other parameters.
*   **Security:** Supports SSL for secure communication with the Logstash server, configurable via the JSON configuration file.

**Main Technologies and Libraries Used:**

*   **Java:** The core programming language for the application.
*   **Maven:** Used as the build automation and project management tool, as defined in `pom.xml`.
*   **Apache Commons IO:** Leveraged for file utility operations, as indicated by its dependency in `pom.xml`. Specifically, `FileUtils` and `IOUtils` are likely used.
*   **Jackson:** Used for JSON processing, primarily for parsing the configuration file. Dependencies for `jackson-core`, `jackson-databind`, and `jackson-annotations` are present in `pom.xml`.
*   **Log4j:** Utilized for application logging, as shown by the `log4j` dependency.
*   **JUnit:** Used for unit testing, as indicated by the `junit` dependency in `pom.xml`.

**Origin:**

The project was created as a Java alternative to the original `logstash-forwarder` (which was written in Go). This was likely motivated by a desire to have a log forwarding solution that could be more easily integrated or managed within Java-centric environments.
