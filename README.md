# C# Client-Server Application using Socket Programming

## Summary
This project is a practical implementation of socket-based network communication in C#, demonstrating the development of server and client applications over a computer network. The application showcases TCP/IP communication using the **System.Net.Sockets** namespace with a WPF graphical interface.

<a href="https://imgur.com/CRI5mR1"><img src="https://i.imgur.com/CRI5mR1.png" title="source: imgur.com" /></a>

---

## Table of Contents
- [Overview](#overview)
- [Technologies Used](#technologies-used)
- [System Architecture](#system-architecture)
- [Socket Programming Concepts](#socket-programming-concepts)
- [Server Application](#server-application)
- [Client Application](#client-application)
- [Implementation Details](#implementation-details)
- [How to Use](#how-to-use)
- [Code Structure](#code-structure)
- [Features](#features)
- [Limitations](#limitations)

---

## Overview

This project demonstrates **socket programming exercises** focused on developing applications using the socket interface for communication between applications in a computer network. The implementation includes:

- **Server Applications**: A TCP server that listens for incoming connections and handles multiple concurrent clients
- **Client Applications**: Any TCP client (e.g., PuTTY, Telnet, custom client) that can connect to the server and send messages

The project serves as an educational resource for understanding network programming fundamentals, including:
- Socket creation and management
- TCP/IP protocol implementation
- Concurrent client handling
- Asynchronous communication patterns
- Network stream processing

---

## Technologies Used

| Technology | Purpose |
|------------|---------|
| **C# (.NET Framework 4.0)** | Core programming language |
| **System.Net.Sockets** | TCP/IP socket implementation |
| **WPF (Windows Presentation Foundation)** | User interface framework |
| **BackgroundWorker** | Asynchronous operations management |
| **TcpListener** | Server-side socket listener |
| **TcpClient** | Client connection management |
| **NetworkStream** | Data transmission over network |

---

## System Architecture

![image1](image1)

The system follows a **client-server architecture** where:

1. **Multiple Clients**: Any number of clients can connect simultaneously
2. **TCP Socket Layer**: Transport layer using TCP protocol for reliable communication
3. **Server Application**: Central application that manages all connections
4. **ClientHandler**: Individual handler for each client connection (one per client)
5. **WPF UI**: Graphical interface displaying server status and client messages

### Architecture Components:
- **Transport Layer**: TCP/IP protocol ensures reliable, ordered, and error-checked delivery
- **Application Layer**: Custom message protocol (ASCII text-based)
- **Presentation Layer**: WPF interface for monitoring and control

---

## Socket Programming Concepts

### What is a Socket?

A **socket** is an endpoint for sending or receiving data across a computer network. It's defined by:
- **IP Address**: Identifies the host machine
- **Port Number**: Identifies the specific application or service
- **Protocol**: Typically TCP or UDP

### TCP vs UDP

This implementation uses **TCP (Transmission Control Protocol)**:
- ✅ **Connection-oriented**: Establishes a connection before data transfer
- ✅ **Reliable**: Guarantees data delivery and order
- ✅ **Error-checking**: Built-in error detection and correction
- ❌ **Slower**: More overhead than UDP

### Socket Communication Flow

**Server Side:**
1. Create a socket
2. Bind to an IP address and port
3. Listen for connections
4. Accept incoming connections
5. Read/Write data
6. Close connection

**Client Side:**
1. Create a socket
2. Connect to server (IP + Port)
3. Send/Receive data
4. Close connection

---

## Server Application

### Server Architecture

![image2](image2)

### Key Components

#### 1. **TcpListener**
```csharp
TcpListener tcpListener = new TcpListener(IPAddress.Any, portNumber);
tcpListener.Start();
```
- Listens on specified port for incoming connections
- `IPAddress.Any` allows connections from any network interface

#### 2. **Server Class** (`Server.cs`)

**Properties:**
- `IPAddress localAddr`: Server's IP address
- `TcpListener tcpListener`: Socket listener
- `List<ClientHandler> chList`: List of connected clients
- `bool isServerRunning`: Server state flag
- `BackgroundWorker bgWorker`: Manages client acceptance loop
- `BackgroundWorker bgWorkerPingClients`: Monitors client connections

**Key Methods:**

##### `StartServer(string ipAddress, int port)`
```csharp
public string StartServer(string ipAddressString, int portNumber)
{
    localAddr = IPAddress.Parse(ipAddressString);
    tcpListener = new TcpListener(IPAddress.Any, portNumber);
    tcpListener.Start();
    isServerRunning = true;
    bgWorker.RunWorkerAsync();
    return "Server started successfully.";
}
```
- Parses IP address and port
- Initializes and starts TcpListener
- Launches background worker for accepting clients

##### `StopServer()`
```csharp
public string StopServer()
{
    tcpListener.Stop();
    tcpListener = null;
    return "Server stopped.";
}
```
- Stops accepting new connections
- Triggers cleanup of existing connections

##### `WaitForClient()`
```csharp
private void WaitForClient()
{
    TcpClient client = tcpListener.AcceptTcpClient();
    ClientHandler ch = new ClientHandler(client);
    chList.Add(ch);
}
```
- Blocks until a client connects (**blocking call**)
- Creates a `ClientHandler` for each connection
- Adds handler to the client list

### Server Lifecycle

1. **Initialization**: User enters IP and port in UI
2. **Start**: `TcpListener` begins listening
3. **Accept Loop**: Continuously accepts client connections (runs in background thread)
4. **Client Management**: Each client gets a dedicated `ClientHandler`
5. **Monitoring**: Background worker checks client connectivity every 250ms
6. **Shutdown**: Disconnects all clients and stops listener

### Concurrent Client Handling

The server uses **multi-threading** to handle multiple clients:
- **Main Thread**: UI updates and user interaction
- **Accept Thread**: Waits for new connections (BackgroundWorker)
- **Client Threads**: One per connected client (BackgroundWorker in ClientHandler)
- **Monitor Thread**: Checks client connectivity (BackgroundWorker)

---

## Client Application

### Client Handler Architecture

![image3](image3)

### ClientHandler Class (`ClientHandler.cs`)

The `ClientHandler` manages individual client connections.

**Properties:**
- `TcpClient ClientSocket`: Connected client socket
- `BackgroundWorker bgWorker`: Handles message reading asynchronously

**Lifecycle:**

#### 1. **Connection Established**
```csharp
public ClientHandler(TcpClient client)
{
    this.ClientSocket = client;
    InitializebgWorker();
    bgWorker.RunWorkerAsync();
}
```
- Receives `TcpClient` from server
- Starts background worker to read messages

#### 2. **Message Reading Loop**
```csharp
void bgWorker_DoWork(object obj, DoWorkEventArgs e)
{
    bgWorker.ReportProgress(2, ClientSocket); // Report connection
    
    while (ClientSocket != null)
    {
        string message = ReadFromClient();
        
        if (message != "")
            bgWorker.ReportProgress(1, message); // Report message
        else if (message == "" && !CheckIfClientAlive())
        {
            bgWorker.ReportProgress(0, ClientSocket); // Report disconnection
            break;
        }
    }
}
```

#### 3. **Reading from Network Stream**
```csharp
private string ReadFromClient()
{
    NetworkStream nwStream = ClientSocket.GetStream();
    byte[] buffer = new byte[ClientSocket.ReceiveBufferSize];
    int bytesRead = nwStream.Read(buffer, 0, ClientSocket.ReceiveBufferSize);
    string dataReceived = Encoding.ASCII.GetString(buffer, 0, bytesRead);
    return dataReceived;
}
```
- Gets `NetworkStream` from socket
- Reads bytes into buffer
- Converts bytes to ASCII string

#### 4. **Client Alive Check**
```csharp
private bool CheckIfClientAlive()
{
    return !(ClientSocket.Client.Poll(1000, SelectMode.SelectRead) 
             && ClientSocket.Client.Available == 0);
}
```
- Uses `Poll()` to check if socket is readable
- If readable but no data available, client has disconnected

#### 5. **Disconnection**
```csharp
private void CloseClientSocket()
{
    ClientSocket.Close();
    ClientSocket = null;
}
```

### Supported Client Types

Any TCP client can connect to this server:

1. **PuTTY**
   - Protocol: Raw TCP
   - Host: Server IP
   - Port: Server Port

2. **Telnet**
   ```bash
   telnet <server-ip> <port>
   ```

3. **Netcat**
   ```bash
   nc <server-ip> <port>
   ```

4. **Custom C# Client**
   ```csharp
   TcpClient client = new TcpClient("192.168.0.4", 12345);
   NetworkStream stream = client.GetStream();
   byte[] data = Encoding.ASCII.GetBytes("Hello Server");
   stream.Write(data, 0, data.Length);
   ```

---

## Implementation Details

### Class Diagram

![image4](image4)

### Key Design Patterns

#### 1. **Observer Pattern**
- BackgroundWorker reports progress to UI thread
- UI observes server events (connections, messages, disconnections)

#### 2. **Thread-Safe UI Updates**
```csharp
Application.Current.Dispatcher.Invoke(new Action(() =>
{
    ((MainWindow)Application.Current.MainWindow).textBoxInfo.AppendText(message);
}), DispatcherPriority.ContextIdle);
```
- Uses `Dispatcher` to marshal calls to UI thread
- Prevents cross-thread exceptions

#### 3. **Asynchronous Processing**
- `BackgroundWorker` handles I/O operations off the UI thread
- Prevents UI freezing during blocking socket operations

### Threading Model

| Thread | Purpose | Lifetime |
|--------|---------|----------|
| **UI Thread** | Handles WPF interface | Application lifetime |
| **Server Accept Thread** | Waits for client connections | While server running |
| **Client Read Thread** | Reads messages from one client | Per client connection |
| **Monitor Thread** | Checks client connectivity | While server running |

### Data Flow

1. **Client → Server:**
   - Client sends bytes over TCP
   - `NetworkStream.Read()` receives bytes
   - Bytes converted to ASCII string
   - String displayed in UI

2. **Server → UI:**
   - Background worker detects event
   - `ReportProgress()` called with data
   - `Dispatcher.Invoke()` updates UI safely

---

## How to Use

### Starting the Server

1. **Launch the application**
2. **Configure connection:**
   - **IP Address**: Enter server's IP (e.g., `192.168.0.4`)
   - **Port**: Enter port number (e.g., `12345`)
3. **Click "Start Server"**
4. **Monitor**: Server status appears in text box

### Connecting a Client

#### Using PuTTY:
1. Select **Connection type**: Raw
2. Enter **Host Name**: Server IP address
3. Enter **Port**: Server port number
4. Click **Open**
5. Type messages and press Enter

#### Using Telnet:
```bash
telnet 192.168.0.4 12345
```

#### Using Netcat:
```bash
nc 192.168.0.4 12345
```

### Stopping the Server

1. **Click "Stop Server"**
2. All clients are forcibly disconnected
3. Server stops accepting new connections

---

## Code Structure

```
ClientServerCore/
├── App.xaml                    # Application definition
├── App.xaml.cs                 # Application code-behind
├── MainWindow.xaml             # UI layout
├── MainWindow.xaml.cs          # UI logic
├── Server.cs                   # Server implementation
├── ClientHandler.cs            # Client connection handler
├── ClientServer.csproj         # Project file
└── Properties/
    ├── AssemblyInfo.cs
    ├── Resources.resx
    └── Settings.settings
```

### File Descriptions

| File | Description |
|------|-------------|
| `Server.cs` | Manages TcpListener, accepts clients, maintains client list |
| `ClientHandler.cs` | Handles individual client connections, reads messages |
| `MainWindow.xaml` | WPF UI design with text boxes, buttons, labels |
| `MainWindow.xaml.cs` | Event handlers for UI buttons, server instance |

---

## Features

### ✅ Implemented Features

- ✅ **TCP Server**: Listens on configurable IP and port
- ✅ **Multiple Concurrent Clients**: Handles unlimited simultaneous connections
- ✅ **Asynchronous Processing**: Non-blocking I/O operations
- ✅ **Connection Monitoring**: Real-time client count display
- ✅ **Automatic Disconnection Detection**: Detects client disconnects
- ✅ **Message Display**: Shows all received messages with client IP
- ✅ **Graceful Shutdown**: Cleanly disconnects all clients on stop
- ✅ **Thread-Safe UI**: Safe updates from background threads

### 🔄 Partial/Future Features

- ⚠️ **Ping Functionality**: UI button exists but not implemented (TO DO)
- 🔄 **Server-to-Client Messages**: Currently one-way (client → server)
- 🔄 **Protocol Definition**: No structured message format

---

## Limitations

1. **One-Way Communication**: Server receives but doesn't send messages to clients
2. **No Authentication**: Any client can connect without credentials
3. **No Encryption**: Messages sent in plain text (ASCII)
4. **No Message Protocol**: No structured format for messages
5. **No Logging**: No persistent log of connections/messages
6. **Windows Only**: WPF is Windows-specific
7. **No Configuration File**: Settings must be entered manually

---

## Educational Value

This project demonstrates essential networking concepts:

### Socket Programming Fundamentals
- Creating and managing TCP sockets
- Binding to IP addresses and ports
- Listening for and accepting connections
- Reading from network streams

### Concurrent Programming
- Multi-threading with BackgroundWorker
- Thread-safe UI updates with Dispatcher
- Managing multiple asynchronous operations

### Network Communication
- TCP/IP protocol usage
- Client-server architecture
- Connection lifecycle management

### Software Design
- Separation of concerns (UI, Network, Business Logic)
- Event-driven architecture
- Resource management and cleanup

---

## Potential Enhancements

1. **Bidirectional Communication**: Send messages from server to specific clients
2. **Message Protocol**: Implement structured messages (e.g., JSON, Protocol Buffers)
3. **Authentication**: Add login mechanism
4. **Encryption**: Use TLS/SSL for secure communication
5. **Chat Rooms**: Implement multiple channels/rooms
6. **File Transfer**: Support sending files between clients
7. **Persistent Storage**: Log messages to database
8. **Cross-Platform**: Port to .NET Core with Avalonia UI

---

## Socket Programming Best Practices Demonstrated

✅ **Proper Resource Disposal**: Sockets are closed when no longer needed  
✅ **Exception Handling**: Try-catch blocks around socket operations  
✅ **Non-Blocking UI**: Network I/O doesn't freeze the interface  
✅ **Connection Monitoring**: Regular checks for disconnected clients  
✅ **Graceful Shutdown**: Clean disconnection process  

---

## Academic Context

This project fulfills the requirements for **socket programming exercises** in network programming courses, covering:

### Server Application Development
- ✅ Implementation of TCP server using socket interface
- ✅ Handling multiple concurrent client connections
- ✅ Network stream processing and data reception
- ✅ Connection lifecycle management

### Client Application Development
- ✅ Understanding client-side socket connection
- ✅ Demonstration with multiple client types (PuTTY, Telnet, custom)
- ✅ Data transmission over network
- ✅ Connection establishment and termination

### Network Communication Concepts
- ✅ TCP/IP protocol implementation
- ✅ Socket creation and binding
- ✅ Listening and accepting connections
- ✅ Bidirectional data streams (NetworkStream)

---

## References and Learning Resources

- [Microsoft Docs: System.Net.Sockets](https://docs.microsoft.com/en-us/dotnet/api/system.net.sockets)
- [TCP/IP Protocol Suite](https://en.wikipedia.org/wiki/Internet_protocol_suite)
- [Socket Programming in C#](https://docs.microsoft.com/en-us/dotnet/fundamentals/networking/sockets/socket-services)

---

## License

This is an educational project. Feel free to use and modify for learning purposes.

---

## Author

Created as a practical exercise in network programming and socket-based application development.

---

*Last Updated: 2025*