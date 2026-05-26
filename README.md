# Q1: UDP P2P Chat

A peer-to-peer chat application built with Python's native `socket` library. Each peer sends and receives messages simultaneously using threads, following a custom binary protocol over UDP.

## Problem Statement

Proposed by Prof. Rodrigo Campiolo as part of the Distributed Systems course. The goal is to implement a P2P chat where clients exchange messages directly with each other using UDP sockets and a structured binary protocol supporting different message types (normal, emoji, URL, and ECHO).

---

## Project Structure

```
q1/
├── chat_udp.py      # Entry point: starts the peer, reads terminal input and sends packets
├── network.py       # Socket creation, packet sending, and receiver thread
└── protocol.py      # build_packet / parse_packet and message type constants
```

---

## Requirements

- Python 3.8+
- No external libraries required

---

## How to Run

Open **two terminals** at the project root.

**Terminal 1: Peer A:**

```bash
python chat_udp.py 127.0.0.1 5000 127.0.0.1 5001
```

**Terminal 2: Peer B:**

```bash
python chat_udp.py 127.0.0.1 5001 127.0.0.1 5000
```

> Each peer binds to its own port and sends to the other's port.  
> To connect from another machine, replace `127.0.0.1` with the target machine's IP address.

---

## Supported Commands

| Command         | Description                                                         |
| --------------- | ------------------------------------------------------------------- |
| `<text>`        | Sends a normal message                                              |
| `/emoji <text>` | Sends a message of type emoji                                       |
| `/url <link>`   | Sends a message of type URL                                         |
| `/echo <text>`  | Sends an ECHO request to check if the other peer is active          |
| `/help`         | Displays available commands (local only, not sent over the network) |
| `/quit`         | Closes the chat                                                     |

---

## Binary Protocol

All communication uses a **binary protocol** with variable-length fields.

### Packet structure

```
| 1 byte: Message Type | 1 byte: Nickname Size | N bytes: Nickname | 1 byte: Message Size | N bytes: Message |
```

### Field constraints

| Field         | Size    | Constraint      |
| ------------- | ------- | --------------- |
| Message Type  | 1 byte  | See table below |
| Nickname Size | 1 byte  | 1 to 64         |
| Nickname      | N bytes | UTF-8 encoded   |
| Message Size  | 1 byte  | 0 to 255        |
| Message       | N bytes | UTF-8 encoded   |

### Message type identifiers

| Type         | Code |
| ------------ | ---- |
| MSG_NORMAL   | 0x01 |
| MSG_EMOJI    | 0x02 |
| MSG_URL      | 0x03 |
| MSG_ECHO_REQ | 0x04 |
| MSG_ECHO_RES | 0x05 |

### ECHO flow

To avoid infinite loops, ECHO is split into two distinct types:

```
Peer A  --MSG_ECHO_REQ-->  Peer B   (Peer A asks if Peer B is active)
Peer A  <--MSG_ECHO_RES--  Peer B   (Peer B confirms and does not reply again)
```

---

## Architecture

Each peer runs **two concurrent execution flows**:

- **Main thread**: blocked on `input()`, reads user commands and sends packets
- **Receiver thread**: blocked on `recvfrom()`, listens for incoming packets and prints them

The receiver thread is started as a `daemon`, meaning it is automatically terminated when the main thread exits.

---

## Error Handling

- **Peer:** `Ctrl+C` exits cleanly without a traceback.
- **Peer:** Malformed packets (too short or incomplete) are detected in `parse_packet` and discarded with a warning.
- **Peer:** Nicknames longer than 64 characters and messages longer than 255 characters are silently truncated before sending.

# Q2: UDP File Transfer

A client-server application built with Python's native `socket` library for transferring files over a network. It uses a custom binary protocol built on top of UDP, implementing a Stop-and-Wait flow control mechanism to ensure reliable delivery and SHA-1 checksums for final integrity verification.

## Problem Statement

Proposed by Prof. Rodrigo Campiolo as part of the Distributed Systems course. The objective is to build a UDP file upload system. A UDP server must receive files in chunks (1024 bytes), verify data integrity using an SHA-1 checksum at the end of the transfer, and save the file in a default folder. The protocol must be explicitly designed and handle binary data (e.g., images, PDFs) of arbitrary sizes.

---

## Project Structure

```

q2/
├── client.py # Connects to the server, reads a local file, and sends it chunk by chunk
├── server.py # Listens for incoming connections, reconstructs files, and verifies checksums
├── protocol.py # Packet construction (build) and interpretation (parse) logic
├── client-storage/ # Some files to move to server just for tests
└── server-storage/ # Default directory where the server saves received files

```

---

## Requirements

- Python 3.8+
- No external libraries required

---

## How to Run

Open **two terminals** at the project root.

**Terminal 1: Server:**

```bash
python server.py
```

> The server will start listening on port 6730 (default).

