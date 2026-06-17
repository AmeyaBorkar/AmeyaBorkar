# JavaChat

> A WhatsApp-style desktop chat clone written in plain Java: a multi-threaded TCP socket server (one `Thread` per client) speaking a custom newline-delimited text protocol, a Swing GUI client with custom-painted chat bubbles, and MySQL persistence over raw JDBC. Implements registration/login, one-to-one direct conversations, online/offline presence, offline message queueing, and a three-tier SENT/DELIVERED/READ status system. **Note:** the repo as committed does not compile — five server files import non-existent `whatsapp.common.*` / `whatsapp.server.*` packages instead of the actual `model.*` / `server.*` packages (see "README vs. code").

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/JavaChat |
| **Visibility** | Public |
| **Category** | apps |
| **Primary language(s)** | Java (100%) |
| **Local path** | `C:\Users\ameya\.repo-cache\JavaChat` |
| **Default branch** | `main` |
| **Lines of code (computed)** | 4,192 lines of Java across 17 `.java` files (plus a 6-line `database.properties` and a 344-line `README.md`) |
| **Source files (computed)** | 17 `.java` files; 3 non-code files (`.gitignore`, `README.md`, `server/config/database.properties`) |
| **Key components** | `server/` (5 files), `frontend/` (6 files), `model/` (6 files) |
| **License** | None (no `LICENSE` file; README states "created for educational purposes") |
| **Last commit** | `3114d14` — "Updated ReadMe" — 2026-02-11 (single squashed commit; `git rev-list --count HEAD` = 1) |

*All line/file counts computed with `find ... -name '*.java' | wc -l` and `wc -l` over the working tree, excluding `.git`. No `.jar`, `.class`, or `lib/` artifacts are present in the clone — jars are gitignored (`*.jar`).*

---

## What it actually is

JavaChat (README title: "WhatsApp Clone – Real-Time Chat Application") is a two-process, client/server desktop chat app:

- A **server** (`server.WhatsAppServer`) that listens on TCP port **8080**, accepts socket connections, and spawns a dedicated `ClientHandler` thread per client.
- A **Swing client** (`frontend.WhatsAppClient`) that connects to the server, shows login/register windows, then a main chat window with a chat list, a user-discovery list, and a custom-rendered message area.
- A shared **`model`** package of POJOs and a `Protocol` class with the wire-format constants and helpers used by both sides.
- **MySQL** persistence accessed via raw JDBC (`com.mysql.cj.jdbc.Driver`) through a singleton `DatabaseManager`.

The real, working feature set (verified in code): registration, password login (plaintext comparison), direct (1:1) conversations created on demand, persistent message history, online/offline presence broadcasting, offline-message delivery on login, unread counters, and per-recipient SENT/DELIVERED/READ status. Group chat, encryption, file sharing, and typing indicators are **not** implemented (README lists them as future work, and the code confirms their absence — see gaps).

### Computed file inventory (by lines)

```
frontend/MainChatWindow.java     1193   server/DatabaseManager.java       827
server/ClientHandler.java         678   frontend/NetworkManager.java      374
frontend/RegisterWindow.java      222   frontend/LoginWindow.java         215
server/UserManager.java           153   server/MessageRouter.java         134
server/WhatsAppServer.java         94   model/Conversation.java            74
model/Protocol.java                58   model/User.java                    55
model/Message.java                 50   frontend/WhatsAppClient.java       29
frontend/TestConnection.java       17   model/MessageType.java             13
model/MessageStatus.java            6
                                                          TOTAL: 4192 lines
```

---

## Architecture & how it's structured

Three flat packages (no Maven/Gradle; no build tree). The MySQL JDBC jar is expected in a `lib/` folder that is **not** in the repo (gitignored).

