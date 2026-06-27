# 🧩 High Performance HTTP/HTTPS Proxy Server

A high-performance **forward proxy server** implemented in Modern **C++20**.  
This project focuses on systems programming, network socket management, and concurrency patterns. It implements a custom LRU cache, robust thread-safe logging, and supports both **HTTP (GET)** requests and **HTTPS (CONNECT)** tunneling.
**Tech:** C++20 • CMake • Socket Programming • HTTP/1.1 • HTTPS CONNECT • Multithreading • LRU Cache • RAII • GoogleTest

## ⭐ Highlights
- HTTP/1.1 GET proxy implementation
- HTTPS CONNECT tunneling
- Thread-safe custom LRU cache
- Thread-per-connection architecture
- Connection limiting with std::counting_semaphore
- RAII-based socket lifetime management
- Graceful shutdown via signal handling
- Thread-safe structured logging
- GoogleTest unit test suite
- Modular socket abstraction layer

## 🚀 What This Project Demonstrates

- TCP/IP socket programming
- HTTP protocol implementation
- Concurrent server architecture
- Thread synchronization
- RAII resource management
- Custom cache design
- Systems programming in Modern C++20

### ⚠️ Note on Ownership

- The core proxy server, cache, and logging infrastructure are fully implemented by me.
- The log analyzer dashboard (`log_analyzer.html`) and log format design (`proxy.log`) are included for demonstration only — I have not written the visualization code.

## 🚀 Key Features

### 🧵 C++20 Multi-Threaded Architecture

- Uses the **Thread-per-connection** model implemented via the std::thread.
- Decouples connection logic from the main acceptor loop, ensuring high responsiveness.

### 🔒 Semaphore-Based Connection Limiting
- Utilizes C++20 **std::counting_semaphore** to cap the maximum number of active clients (default: 2000).
- Prevents resource exhaustion (DoS) under heavy load without relying on OS-specific API calls for synchronization.

### 🌐 HTTP/1.1 GET Handling
- Parses incoming HTTP GET requests to extract host, port, and path.
- Forwards requests to origin servers and streams responses back to clients.
- **Automatic Caching:** Responses are intercepted and stored in memory to speed up subsequent requests.

### 🔐 HTTPS CONNECT Tunneling
- Implements the CONNECT method to handle SSL/TLS traffic.
- Establishes a bi-directional TCP tunnel between the client and the remote server.
- Uses `select()` I/O multiplexing to relay encrypted data efficiently between sockets.

### ⚡ Thread-Safe LRU Cache
- Custom **Least Recently Used (LRU)** cache implementation.
- Internals:
  - `std::unordered_map` for O(1) lookups.
  - Doubly‑linked list using `std::shared_ptr` and `std::weak_ptr` for memory-safe recency tracking.
  - Protected by `std::mutex` to ensure thread safety during concurrent access.

### 🧾 Modern Thread-Safe Logging

The project includes a centralized singleton logger featuring:
- Timestamped log entries
- C++20 `std::format`
- Thread-safe writes
- Automatic flushing
- Structured log output suitable for analysis dashboards


## 🖥️ Sample Execution

The log below demonstrates HTTPS tunneling, HTTP forwarding,
LRU caching behavior, and failure handling under real browser traffic.

```text
[INFO] Logger initialized
[INFO] Server started on port 8000
[INFO] LRU Cache initialized
[INFO] Listening for connections...

--- HTTPS CONNECT tunneling ---

[CLIENT] CONNECT www.google.com:443
[TUNNEL_ESTABLISHED]

[CLIENT] CONNECT chatgpt.com:443
[TUNNEL_ESTABLISHED]

[CLIENT] CONNECT api.github.com:443
[TUNNEL_ESTABLISHED]
[TUNNEL_CLOSED] 26842 bytes relayed


--- HTTP request with cache miss + store ---

[CLIENT] GET http://www.example.com/
[CACHE_MISS]

[REMOTE] Connecting to www.example.com:80
[REMOTE] HTTP/1.1 200 OK
[REMOTE] Forwarded 715 bytes

[CACHE_STORE] http://www.example.com/ (715 bytes)


--- Subsequent request served from cache ---

[CLIENT] GET http://www.example.com/
[CACHE_HIT]

[CLIENT] GET http://www.example.com/
[CACHE_HIT]


--- Example remote connection failure handling ---

[CLIENT] GET http://neverssl.com/
[CACHE_MISS]
[REMOTE_ERROR] Failed to connect to remote host
```

## ⚙️ Architecture Overview

### Overall Architecture
```text
                    Browser / HTTP Client
                             │
                             ▼
                  Listening TCP Socket
                             │
                             ▼
             Connection Counting Semaphore
                             │
                             ▼
                  Worker Thread (per Client)
                             │
               ┌─────────────┴─────────────┐
               │                           │
               ▼                           ▼
        HTTP/1.1 GET                 HTTPS CONNECT
               │                           │
               ▼                           ▼
      Thread-Safe LRU Cache          TCP Tunnel Relay
               │                           │
               ▼                           ▼
          Origin Server               Remote Server
```

### 1️⃣ Server Initialization (proxy_main.cpp)
The application performs the following initialization steps:

- Initializes the platform socket subsystem.
- Creates the global in-memory LRU cache.
- Initializes a std::counting_semaphore to limit concurrent client connections.
- Creates, binds, and listens on the server socket.
- Registers signal handlers for graceful shutdown.
- Enters the client accept loop.

### 2️⃣ Client Accept Loop (proxy_main.cpp)
The main server thread continuously waits for incoming client connections.

For each accepted connection, the server:
- Acquires a semaphore slot.
- Accepts the client socket.
- Creates a detached worker thread.
- Passes ownership of the client socket to the worker thread.