**Terminal 2: Client:**

```bash
python client.py client-storage/potato.png
```

> The client will read `client-storage/potato.png` and begin the upload process. Check the `server-storage` folder after completion.

---

## The Problem with Large Files and the Stop-and-Wait Solution

### The Problem: Bursting and Packet Loss

After testing the first version of this code, I tried to transfer an image larger than 400KB and encountered an error: several data chunks were missing.

UDP is a connectionless protocol that lacks built-in flow or congestion control. When attempting to send large files using a simple loop, the client injects thousands of packets into the network in a fraction of a second (a phenomenon known as a packet "burst"). This rapid influx quickly overwhelms the memory buffers of the operating system, routers, or the receiving server.

When a buffer gets full, the network handles incoming UDP packets by silently dropping them. Consequently, the server reconstructs a file with missing chunks, leading to data corruption and an inevitable Checksum (SHA-1) mismatch at the end of the transfer.

### The Solution: Stop-and-Wait

To guarantee data integrity without introducing the extreme complexity of TCP-like algorithms (such as Sliding Windows), a **Stop-and-Wait** flow control mechanism was implemented.

How it solves the problem:

1. The client sends a single payload and starts a timer (timeout).
2. The server processes it and replies with an explicit acknowledgment (`PKT_ACK`).
3. The client only sends the next chunk _after_ receiving this ACK.
4. If a packet or its ACK is dropped (timer expires), the client simply retransmits it.

**Why this approach?**
Stop-and-Wait effectively acts as a dynamic rate limiter, preventing buffer overflows by forcing the sender to match the receiver's processing speed. While it introduces a slight overhead due to waiting for Round-Trip Times (RTT), it is incredibly reliable. In a Local Area Network (LAN) or localhost environment—where this project is typically executed and evaluated—the RTT is near zero, meaning transfers remain extremely fast, 100% reliable, and the codebase remains elegant and easy to maintain.

## Flow Control

Because UDP is connectionless and does not guarantee delivery, this application implements a stop-and-Wait strategy:

1. The client sends a data chunk (up to 1024 bytes).
2. The client sets a timeout (e.g., 0.5s) and waits for an acknowledgment (ACK).
3. The server receives the chunk and immediately replies with an ACK for that specific chunk number.
4. If the client receives the correct ACK, it moves to the next chunk.
5. If the client's timeout expires (meaning the packet or the ACK was lost), the client retransmits the same chunk.

---

## Binary Protocol

All communication uses a custom **binary protocol** defined in `protocol.py`.

### Packet Types

| Type           | Code | Description                                                                        |
| :------------- | :--- | :--------------------------------------------------------------------------------- |
| `PKT_INFO`     | 0x01 | Initial packet sent by the client. Contains the filename and total file size.      |
| `PKT_DATA`     | 0x02 | Contains the chunk sequence number and the actual file payload (up to 1024 bytes). |
| `PKT_CHECKSUM` | 0x03 | Final packet sent by the client. Contains the SHA-1 hash of the complete file.     |
| `PKT_ACK`      | 0x04 | Sent by the server to confirm receipt of a specific data chunk.                    |

### Packet Structures

**PKT_INFO (1)**

```
| 1 byte: Type (1) | 1 byte: Filename Size | N bytes: Filename | 4 bytes: File Size |
```

**PKT_DATA (2)**

```
| 1 byte: Type (2) | 4 bytes: Chunk Number | 2 bytes: Data Size | N bytes: Payload |
```

**PKT_CHECKSUM (3)**

```
| 1 byte: Type (3) | 20 bytes: SHA-1 Hash |
```

**PKT_ACK (4)**

```
| 1 byte: Type (4) | 4 bytes: Chunk Number |
```

---

## Server Workflow

1. **Receive `PKT_INFO`:** Creates a new transfer record in memory linked to the client's address, storing the filename and expected size.
2. **Receive `PKT_DATA`:** Extracts the chunk number and payload. Saves the chunk in the transfer record and immediately sends a `PKT_ACK` back to the client.
3. **Receive `PKT_CHECKSUM`:** Reconstructs the file bytes by ordering the chunks. Computes the SHA-1 hash of the assembled data and compares it to the received hash.
4. **Finalize:** If the checksums match, the file is saved to the `server-storage` directory. If they differ, the file is discarded with a warning.

---

## Authors

<table>
  <tr>
    <td align="center">
      <a href="https://github.com/GabrielCraco">
        <img src="https://github.com/GabrielCraco.png" width="80px" alt="Gabriel Craco Tasarz"/><br/>
        <sub><b>Gabriel Craco Tasarz</b></sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/leonardoozima">
        <img src="https://github.com/leonardoozima.png" width="80px" alt="Leonardo Jun'Ity Ozima"/><br/>
        <sub><b>Leonardo Jun'Ity Ozima</b></sub>
      </a>
    </td>
  </tr>
</table>


```

```