```
JavaChat/
├── server/                       # Server process + DB access
│   ├── WhatsAppServer.java        # main(): ServerSocket on 8080, accept loop, per-client thread
│   ├── ClientHandler.java         # extends Thread; parses protocol, dispatches commands
│   ├── DatabaseManager.java       # singleton; all JDBC + SQL (the de-facto schema lives here)
│   ├── UserManager.java           # singleton; ConcurrentHashMap of online users -> ClientHandler
│   ├── MessageRouter.java         # singleton; routes a saved message to recipients' handlers
│   └── config/
│       └── database.properties    # db.url / db.username / db.password (root/root, whatsapp_clone_db)
├── frontend/                      # Swing client process
│   ├── WhatsAppClient.java        # main(): builds NetworkManager + LoginWindow on the EDT
│   ├── NetworkManager.java        # socket I/O + background reader thread; routes server msgs to UI
│   ├── LoginWindow.java           # JFrame login form
│   ├── RegisterWindow.java        # JDialog registration form
│   ├── MainChatWindow.java        # JFrame chat UI (bubbles, lists, custom renderers) — largest file
│   └── TestConnection.java        # tiny standalone socket-connectivity smoke test
└── model/                        # Shared POJOs + protocol constants
    ├── Protocol.java              # delimiters, command/response constants, format/parse helpers
    ├── User.java                  # userId, username, displayName, status, isOnline, lastSeen
    ├── Message.java               # messageId, conversationId, senderId, content, sentAt, type
    ├── Conversation.java          # id, name, type(DIRECT|GROUP), members, lastMessage
    ├── MessageType.java           # enum: TEXT, SYSTEM, LOGIN, ... (declared; mostly unused on the wire)
    └── MessageStatus.java         # enum: SENT, DELIVERED, READ (declared; status is sent as strings)
```

**Note on package integrity:** `model/` POJOs and `server/` classes use the right package declarations (`package model;`, `package server;`). But `ClientHandler`, `MessageRouter`, and `WhatsAppServer` `import whatsapp.common.*` / `whatsapp.server.*` — packages that don't exist anywhere in the tree. As committed this fails `javac`. The intended types are `model.Conversation.ConversationType`, `model.Protocol`, and the inner classes `server.DatabaseManager.ConversationInfo` / `.MessageWithStatus`.

---

## The custom TCP protocol — actual wire format

Defined entirely in `model/Protocol.java`. It is a **text, line-oriented protocol**: every logical message is a single line terminated by the socket's newline (sent via `PrintWriter.println(...)` and read via `BufferedReader.readLine()` on both ends).

### Framing

- **Field delimiter** (`FIELD_DELIMITER`): `::`
- **Message terminator** (`MESSAGE_DELIMITER`): `|||`
- A message is: `COMMAND::param1::param2::...|||`, then a newline from `println`.

```java
// Protocol.formatMessage
sb.append(command);
for (param : params) sb.append("::").append(param);
sb.append("|||");

// Protocol.parseMessage
if (message.endsWith("|||")) strip it;
return message.split("::");   // parts[0] = command, parts[1..] = params
```

There is **no length prefix, no opcode byte, no binary framing** — it's split-on-`::`. This makes the protocol fragile: any `::` or `|||` inside a username, display name, or message body would corrupt parsing (no escaping is done anywhere).

### Client → Server commands (from `Protocol` + `ClientHandler.handleClientMessage`)

| Command | Params | Handler |
|---|---|---|
| `LOGIN` | `username::password` | `handleLogin` |
| `REGISTER` | `username::password::displayName` | `handleRegister` |
| `LOGOUT` | (none) | `handleLogout` |
| `GET_CONVERSATIONS` | (none) | `handleGetConversations` |
| `GET_MESSAGES` | `conversationId` | `handleGetMessages` (also marks read) |
| `SEND_MESSAGE` | `conversationId::content` | `handleSendMessage` |
| `SEND_DIRECT_MESSAGE` | `targetDisplayName::content` | `handleSendDirectMessage` |
| `CREATE_CHAT` | `targetDisplayName` | `handleCreateChat` — **string literal, not in `Protocol`** |
| `MARK_READ` | `conversationId` | `handleMarkRead` — **string literal, not in `Protocol`** |
| `REQUEST_MESSAGE_STATUS` | `messageId` | `handleRequestMessageStatus` |
| `GET_USERS` | (none) | `handleGetUsers` |

