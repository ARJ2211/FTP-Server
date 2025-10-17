# CS549 FTP (RMI + Sockets) — README

A tiny FTP-like system in Java.  
**Control channel:** Java RMI for commands.  
**Data channel:** a fresh TCP socket per file (like classic FTP).

---

## ✨ Features

- `pwd`, `cd`, `dir`, `ldir` for remote/local navigation
- `get` and `put` with **Passive (PASV)** and **Active (PORT)** modes
- Works locally or across multiple EC2 instances
- Minimal configuration via CLI flags / properties
- Easy submission packaging (sources + jars + videos + rubric)

---

## 🧠 Architecture (30-sec version)

- **RMI (control):** client ⇄ server for commands (fast, tiny messages).
- **TCP (data):** one ephemeral socket per transfer.

```
Passive (PASV)                           Active (PORT)
-----------------                        -----------------
Server: listen(data port S)              Client: listen(data port C)
Client: connect(S)                       Server: connect(C)
Direction: client→server connects        Direction: server→client connects
```

---

## 🗂 Project Layout

```
ftpinterface/   RMI interfaces
ftpserver/      Server (builds ftpd.jar)
ftpclient/      Client (builds ftp.jar)
```

---

## 🔧 Build

```bash
# from repo root
mvn clean package

# artifacts:
#   ftpserver/target/ftpd.jar
#   ftpclient/target/ftp.jar
```

---

## 🚀 Run Locally (single machine)

**Server (Terminal 1):**
```bash
cd ftpserver/target
# bind passive data sockets to loopback for local testing
java -jar ftpd.jar --serverIp 127.0.0.1 --serverPort 1099
```

**Client (Terminal 2):**
```bash
cd ftpclient/target
java -jar ftp.jar --serverAddr localhost --serverPort 1099 --clientIp 127.0.0.1
```

Create a test tree next to where you start the **server** (e.g., `root/hello.txt`), then use commands below.

---

## ☁️ Run Across Two EC2 Instances

**Assumptions**
- Same VPC; use **private IPs** (e.g., Server `172.31.25.252`, Client `172.31.24.120`).
- One security group for both is fine.

**Security Group (simple demo setup)**
- Inbound TCP **from 172.31.24.120/32** (client) — enables **PASV** data + **RMI**.
- Inbound TCP **from 172.31.25.252/32** (server) — enables **PORT** data.
  - Or add a **self-reference** rule (“this security group”) so any instance using the SG can talk to any other on all TCP ports.
- Keep SSH 22 from your public IP.

**Start the server (on Server VM):**
```bash
java -jar ftpd.jar --serverIp 172.31.25.252 --serverPort 1099
```

**Start the client (on Client VM):**
```bash
java -jar ftp.jar --serverAddr 172.31.25.252 --serverPort 1099 --clientIp 172.31.24.120
```

> **Tip:**  
> `--serverIp` chooses which interface the **server** binds for **PASV**.  
> `--clientIp` chooses which interface the **client** binds for **PORT** (must be reachable by the server).

---

## ⌨️ Client CLI — Commands

| Command  | Usage            | Description |
|---------:|------------------|-------------|
| `help`   | `help`           | Show commands. |
| `pwd`    | `pwd`            | Print server’s current working directory. |
| `cd`     | `cd dir`         | Change server directory (`..` up, `.` stay). |
| `dir`    | `dir`            | List files/dirs **on server** (within `pwd`). |
| `ldir`   | `ldir`           | List files in the **client’s** current folder. |
| `pasv`   | `pasv`           | **Passive**: server will listen for data; client will connect. |
| `port`   | `port`           | **Active**: client will listen for data; server will connect back. |
| `get`    | `get filename`   | Download file from server → client. |
| `put`    | `put filename`   | Upload file from client → server. |
| `quit`   | `quit`           | Exit the client. |

**Examples**

Passive (firewall/NAT-friendly):
```txt
ftp> pasv
ftp> cd root
ftp> dir
ftp> get hello.txt
ftp> put upload.bin
```