This architecture keeps the accept loop responsive while allowing multiple clients to be served concurrently.

### 3️⃣ Request Processing (proxy_handler.cpp)
Each worker thread processes exactly one client connection.

**🌐 HTTP GET Processing**
For HTTP requests the proxy:

- Reads the complete HTTP request header.
- Parses the absolute-form request URI.
- Extracts the destination host, port, and resource path.
- Checks the thread-safe LRU cache for a cached response.
- Immediately serves cached responses on a cache hit.
- On a cache miss:
  - Connects to the origin server.
  - Rebuilds the request into origin-form.
  - Removes duplicate `Host` and `Connection` headers.
  - Streams the response back to the client.
  - Stores cacheable responses in the LRU cache for future requests.

**🔐 HTTPS CONNECT Processing**
For HTTPS traffic the proxy:

- Parses the destination host and port.
- Establishes a TCP connection to the remote server.
- Returns `HTTP/1.1 200 Connection Established`.
- Uses `select()` to relay encrypted traffic between client and server.
- Closes the tunnel when either endpoint disconnects or a timeout occurs.

Since encrypted TLS packets are forwarded transparently, HTTPS communication remains end-to-end encrypted.

### 4️⃣ Thread-Safe LRU Cache (proxy_cache.cpp)
The cache combines:
- `std::unordered_map` for O(1) lookup.
- A custom doubly linked list for recency tracking.
- `std::shared_ptr` and `std::weak_ptr` for safe ownership.
- `std::mutex` for concurrent synchronization.

Supported operations include: 
| Operation | Complexity |
|-----------|-----------|
| Lookup | O(1) |
| Insert | O(1) |
| Update | O(1) |
| Promotion | O(1) |
| Eviction | O(1) |

The least recently used objects are automatically evicted whenever the cache exceeds its configured capacity.

### 5️⃣ Resource Management
The project follows RAII principles throughout the implementation.

Key resource management features include:

- Automatic socket cleanup using RAII wrapper classes.
- Automatic semaphore release using scope guards.
- Smart pointers for dynamic memory management.
- Graceful shutdown using signal handling.
- Exception-safe cleanup of networking resources.

### 🧪 Testing

The cache subsystem is validated using GoogleTest.

Current test coverage includes:
- Basic cache insertion and lookup
- Cache overwrite
- Cache miss handling
- LRU eviction
- Multiple-item eviction
- Capacity overflow handling
- Empty URL rejection
- Empty payload rejection
- Oversized object rejection
- Cache promotion after access
- Thread safety under concurrent access

## 📁 Project Structure

```
.
├── CMakeLists.txt         # CMake build configuration
├── proxy_main.cpp         # Server initialization and connection handling
├── proxy_utils.hpp        # Platform abstraction and RAII utilities
├── proxy_handler.cpp      # Logic for HTTP parsing, CONNECT tunneling, and relaying
├── proxy_handler.hpp      
├── proxy_cache.cpp        # Thread-safe custom LRU cache
├── proxy_cache.hpp
├── proxy_logger.cpp       # Thread-safe singleton logger
├── proxy_logger.hpp
├── log_analyzer.html      # Log visualization dashboard
├── proxy_cache_test.cpp   # GoogleTest unit tests
└── README.md
```

## 🧰 Dependencies

### Requirements

- C++20 compatible compiler
- CMake 3.26 or newer
- GoogleTest (optional, for unit testing)


## 🛠️ Building Instructions

### Clone the Repository
```bash
git clone https://github.com/jasbeersinghchauhan/proxyWebServer.git
cd proxy_web_server
```

### Configure the Build
```bash
mkdir build
cd build

cmake ..
```

### Compile
```bash
cmake --build . --config Release
```

## ▶️ Running the Proxy

### 1️⃣ Start the Proxy

```bash
./proxy_main 8080
```

### 2️⃣ Configure Your Browser

- Proxy IP: `127.0.0.1`

- Port: `8080`

- Enable proxy for both HTTP and HTTPS.

### 3️⃣ Browse the Web

- Visit HTTP sites (cached).

- Visit HTTPS sites (tunneled).

Observe real-time logging in the console and proxy.log.

### 4️⃣ Analyze Logs

1. Open log_analyzer.html in a browser.

2. Drag and drop the proxy.log file.

3. View cache hit rate, misses, and live parsed logs.

## 📈 Log Visualization Example

You can use the included log_analyzer.html — a simple HTML/JS dashboard that reads proxy.log and displays metrics using Chart.js:

```bash
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
```

That script loads the Chart.js library used for the pie chart visualization.

### ⚠️ Limitations

- ❌ HTTP Keep-Alive is not supported.

- 💾 GET-only caching (no POST/PUT/DELETE)

- 🔍 Implements a subset of the HTTP/1.1 specification and may not handle all edge cases.

- 🧠 In-memory cache — cleared on restart

## 🧭 Future Work / Roadmap

- 🔁 Thread pool (replace Thread-per-connection model)

- 🔄 Keep-alive connections

- 📨 Support additional HTTP methods (POST, PUT, DELETE)

- 💽 Persistent on-disk cache

- ⚙️ Config file support (config.ini)

## 🤝 Contributing

Contributions are welcome!

Fork the repo
Create a branch:

```bash
git checkout -b feature/my-new-feature
```

Commit your changes:

```bash
git commit -am "Add some feature"
```

Push to your branch:

```bash
git push origin feature/my-new-feature
```

Open a Pull Request

💬 For major changes, please open an issue first to discuss what you’d like to modify.

## 👤 Author

Jasbeer Singh Chauhan
📧 jasbeersinghchauhan377@gmail.com