Declared in `Protocol` but **never handled** by the server: `CREATE_GROUP`, `JOIN_GROUP`, `SEND_DIRECT_MESSAGE` is handled but the client never sends it (the client sends `SEND_MESSAGE` and `CREATE_CHAT`).

### Server → Client responses

| Message | Shape | Notes |
|---|---|---|
| `SUCCESS` | `SUCCESS::...` | Generic OK. Login reply: `SUCCESS::LOGIN::userId::username::displayName::statusMessage`. Send reply: `SUCCESS::Message sent::messageId`. |
| `ERROR` | `ERROR::reason` | |
| `NEW_MESSAGE` | `NEW_MESSAGE::conversationId::messageId::senderName::content::timestamp` | Pushed by `MessageRouter`. |
| `CONVERSATION_LIST_WITH_INFO` | `...::id,type,displayName,unreadCount,lastSenderName,lastContent,lastSentAt::...` | **Uses commas inside each `::` field**, parsed by `split(",")` on the client. |
| `MESSAGE_LIST_WITH_STATUS` | `...::messageId,senderName,content,sentAt,status,userRole::...` | Comma-CSV per field; client splits with `split(",", 6)`. |
| `USER_LIST` | `...::userId,displayName,ONLINE\|OFFLINE,lastSeen::...` | |
| `USER_ONLINE` / `USER_OFFLINE` | `USER_ONLINE::userId::displayName` | Broadcast presence. |
| `DELIVERY_RECEIPT` | `DELIVERY_RECEIPT::conversationId::userName` | |
| `READ_RECEIPT` | `READ_RECEIPT::conversationId::userName` | |
| `MESSAGE_STATUS_UPDATE` | `MESSAGE_STATUS_UPDATE::messageId::status` | Reply to `REQUEST_MESSAGE_STATUS`. |
| `SYSTEM_MESSAGE` | `SYSTEM_MESSAGE::conversationId::text::timestamp` | Emitted by `MessageRouter.broadcastSystemMessage` (no caller wired up). |
| `NEW_CONVERSATION` | `NEW_CONVERSATION::id::name` | Emitted by `MessageRouter.notifyNewConversation` (no caller wired up). |

**Two-layer delimiter quirk:** list responses use `::` to separate *rows* but `,` to separate *columns within a row*. So the protocol effectively has two delimiter schemes depending on the message type. The client `NetworkManager` also handles `CONVERSATION_LIST` and `MESSAGE_LIST` (comma-less variants) for which the server has **no producer** — dead branches.

---

## Server — concurrency model, connection handling, routing

