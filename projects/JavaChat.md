# JavaChat

> Real-time messaging application in Java with a multi-threaded TCP server, Swing GUI, MySQL persistence, and a hand-rolled wire protocol supporting sent/delivered/read status and offline message queuing.

**Repository:** [`AmeyaBorkar/JavaChat`](https://github.com/AmeyaBorkar/JavaChat)  
**Category:** Networking / Application  
**Visibility:** Public  
**Primary language:** Java  
**Default branch:** `main`  
**License:** —  
**Created:** 2026-02-11  
**Last pushed:** 2026-02-11  
**Metadata updated:** 2026-02-11  
**Size (GitHub reported):** 37 KB  

---

## What it is (one-paragraph version)

Real-time messaging application in Java with a multi-threaded TCP server, Swing GUI, MySQL persistence, and a hand-rolled wire protocol supporting sent/delivered/read status and offline message queuing.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| Java | 167,381 | 100.0% |

## File tree

- Total entries indexed: **24** (20 files, 4 directories)

```
.gitignore  (480 B)
README.md  (11 KB)
frontend/    [6 files]
  frontend/LoginWindow.java
  frontend/MainChatWindow.java
  frontend/NetworkManager.java
  frontend/RegisterWindow.java
  frontend/TestConnection.java
  frontend/WhatsAppClient.java
model/    [6 files]
  model/Conversation.java
  model/Message.java
  model/MessageStatus.java
  model/MessageType.java
  model/Protocol.java
  model/User.java
server/    [6 files]
  server/ClientHandler.java
  server/DatabaseManager.java
  server/MessageRouter.java
  server/UserManager.java
  server/WhatsAppServer.java
  server/config/database.properties
```

## README (verbatim)

# 💬 WhatsApp Clone - Real-Time Chat Application

A feature-rich, real-time messaging application built with Java, featuring a multi-threaded server architecture, MySQL database integration, and a modern Swing-based GUI. This project demonstrates advanced socket programming, concurrent user management, and real-time message delivery with comprehensive status tracking.

---

## 🎯 Key Features

### 🔐 Authentication & User Management
- **User Registration**: Create new accounts with username, password, and display name
- **Secure Login**: Password-based authentication with session management
- **Online Presence**: Real-time online/offline status indicators
- **Last Seen**: Track when users were last active
- **User Discovery**: Browse all registered users to start conversations

### 💬 Messaging Capabilities
- **Direct Messaging**: One-on-one private conversations
- **Real-Time Delivery**: Instant message delivery to online users
- **Offline Message Queue**: Messages delivered when users come back online
- **Message History**: Persistent storage of all conversations in MySQL database
- **Conversation Management**: Create and manage multiple chat threads

### ✅ Advanced Message Status Tracking
- **Three-Tier Status System**:
  - ✓ **Sent**: Message saved to database
  - ✓✓ **Delivered**: Message received by recipient's client
  - ✓✓ **Read**: Recipient opened the conversation
- **Real-Time Status Updates**: Senders see status changes instantly
- **Per-User Status**: Track read/delivered status for each conversation participant
- **Unread Message Counters**: Visual badges showing unread message counts

### 🎨 Modern User Interface
- **Custom Swing Components**: Modern chat bubbles with rounded corners
- **Gradient Headers**: Eye-catching gradient designs
- **Message Grouping**: Messages grouped by sender with timestamps
- **Responsive Layout**: Adapts to different window sizes
- **Tabbed Navigation**: Separate tabs for chats and user discovery
- **Visual Feedback**: Hover effects, status icons, and smooth animations

---

## 🏗️ Architecture & Technical Highlights

### 📡 Network Architecture
- **Protocol**: Custom TCP/IP-based application protocol
- **Server Model**: Multi-threaded server with thread-per-client architecture
- **Port**: Configurable (default: 8080)
- **Message Format**: Delimiter-based protocol (`:::` for fields, `|||` for messages)
- **Persistent Connections**: Long-lived socket connections for real-time updates

### 🧵 Concurrency & Thread Safety
- **ClientHandler Threads**: Each client connection runs in a dedicated thread
- **ConcurrentHashMap**: Thread-safe user session management
- **UserManager Singleton**: Centralized concurrent user tracking
- **MessageRouter**: Efficient message routing to online users
- **Atomic Operations**: Transaction-based database operations for consistency

### 💾 Database Design
- **Normalized Schema**: Efficient relational database structure
- **Key Tables**:
  - `users`: User accounts and authentication
  - `conversations`: Chat threads (direct/group)
  - `conversation_members`: Many-to-many relationship
  - `messages`: Message content and metadata
  - `message_status`: Per-user message status tracking
  - `unread_messages`: Unread message tracking
- **JDBC Integration**: Prepared statements for SQL injection prevention
- **Connection Pooling**: Singleton DatabaseManager for efficient connections

### 🎨 Design Patterns
- **Singleton Pattern**: DatabaseManager, UserManager, MessageRouter
- **MVC Architecture**: Separation of model, view, and controller logic
- **Observer Pattern**: Real-time status updates and notifications
- **Factory Pattern**: Protocol message formatting and parsing
- **Thread-Per-Request**: Scalable client handling

---

## 📁 Project Structure

```
whatsapp-clone/
├── server/                    # Server-side logic
│   ├── WhatsAppServer.java   # Main server entry point
│   ├── ClientHandler.java    # Per-client connection handler
│   ├── DatabaseManager.java  # Database operations & connection
│   ├── UserManager.java      # Online user session management
│   ├── MessageRouter.java    # Message routing logic
│   └── config/
│       └── database.properties
├── frontend/                  # Client-side GUI
│   ├── WhatsAppClient.java   # Main client entry point
│   ├── LoginWindow.java      # Login interface
│   ├── RegisterWindow.java   # Registration interface
│   ├── MainChatWindow.java   # Main chat interface
│   └── NetworkManager.java   # Client-server communication
├── model/                     # Shared data models
│   ├── Protocol.java         # Protocol constants & utilities
│   ├── User.java            # User entity
│   ├── Message.java         # Message entity
│   ├── Conversation.java    # Conversation entity
│   ├── MessageType.java     # Message type enum
│   └── MessageStatus.java   # Status enum
├── lib/                      # External dependencies
│   └── mysql-connector-j-9.4.0.jar
└── misc/                     # IDE settings & compiled files
```

---

## 🚀 Getting Started

### Prerequisites

- **Java Development Kit (JDK)**: Version 8 or higher
- **MySQL Server**: Version 5.7 or higher
- **MySQL Connector/J**: Included in `lib/` directory

### Database Setup

1. **Create Database**:
   ```sql
   CREATE DATABASE whatsapp_clone;
   USE whatsapp_clone;
   ```

2. **Create Tables** (see database schema section below)

3. **Configure Connection**:
   Edit `server/config/database.properties`:
   ```properties
   db.url=jdbc:mysql://localhost:3306/whatsapp_clone
   db.username=your_username
   db.password=your_password
   ```

### Compilation

```bash
# Navigate to project root
cd whatsapp-clone

# Compile all source files
javac -cp "lib/mysql-connector-j-9.4.0.jar;." server/*.java frontend/*.java model/*.java
```

> **Note**: On Linux/Mac, use `:` instead of `;` in the classpath:
> ```bash
> javac -cp "lib/mysql-connector-j-9.4.0.jar:." server/*.java frontend/*.java model/*.java
> ```

### Running the Application

1. **Start the Server**:
   ```bash
   java -cp "lib/mysql-connector-j-9.4.0.jar;." server.WhatsAppServer
   ```
   
   You should see:
   ```
   =================================
   WhatsApp Clone Server Started
   Listening on port: 8080
   =================================
   Testing database connection...
   Database connected successfully!
   ```

2. **Launch Client(s)**:
   ```bash
   java -cp "lib/mysql-connector-j-9.4.0.jar;." frontend.WhatsAppClient
   ```
   
   You can launch multiple client instances to test messaging between users.

---

## 📊 Database Schema

### Users Table
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    display_name VARCHAR(100) NOT NULL,
    status_message VARCHAR(255),
    is_online BOOLEAN DEFAULT FALSE,
    last_seen TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Conversations Table
```sql
CREATE TABLE conversations (
    conversation_id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    type ENUM('DIRECT', 'GROUP') NOT NULL,
    created_by INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (created_by) REFERENCES users(user_id)
);
```

### Conversation Members Table
```sql
CREATE TABLE conversation_members (
    conversation_id INT,
    user_id INT,
    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (conversation_id, user_id),
    FOREIGN KEY (conversation_id) REFERENCES conversations(conversation_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

### Messages Table
```sql
CREATE TABLE messages (
    message_id INT PRIMARY KEY AUTO_INCREMENT,
    conversation_id INT NOT NULL,
    sender_id INT NOT NULL,
    content TEXT NOT NULL,
    message_type ENUM('TEXT', 'IMAGE', 'FILE') DEFAULT 'TEXT',
    sent_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (conversation_id) REFERENCES conversations(conversation_id),
    FOREIGN KEY (sender_id) REFERENCES users(user_id)
);
```

### Message Status Table
```sql
CREATE TABLE message_status (
    message_id INT,
    user_id INT,
    status ENUM('SENT', 'DELIVERED', 'READ') DEFAULT 'SENT',
    delivered_at TIMESTAMP NULL,
    read_at TIMESTAMP NULL,
    PRIMARY KEY (message_id, user_id),
    FOREIGN KEY (message_id) REFERENCES messages(message_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

### Unread Messages Table
```sql
CREATE TABLE unread_messages (
    user_id INT,
    conversation_id INT,
    message_id INT,
    PRIMARY KEY (user_id, message_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (conversation_id) REFERENCES conversations(conversation_id),
    FOREIGN KEY (message_id) REFERENCES messages(message_id)
);
```

---

## 🔧 Technical Implementation Details

### Protocol Design

The application uses a custom text-based protocol:

**Message Format**: `COMMAND::param1::param2::...|||`

**Example Messages**:
```
LOGIN::john_doe::password123|||
SEND_MESSAGE::5::Hello, how are you?|||
MESSAGE_STATUS_UPDATE::42::READ|||
```

### Message Flow

1. **Sending a Message**:
   ```
   Client → Server: SEND_MESSAGE::conversationId::content|||
   Server → Database: INSERT INTO messages
   Server → Database: INSERT INTO message_status (for all members)
   Server → Sender: SUCCESS::Message sent::messageId|||
   Server → Recipients: NEW_MESSAGE::messageId::senderId::content::timestamp|||
   ```

2. **Status Updates**:
   ```
   Recipient Client Receives Message → DELIVERED status
   Recipient Opens Conversation → READ status
   Server → Sender: MESSAGE_STATUS_UPDATE::messageId::READ|||
   ```

### Concurrency Handling

- **User Sessions**: `ConcurrentHashMap<Integer, ClientHandler>` for thread-safe access
- **Message Routing**: Synchronized methods prevent race conditions
- **Database Transactions**: Atomic operations for message + status insertion
- **Graceful Shutdown**: Cleanup handlers for proper resource deallocation

---

## 🎓 Learning Outcomes & Skills Demonstrated

- ✅ **Socket Programming**: TCP/IP client-server communication
- ✅ **Multi-threading**: Concurrent request handling and thread safety
- ✅ **Database Design**: Normalized schema with complex relationships
- ✅ **JDBC**: Database connectivity and prepared statements
- ✅ **GUI Development**: Swing components and custom rendering
- ✅ **Protocol Design**: Custom application-layer messaging protocol
- ✅ **Design Patterns**: Singleton, MVC, Observer, Factory
- ✅ **Real-Time Systems**: Event-driven architecture for instant updates
- ✅ **State Management**: Session tracking and user presence

---

## 🚧 Future Enhancements

- [ ] Group chat functionality
- [ ] File/image sharing
- [ ] End-to-end encryption
- [ ] Voice/video calling
- [ ] Push notifications
- [ ] Message search
- [ ] User blocking
- [ ] Typing indicators
- [ ] Message reactions
- [ ] Profile pictures

---

## 📝 License

This project is created for educational purposes.

---

## 👤 Author

**Ameya Borkar**

*A demonstration of advanced Java programming, networking, and database integration skills.*

---

*Generated 2026-05-02 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/JavaChat*
