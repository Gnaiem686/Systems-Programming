# 🏆 SPL231 Assignment 3 – STOMP World Cup Updates System

This repository contains my full implementation of **SPL Assignment 3**, including a **Java STOMP server** and a **C++ World Cup Informer client**.  
The system enables multiple users to subscribe to game channels, publish events, receive real-time updates, and generate game summaries.

This project demonstrates event-driven networking, concurrency, protocol parsing, client/server architecture, and runtime environments.

---

## 📦 Project Structure

```text
SPL231-Assignment3/
│
├── Server/
│   ├── src/
│   ├── pom.xml
│   └── (TPC & Reactor implementations)
│
└── Client/
    ├── src/
    ├── include/
    ├── bin/
    └── Makefile
```

- **Server/** – Java implementation of a STOMP server supporting both Thread-Per-Client and Reactor patterns.  
- **Client/** – C++ implementation of the World Cup Informer STOMP client.

---

## 🚀 Assignment Overview

The goal of this assignment is to implement a “community-led” **World Cup update subscription service**.

- Users connect to a central **STOMP server**.
- They **subscribe** to game channels (topics) such as `germany_japan`.
- They **send reports** (game events) which are broadcast to all subscribers.
- Each client maintains a local view of the game and can output a **summary** to a file.

All communication between the client and server follows the  
**STOMP – Simple Text Oriented Messaging Protocol (v1.2).**

---

## 📘 STOMP Protocol Summary

### STOMP Frame Format

A STOMP frame has the structure:

```text
<COMMAND>
header1:value1
header2:value2

<body>^@
```

- One command line (e.g., `SEND`, `SUBSCRIBE`).
- Zero or more headers (`key:value`).
- A blank line.
- Optional body.
- Null terminator (`\0`).

### Frames Used in This Assignment

#### Client → Server

- `CONNECT` – initiate a session.
- `SEND` – send a message to a destination (topic).
- `SUBSCRIBE` – subscribe to a topic (game channel).
- `UNSUBSCRIBE` – stop receiving from a topic.
- `DISCONNECT` – gracefully disconnect from server.

#### Server → Client

- `CONNECTED` – acknowledge successful connection.
- `MESSAGE` – delivers messages from subscribed topics.
- `RECEIPT` – ack for frames that requested `receipt`.
- `ERROR` – error notification, followed by closing the connection.

---

## 🖥️ Java Server

The server is written in **Java**, uses **Maven** as a build tool, and supports:

- **Thread-Per-Client (TPC)** – a dedicated thread per connection.
- **Reactor** – non-blocking I/O with a selector and worker threads.

### Key Interfaces

#### `Connections<T>`

Manages all active client connections.

- `boolean send(int connectionId, T msg)`  
  Sends a message to a specific client.
- `void send(String channel, T msg)`  
  Sends a message to all clients subscribed to a given channel (topic).
- `void disconnect(int connectionId)`  
  Removes the client from the active connections map.

#### `ConnectionHandler<T>`

Responsible for I/O with a single client.

- `void send(T msg)` – sends a message to the client over the socket.

#### `StompMessagingProtocol`

Implements the STOMP logic.

- `void start(int connectionId, Connections<String> connections)`  
  Initializes protocol with connection id and shared `Connections`.
- `void process(String message)`  
  Parses and handles a STOMP frame.
- `boolean shouldTerminate()`  
  Indicates if the connection should be closed.

### Server Responsibilities

- Parse incoming STOMP frames.
- Handle:
  - `CONNECT` / `CONNECTED`
  - Subscriptions (`SUBSCRIBE` / `UNSUBSCRIBE`)
  - Message routing (`SEND` → `MESSAGE` to subscribers)
  - Receipts (`RECEIPT` for frames with a `receipt` header)
  - Errors (`ERROR` frames and closing the connection)
- Maintain:
  - Mapping: **connectionId → client**
  - Subscriptions: **channel (topic) → subscribers**
  - Unique `message-id` values

The server is **stateless with regard to game logic** – it only routes messages.  
All game semantics are handled by the client.

---

## 🎮 C++ Client – STOMP World Cup Informer

The client is written in **C++** and uses:

- **Two threads**:
  - One for **keyboard input**.
  - One for **socket reading**.
- A **Makefile** for building.
- STOMP frames for communication with the server.

The client translates **console commands** into appropriate STOMP frames and updates its internal game state according to messages received from the server.

### Multithreading Model

- **Keyboard thread**:
  - Reads commands from stdin.
  - Builds and sends STOMP frames.
- **Socket listener thread**:
  - Reads STOMP frames from the server.
  - Handles `CONNECTED`, `MESSAGE`, `RECEIPT`, `ERROR`.
  - Updates game states and subscription info.

---

## 💻 Client Commands

All commands except `login` require the client to be logged in.

### 1. `login {host:port} {username} {password}`

- Opens a TCP connection and sends a `CONNECT` frame.
- Possible outcomes:
  - Could not connect → print:  
    `Could not connect to server`
  - Already logged in →  
    `The client is already logged in, log out before trying again`
  - New user created successfully →  
    `Login successful`
  - Existing user, correct password, no active session →  
    `Login successful`
  - User already logged in elsewhere → `User already logged in`
  - Wrong password → `Wrong password`

Example frame:

```text
CONNECT
accept-version:1.2
host:stomp.cs.bgu.ac.il
login:meni
passcode:films

^@
```

---

### 2. `join {game_name}`

Subscribes to a game channel.

- Sends a `SUBSCRIBE` frame with:
  - `destination:/<game_name>`
  - unique `id`
  - optional `receipt`
- On `RECEIPT`, prints:  
  `Joined channel {game_name}`

Example:

```text
SUBSCRIBE
destination:/germany_spain
id:17
receipt:73

^@
```

---

### 3. `exit {game_name}`

Unsubscribes from a game channel.

- Sends `UNSUBSCRIBE` with the subscription `id`.
- On `RECEIPT`, prints:  
  `Exited channel {game_name}`

---

### 4. `report {file}`

Publishes game events from a JSON file.

Steps:

1. Parse `{file}` (JSON) using the provided C++ parser (`Event`, `parseEventsFile`).
2. For each game event:
   - Save it locally as part of the game’s event history, per user.
   - Send a `SEND` frame to the topic `/game_name` with all event details in the body.

Example `SEND` frame:

```text
SEND
destination:/spain_japan
user:meni
team a:spain
team b:japan
event name:kickoff
time:0
general game updates:
active:true
before halftime:true
team a updates:
a:b
c:d
e:f
team b updates:
a:b
c:d
e:f
description:
And we’re off!

^@
```

The **body format** is crucial and must follow the assignment’s specification so that other clients can parse it correctly.

---

### 5. `summary {game_name} {user} {file}`

Generates a summary file locally using events from `{user}` for `{game_name}`.

- Uses all previously saved events for that (game, user).
- Aggregates stats:
  - General stats
  - Team A stats
  - Team B stats
- Orders:
  - Stats: lexicographically by stat name.
  - Events: by game time (with halftime considerations).

Output format:

```text
<team_a> vs <team_b>
Game stats:
General stats:
stat1: value1
stat2: value2
...
<team_a> stats:
stat1: value1
...
<team_b> stats:
stat1: value1
...

Game event reports:
time1 - eventName1:
description1
time2 - eventName2:
description2
...
```

If `{file}` exists, overwrite it; otherwise, create it.

---

### 6. `logout`

Gracefully disconnects.

- Sends a `DISCONNECT` frame with a unique `receipt` header.
- Waits for a `RECEIPT` with the same `receipt-id`.
- Closes the socket and returns to a state where new `login` is possible.
- Removes the user from all channel subscriptions.

Example:

```text
DISCONNECT
receipt:113

^@
```

Server response:

```text
RECEIPT
receipt-id:113

^@
```

---

## 🎮 Game Events & JSON Format

Game events are supplied in JSON files. Example:

```json
{
  "team a": "Germany",
  "team b": "Japan",
  "events": [
    {
      "event name": "kickoff",
      "time": 0,
      "general game updates": {
        "active": true,
        "before halftime": true
      },
      "team a updates": {},
      "team b updates": {},
      "description": "The game has started! What an exciting evening!"
    }
  ]
}
```

Each event contains:

- `event name`
- `time` (in seconds)
- `description`
- `general game updates` (map)
- `team a updates` (map)
- `team b updates` (map)

The client:

- Uses the provided `parseEventsFile(std::string json_path)` function.
- Maintains:
  - Per-game stats and event list.
  - Per-user separation of reports.
- Handles halftime and time-order edge cases as specified.

---

## 🛠 Build & Run Instructions

### 🔧 Server (Java + Maven)

From the `Server/` directory:

```bash
mvn compile
```

Run as Thread-Per-Client:

```bash
mvn exec:java -Dexec.mainClass="bgu.spl.net.impl.stomp.StompServer" -Dexec.args="<port> tpc"
```

Run as Reactor:

```bash
mvn exec:java -Dexec.mainClass="bgu.spl.net.impl.stomp.StompServer" -Dexec.args="<port> reactor"
```

---

### 🔧 Client (C++ + Makefile)

From the `Client/` directory:

```bash
make
./bin/StompWCIClient
```

Then in the client console, you can run commands such as:

```text
login 127.0.0.1:7777 meni films
join germany_japan
report data/events1_partial.json
summary germany_japan meni summary.txt
logout
```

---

## 🧪 Testing Notes

- You can initially test using the **Echo** example (given in the course template) to ensure basic networking is correct.
- Make sure `receipt` headers are handled correctly for:
  - `SUBSCRIBE`
  - `UNSUBSCRIBE`
  - `DISCONNECT`
- On `ERROR` frames:
  - Print a helpful message to stdout.
  - Close the connection if required.

---

## 🎯 Main Concepts Demonstrated

- STOMP-based messaging.
- Client/Server architecture.
- Thread-Per-Client vs. Reactor server patterns.
- Multithreaded C++ client design.
- Text protocol parsing & frame construction.
- Game state aggregation from distributed events.
- Use of JSON parsing in C++.

---

## 👤 Author

**Mohammad Gnaim**  
SPL231 – Assignment 3  
Ben-Gurion University of the Negev