**Entry point** (`WhatsAppServer.start`): opens `new ServerSocket(8080)`, prints a banner, calls `DatabaseManager.getInstance()` to force a DB connection, then loops `serverSocket.accept()`. For each accepted socket it does `new ClientHandler(socket).start()` — i.e. **thread-per-client** (the README's claim is accurate here). A JVM shutdown hook calls `server.stop()` which closes the socket and the DB connection.

**`ClientHandler extends Thread`:** in its constructor it wraps the socket in a `BufferedReader` and an auto-flush `PrintWriter`. Its `run()` loops `reader.readLine()` and dispatches each line through a big `switch` in `handleClientMessage`. On disconnect/IO error, `cleanup()` removes the user from `UserManager` and closes streams. Per-client state is the `currentUser` (set after login).

**`UserManager` (singleton)** is the shared online-user registry, the only genuinely concurrent structure:
- `ConcurrentHashMap<Integer, ClientHandler> onlineUsers` (userId → handler)
- `ConcurrentHashMap<Integer, User> userProfiles` (userId → profile)
- `addOnlineUser` registers the handler, flips DB `is_online`, and broadcasts `USER_ONLINE` after a 500ms `javax.swing.Timer` delay (an oddity — using a Swing UI timer inside the server). `sendMessageToUser` returns `true` if the user was online (used to decide DELIVERED vs. pending).

**`MessageRouter` (singleton)** does delivery: given a saved `messageId`, it loads the conversation's members, builds a `NEW_MESSAGE` payload, and for each member ≠ sender calls `UserManager.sendMessageToUser`. If the recipient is online it marks the row DELIVERED, adds an unread entry, and sends a `DELIVERY_RECEIPT` back to the sender. If offline, the message simply stays in `SENT` status and is reconciled at the recipient's next login.

**Thread-safety caveats (real, not in README):** `getInstance()` on all three singletons (`UserManager`, `MessageRouter`, `DatabaseManager`) uses lazy non-synchronized initialization — not safe under concurrent first-call. More importantly, **a single shared JDBC `Connection`** is used by every `ClientHandler` thread (`DatabaseManager` holds one `Connection` and toggles `setAutoCommit(false)` for transactions in `saveMessage` / `markMessagesAsRead`). Concurrent transactions on one shared connection are not isolated and can interfere. The README's "Connection Pooling" claim is **false** — there is no pool, just one connection.

**Routing model:** delivery is **direct to conversation members** (not a global broadcast). Presence changes (`USER_ONLINE`/`USER_OFFLINE`) *are* broadcast to all other online users. `broadcastToAllUsers` exists but is unused.

---

## Client — the Swing UI, screens, event handling

**`WhatsAppClient.main`** runs everything on the EDT via `SwingUtilities.invokeLater`, constructs a `NetworkManager` and a `LoginWindow`.

**`NetworkManager`** owns the socket. `connect()` opens it and starts a **daemon background reader thread** that loops `reader.readLine()`; each received line is dispatched to the UI via `SwingUtilities.invokeLater(() -> handleServerMessage(message))`. `handleServerMessage` is the client-side `switch` mirroring the server's response types, forwarding to whichever window is set (`loginWindow`, `registerWindow`, `mainWindow`).

**Screens:**
- **`LoginWindow` (JFrame, 350×250):** username + password fields, Login/Register buttons, WhatsApp-green header. On login it connects (if needed), sends `LOGIN::user::pass`, and arms a 10s `javax.swing.Timer` timeout. On `SUCCESS::LOGIN::...` it disposes itself and opens `MainChatWindow`.
- **`RegisterWindow` (JDialog, modal, 400×300):** username / display name / password / confirm. Client-side validation: username ≥ 3 chars, password ≥ 4 chars, passwords must match. Sends `REGISTER::user::pass::displayName`, 10s timeout.
- **`MainChatWindow` (JFrame, 1100×700, the 1,193-line centerpiece):**
  - Left sidebar with a gradient-painted header (custom `paintComponent`), a circular avatar drawn from the user's initial, and a `JTabbedPane` with **Chats** and **Users** tabs.
  - **Chats tab:** `JList` of conversations with a custom `ModernConversationRenderer` (avatar, name, "Tap to open" preview, and a green unread-count badge). A "+ New Chat" button opens a `JOptionPane` user picker that issues `CREATE_CHAT::displayName`.
  - **Users tab:** `JList` of all users via `ModernUserRenderer` (avatar with an online dot, ONLINE/OFFLINE label). Double-click a user → `CREATE_CHAT`.
  - **Chat area:** a `BoxLayout` panel of `MessageBubblePanel`s — custom-painted rounded rectangles (`RoundRectangle2D`, antialiasing), green for sent, white for received, with a per-message timestamp and a status glyph. `getStatusIcon` maps SENT→`✓`, DELIVERED/READ→`✓✓`, else `⏳`. A "🔄" refresh button re-requests `GET_MESSAGES` + `GET_CONVERSATIONS`.
  - **Sending:** typing enables the send button; Enter or click sends `SEND_MESSAGE::convId::text`, immediately appends an optimistic bubble with a `⏳` glyph, and clears the field.
  - **Opening a conversation** sends `MARK_READ::id` then `GET_MESSAGES::id`, clears the panel, and zeroes the local unread count.

**Timestamp handling** (`parseServerTime` / `formatMessageTimestamp`) accepts ISO instants (`...Z`), `LocalDateTime` strings, or bare dates, and renders "HH:mm" / "Yesterday HH:mm" / "EEE HH:mm" / "MMM dd …" relative formats.

**Design tension visible in code:** many handlers are deliberately "quiet" (`updateMessageDeliveryStatusQuiet`, `updateSpecificMessageStatus` is a no-op, `REFRESH_CONVERSATIONS` is explicitly ignored). Comments throughout say things like "FIXED: Don't auto-refresh to prevent loops" — the app previously had a refresh-storm bug and the fix was to stop auto-refreshing, which is why the user must hit 🔄 to see a brand-new conversation appear.

---

## Persistence — MySQL schema & queries

There is **no `.sql` schema file in the repo** (no `*.sql`; `*.sql` is even gitignored). The schema is implied entirely by the SQL strings in `DatabaseManager.java` (and documented, mostly-accurately, in the README's "Database Schema" section). Connection config: `server/config/database.properties` → database `whatsapp_clone_db`, user/pass `root`/`root`, `jdbc:mysql://localhost:3306/...`. **Discrepancy:** the properties file uses DB name `whatsapp_clone_db`, while the README's setup commands say `CREATE DATABASE whatsapp_clone;` — the names don't match.

`DatabaseManager` is a singleton holding **one** `Connection`, loaded with `Class.forName("com.mysql.cj.jdbc.Driver")`. All access uses `PreparedStatement` (good — parameterized, SQLi-resistant) except `getAllUsers` which uses a plain `Statement` on a static query.

### Tables referenced by the code

| Table | Columns the code reads/writes |
|---|---|
| `users` | `user_id` PK, `username`, `password_hash`, `display_name`, `status_message`, `is_online`, `last_seen` |
| `conversations` | `conversation_id` PK, `name`, `type` ('DIRECT'/'GROUP'), `created_by`, `created_at` |
| `conversation_members` | `conversation_id`, `user_id`, `is_active`, (`joined_at`) — junction table |
| `messages` | `message_id` PK, `conversation_id`, `sender_id`, `content`, `message_type` ('TEXT'), `sent_at` |
| `message_status` | `message_id`, `user_id`, `status`, `delivered_at`, `read_at` — per-recipient status |
| `unread_messages` | `user_id`, `conversation_id`, `message_id` — unread tracking / counters |

### Key queries / operations

- **Auth:** `SELECT ... FROM users WHERE username=? AND password_hash=?` — plaintext compare (the comment literally says "In real app, hash the password first").
- **Register:** `INSERT INTO users (username, password_hash, display_name) VALUES (?,?,?)` — stores password verbatim into `password_hash`.
- **Save message (transactional):** `saveMessage` sets `autoCommit=false`, inserts the message with `RETURN_GENERATED_KEYS`, then seeds `message_status` rows for **all** active members via `INSERT ... SELECT ... FROM conversation_members`, all status `'SENT'`, then commits (rollback on failure).
- **Find/create direct chat:** `findOrCreateDirectConversation` does a bidirectional self-join over `conversation_members` to find an existing DIRECT conversation between two users, else `createDirectConversation` inserts a `conversations` row and two `conversation_members` rows.
- **Status transitions:** `markMessageAsDelivered`, `markMessagesAsRead` (deletes from `unread_messages` and `UPDATE message_status ... SET status='read'` in one transaction), `markConversationMessagesAsDelivered`. Note an inconsistency: code writes `'read'` lowercase but `'DELIVERED'`/`'SENT'` uppercase, and queries sometimes compare `status='read'` — relies on MySQL's default case-insensitive collation to work.
- **Aggregated status for sender:** `getMessageStatusForSender` / `getMessageStatus` count recipients' rows (`WHERE user_id != sender`) and roll up to READ (all read) / DELIVERED (all ≥ delivered) / SENT.
- **Unread counts:** `getUnreadCount` / `getTotalUnreadCount` over `unread_messages`.
- **Timezone:** `getConversationMessagesWithStatus` uses `CONVERT_TZ(m.sent_at, @@session.time_zone, '+00:00')` then converts to system-default zone — a workaround for a date-shift bug noted in comments.

---

## Auth / registration

- **Registration:** client validates (username ≥3, password ≥4, confirm match), sends `REGISTER::user::pass::displayName`; server inserts into `users`. Uniqueness is enforced only by the DB (`username` UNIQUE per the README schema) — on insert failure the server returns "Registration failed. Username might already exist."
- **Login:** `LOGIN::user::pass`; server runs the plaintext `SELECT`. On success it sets `currentUser`, registers the handler in `UserManager`, flips `is_online`, reconciles offline messages (mark delivered + add to unread), and replies `SUCCESS::LOGIN::id::username::displayName::status`.
- **Security reality:** passwords are stored and compared **in plaintext** (column is named `password_hash` but no hashing occurs — confirmed by the in-code comments). There are no sessions/tokens; identity is just the live socket's `currentUser`. There is no rate limiting and no input escaping for `::`/`|||`.
- **Logout:** `LOGOUT` removes the user from `UserManager` (broadcasting `USER_OFFLINE`), replies `SUCCESS::Logged out`, and flips `isConnected=false`.

---

## Message delivery model (broadcast vs. direct vs. offline)

- **Direct, member-scoped delivery.** `SEND_MESSAGE::convId::text` → server saves the message + seeds per-member `message_status`, then `MessageRouter.routeMessage` sends `NEW_MESSAGE` only to the conversation's other members who are online. Not a global broadcast.
- **Offline storage / queue.** If a recipient is offline, the message row stays `SENT`. On that user's next login, `markPendingMessagesAsDelivered` marks their pending rows DELIVERED and `addPendingMessagesToUnread` populates `unread_messages` so the conversation shows an unread badge. So "offline message queue" is implemented via DB state reconciliation, not an in-memory queue.
- **Receipts.** Online delivery triggers a `DELIVERY_RECEIPT` to the sender; opening a conversation (`GET_MESSAGES`/`MARK_READ`) marks rows read and conditionally emits `READ_RECEIPT`. The client treats these "quietly" (status-bar notifications only) to avoid the historical refresh-loop.
- **Presence broadcast.** `USER_ONLINE`/`USER_OFFLINE` *are* broadcast to all other online clients.

---

## Tech stack & build

- **Language/runtime:** Java (README says JDK 8+; uses `java.time`, `BoxLayout`, lambdas — all fine on 8+). 100% Java by language; no other languages.
- **UI:** Java Swing/AWT, hand-rolled custom rendering (gradients, rounded bubbles, circular avatars) — no UI libraries.
- **Networking:** `java.net.Socket` / `ServerSocket`, `BufferedReader`/`PrintWriter` line I/O. No NIO, no Netty.
- **Persistence:** MySQL via JDBC, driver `com.mysql.cj.jdbc.Driver` (README references `mysql-connector-j-9.4.0.jar`).
- **Build system:** none. No `pom.xml`, no `build.gradle`, no `Makefile`, no `.sln`. Compilation is manual `javac` with the connector jar on the classpath.
- **Dependencies in repo:** none vendored. `lib/` (with the jar) is gitignored and absent from the clone; you must supply the connector yourself.

---

## Build / run

> These mirror the README, with corrections. **Important:** the committed code will not compile until the bogus `whatsapp.common.*` / `whatsapp.server.*` imports in `server/WhatsAppServer.java`, `server/ClientHandler.java`, and `server/MessageRouter.java` are changed to `model.*` / `server.*` (see next section).

1. **Get the MySQL connector** (e.g. `mysql-connector-j-9.4.0.jar`) and place it in a `lib/` directory at the repo root.

2. **Create the database and tables.** No schema file ships, so create `whatsapp_clone_db` (match `database.properties`, *not* the README's `whatsapp_clone`) and create the six tables from the SQL in `DatabaseManager.java` / the README schema section. Roughly:
   ```sql
   CREATE DATABASE whatsapp_clone_db;
   USE whatsapp_clone_db;
   -- users, conversations, conversation_members, messages, message_status, unread_messages
   -- (see README "Database Schema" for full DDL)
   ```

3. **Configure** `server/config/database.properties` (`db.url`, `db.username`, `db.password`). Defaults are `root`/`root`.

4. **Compile** (run from repo root; the server reads `server/config/database.properties` via a *relative* path, so always run from the root):
   ```bash
   # Linux/macOS
   javac -cp "lib/mysql-connector-j-9.4.0.jar:." server/*.java frontend/*.java model/*.java
   # Windows (PowerShell/cmd) — use ; as the classpath separator
   javac -cp "lib/mysql-connector-j-9.4.0.jar;." server/*.java frontend/*.java model/*.java
   ```

5. **Run the server:**
   ```bash
   java -cp "lib/mysql-connector-j-9.4.0.jar:." server.WhatsAppServer
   ```

6. **Run one or more clients:**
   ```bash
   java -cp "lib/mysql-connector-j-9.4.0.jar:." frontend.WhatsAppClient
   ```
   (`frontend.TestConnection` is a quick socket-only connectivity check against `localhost:8080`.)

---

## Status, completeness & notable gaps

**Working (in code):** registration, plaintext login, presence, direct 1:1 chat with on-demand conversation creation, persistent history, unread counters, SENT/DELIVERED/READ status, offline reconciliation on login, a polished custom Swing UI.

**Gaps / issues found by reading the source:**
- **Does not compile as committed** — phantom `whatsapp.*` imports (the single biggest issue).
- **Plaintext passwords** — no hashing despite the `password_hash` column name; comments admit it.
- **Single shared JDBC connection** across all client threads with `autoCommit` toggling — race-prone; README's "Connection Pooling" is inaccurate.
- **No `::`/`,`/`|||` escaping** — any delimiter character inside a message, name, or display name corrupts the protocol.
- **Group chat absent** — `CREATE_GROUP`/`JOIN_GROUP` constants exist but have no server handlers; `Conversation.ConversationType.GROUP` is defined but never created.
- **Dead/unwired code** — `SEND_DIRECT_MESSAGE` handler (client never sends it), `CONVERSATION_LIST`/`MESSAGE_LIST` client branches (server never produces them), `broadcastSystemMessage`/`notifyNewConversation`/`broadcastToAllUsers` (no callers), `MessageType`/`MessageStatus` enums (status travels as raw strings).
- **Manual refresh required** — real-time conversation-list updates were disabled to fix a refresh-storm; new chats only appear after clicking 🔄.
- **No tests** beyond the trivial `TestConnection`. No CI, no license, single squashed commit.

---

## README vs. code

| README claim | Code reality |
|---|---|
| "Multi-threaded server with thread-per-client" | **True** — `new ClientHandler(socket).start()` per accept. |
| Field/message delimiters `:::` for fields, `\|\|\|` for messages | **Partly wrong** — field delimiter is `::` (two colons, `FIELD_DELIMITER="::"`), not `:::`. Message delimiter `\|\|\|` is correct. |
| "Connection Pooling: Singleton DatabaseManager for efficient connections" | **False** — one shared `Connection`, no pool. |
| "Secure Login / Password-based authentication" | **Misleading** — plaintext password stored & compared; column named `password_hash` but unhashed. |
| "Group chat functionality" listed under *future* work | Consistent — not implemented; constants exist but unhandled. |
| "Offline Message Queue: delivered when users come back online" | **True** in effect — implemented as DB reconciliation on login, not an in-memory queue. |
| "Three-Tier Status (Sent/Delivered/Read)" | **True** — implemented via `message_status` + receipts. |
| Setup: `CREATE DATABASE whatsapp_clone` | **Mismatch** — `database.properties` uses `whatsapp_clone_db`. |
| Compile/run commands with `lib/mysql-connector-j-9.4.0.jar` | Plausible, but `lib/` and the jar are **gitignored / absent** from the repo; you must supply them. |
| Project tree shows `lib/` and `misc/` | Neither is in the repo (both gitignored); `TestConnection.java` exists but isn't listed in the README tree. |
| README never mentions it | The repo **won't compile** due to `whatsapp.common.*` / `whatsapp.server.*` imports that resolve to no package. |

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\.repo-cache\JavaChat. Source of truth: https://github.com/AmeyaBorkar/JavaChat*