Active:
```txt
ftp> port
ftp> get big.iso
ftp> put data.tar
```

> **Gotcha:** `get`/`put` expect exactly **two tokens**. Run `pasv`/`port` on a separate line first.

---

## ⚙️ Runtime Flags

- **Server**
  - `--serverIp <IP>`: interface to bind passive data sockets (e.g., `127.0.0.1`, `172.31.x.x`, or `0.0.0.0`).
  - `--serverPort <port>`: RMI registry port (default in properties).
- **Client**
  - `--serverAddr <host/IP>`: where to find the server’s RMI registry (and the host to dial in PASV).
  - `--serverPort <port>`: RMI registry port.
  - `--clientIp <IP>`: interface to bind the client’s active-mode listener (PORT). Use client’s VPC IP or `0.0.0.0`.

---

## ✅ Testing Checklist (what to show in videos)

1. **Navigation:** `pwd`, `cd` into nested dirs, `cd ..` back up, `dir` at each level.  
2. **Passive transfers:** `pasv`, `get`, `put`, verify with `ldir` and server listing.  
3. **Active transfers:** `port`, `get`, `put` round-trip.  
4. **Security groups:** show inbound rules and explain:  
   - PASV: client connects to **server** data port (allow inbound on **server**).  
   - PORT: server connects to **client** data port (allow inbound on **client**).  
5. **Integrity (optional):**
   ```bash
   # run on both ends for a transferred file
   shasum hello.txt      # macOS
   # or
   sha1sum hello.txt     # Linux
   ```

---

## 📦 Submission Packaging (exact format)

1. Build clean jars:
   ```bash
   mvn clean package
   ```
2. Create a submission folder **named after you** (e.g., `Aayush_Rajesh_Jadhav`) and put everything inside it.
3. Zip **all sources** (exclude `target/`, `.git/`, IDE files) into `src.zip` and place in that folder:
   ```bash
   zip -r src.zip . -x "*/target/*" ".git/*" ".idea/*"
   mv src.zip <Your_Name_Folder>/
   ```
4. Copy jars:
   ```bash
   cp ftpclient/target/ftp.jar  <Your_Name_Folder>/
   cp ftpserver/target/ftpd.jar <Your_Name_Folder>/
   ```
5. Add required **videos** (`01-dir-nav.mp4`, `02-passive.mp4`, `03-active.mp4`, `04-security-groups.mp4`) and **Rubric.pdf**.  
   (Optional) Add `Test_Notes.txt` describing what each video shows + IPs used.
6. Create the **outer** zip named after you that contains the folder named after you:
   ```bash
   cd <parent-of-Your_Name_Folder>
   zip -r Your_Name.zip Your_Name_Folder
   ```

---

## 🧰 Troubleshooting Fast

- **“GET/PUT: No mode set”** → run `pasv` or `port` first.
- **Active mode `ConnectException: Connection refused`**  
  Client listening on wrong interface. Start client with:
  ```bash
  --clientIp <client-private-ip>   # or 0.0.0.0
  ```
  Then `port` again and retry.
- **Passive mode connection refused**  
  Start server with:
  ```bash
  --serverIp <server-private-ip>   # or 0.0.0.0
  ```
  Ensure SG allows inbound from client to server.
- **Typed `get hello.txt pasv`** → won’t run; `pasv` must be its own command first.

---

## 🗒️ Example Sessions

**Local / Passive**
```txt
$ java -jar ftpd.jar --serverIp 127.0.0.1
$ java -jar ftp.jar  --serverAddr localhost --clientIp 127.0.0.1
ftp> pasv
ftp> cd root
ftp> dir
ftp> get hello.txt
ftp> ldir
```

**EC2 / Active**
```txt
# on server VM
$ java -jar ftpd.jar --serverIp 172.31.25.252 --serverPort 1099

# on client VM
$ java -jar ftp.jar --serverAddr 172.31.25.252 --serverPort 1099 --clientIp 172.31.24.120
ftp> port
ftp> get server.log
ftp> put upload.bin
```

---

## License

For course use (CS549).
