## Process

A process is a running instance of a program or a task on an operating system.

It consists a chunk of memory which is allocated to process by a kernel, and it consists 4 main separate regions.

- The code (list of instructions) which process is following.
- Data (any global variables it requires during the task)
- heap (allocated at runtime for the task to run, grows up as code runs and save the processes' data)
- stack (required for function calls, local variables, grows down as code runs)

> Note: both stack and heap increase in size the "grows up" - "grows down" is just used to refer the direction of the increase, look below diagram for better visualization.

```
High addresses
┌─────────────────────┐
│       STACK         │  ← starts here, grows DOWN
│         ↓           │     (function calls push frames downward)
│                     │
│                     │
│      (unused        │
│       space)        │
│                     │
│                     │
│         ↑           │
│       HEAP          │  ← grows UP
│                     │     (malloc/new extends upward)
├─────────────────────┤
│       DATA          │  global variables
├─────────────────────┤
│       CODE          │  your program instructions
└─────────────────────┘
Low addresses
```

Additionally, it stores metadata such as PID, UID, PPID, and the [**file descriptor**](#file-descriptor) table — the list of open files (and other I/O streams) the process is using.

Those 4 separate regions are called **Virtual Memory** (_each process sees their own private address and can't directly access the RAM_) in the terms of operating system and since Heap should grow upward and stack should grow downward towards the memory, so in theory they will never crash into each other. But, even if they did due to any circumstances then the kernel will just kill that process.

### Threads

In the background each process will have one or more Thread as per the requirement. And each Thread will have its own stack. Everything else will be shared such as code, data, heap, open files, and metadata.

A **Thread** is a lightweight part of a process which has its own stack and execution context, while sharing the process address space (code, data, heap, and open files).

### Signals

When we press `CTRL + C` the kernel sends a `signal` (A **signal** is a small numbered message kernel deliver to a process).

Specifically `SIGINT` a signal interrupt, number - 2.

By default, it accepts it and terminates the execution.

But here is the other outcomes that are possible.

1. Default - accept and terminate
2. Catch - run custom code (such as any cleanup code)
3. Ignore

Also, these are some other signals which the kernel can pass to a running process. Each will have a number and kernel pass that number to a process not a word, the word is just for humans to read it and understand easily.

1. `SIGINT` (2) - `CTRL+C` polite and catchable.
2. `SIGTERM` (15) - polite and also catchable defaults to a `kill` command.
3. `SIGKILL` (9) - force kill command, uncatchable.
4. `SIGSEGV` (11) - segmentation fault, touched the memory you shouldn't have.
5. `SIGHUP` (1) - terminal hung up, often repurposed to mean **reload the config**.

### Pitfalls - A Process and some common issues.

- If a process is consisting a recursive function call which is not returning anything will cause a stack to overflow due to at runtime kernel allocate a specific amount of stack space (about **~8MB** in the case of Linux) at runtime and since the function call is not returning anything it will never be popped from the stack and each function call will occupy a new stack frame, and it consists of a detail of the function, local variables, etc.

- If both Threads are updating the same value (in this case let's think of a global variable) this can cause in a data corruption. As updating a global variable might sound one operation but under the hood its 3 at the CPU level.
  - Read the value from the memory into a register.
  - Update the value in the register
  - Write the updated value from the register to the memory.

  > A **register** is a tiny piece of storage inside the CPU unit, it's not a RAM it's a part of a CPU chip. A Typical CPU might have a 16-32 general purpose registers.

  So if there are two Threads `A & B`. And if Thread `A` read the value from the memory (e.g. `5`) and at the same time Thread `B` completes the all 3 steps (read, add, write) then Thread `A` complete its step 2 and 3. then the value will be `6` even though we updated the value twice. This will result in a **race condition**.

  > A **race condition** is a phenomenon which says the correctness of the running program will depend on which thread will run first - and the timing is not always guaranteed.

  To fix the race condition we can use lock/mutex which makes those 3 steps **atomic**.

  > A **mutex** a.k.a. mutual exclusion is a memory flag which says "_I am currently in use_".

  mutex can be implemented by a special CPU instructions that are atomic at hardware level (e.g. `compare-and-swap`)

  So before Thread touches any shared value it has to acquire the mutex. And if another Thread wants to use it has to wait until the first Thread release it.

## File Descriptor

A **File Descriptor** is just a number which a process use to refer to an open file. In simple words it's just a connection ID used by a process to locate an open file. In the case of Linux a file descriptor is nothing but anything that is related to the I/O. Here are some of the examples of it sockets, pipes, terminal, devices.

These are the three standard file descriptors which every process begin with.

1. `STDIN` (0) - where the input come
2. `STDOUT` (1) - where the output come
3. `STDERR` (2) - where the error go

Even though both `STDOUT` and `STDERR` both end up in the same terminal window they are separated at OS level, it's a design choice.

Because when we save the output to a file we have to separate them normal logs go in one file and error log go in other one. If both of them are `STDOUT` they both will end up in the same file, and we can't separate them at the OS level, and we have to parse the text and take a guess to find which line is normal log and which is error log.

The reason that this split exists is that the tools (monitoring, logging, etc.) can route them independently without parsing.

### Pitfalls - of File Descriptor and some not so normal functionality.

- If we run `2> err.log` alone in `zsh`, the shell treats it as an incomplete command and enters multi-line input mode — the cursor moves to a new line, but no fresh prompt appears. It looks "stuck" but it isn't. An empty `err.log` file is created on the way in.

  To cancel and return to a fresh prompt, press `CTRL+C` (which sends `SIGINT` to the shell).

  > **Why this happens**: redirection (`>`, `2>`, `<`) is a modifier attached to a command, not a command itself. With no command to attach to, the shell waits for the rest of the input.

  The **redirection** does **not** persist to the next command — each command line is parsed independently.

  > A **Redirection** tell a shell to send a commands input or output somewhere other than the default.

  By Default:
  - A command read input from our KBD.
  - A command write output to our terminal (fd-1).
  - A command write error to our terminal (fd-2).

  Redirection is used to change these defaults.

  The Operators for redirect:
  - `>` — send stdout to a file (overwrite)
  - `>>` - send stdout to a file (append) - recommended for the logs.
  - `2>` — send stderr to a file
  - `<` — read stdin from a file
  - `2>&1` — send stderr to wherever stdout is going

- Commands like this have a problem `./prog > out.log 2>&1 | grep DATABASE` - what problem exactly?

  In this command, `>` already redirects fd-1 (`STDOUT`) to `out.log`. Then `2>&1` sends fd-2 (`STDERR`) to the same place. The pipe reads from fd-1, but fd-1 is no longer the pipe input, so `grep` receives nothing.

  Use `tee` when you want both file logging and a pipe consumer:

  ```bash
  ./prog 2>&1 | tee out.log | grep DATABASE
  ```

## Permissions

A Linux **Permission** is used to control who can read, write, and execute the file and directories.

The `ls -la` command will show output something like below.

```bash
@rudr ➜ ~ ls -la
drwxr-xr-x@    - rudr 15 May 14:57  .aws
drwxr-xr-x@    - rudr 15 May 13:38  .bun
drwx------@    - rudr 23 May 19:24  .cache
drwxr-xr-x@    - rudr 23 May 23:01  .claude
drwxr-xr-x@    - rudr 23 May 21:06  .config
drwxr-xr-x@    - rudr 14 May 22:55  Desktop
drwxr-xr-x@    - rudr 15 May 14:50 󰲂 Documents
drwxr-xr-x@    - rudr 22 May 19:02 󰉍 Downloads
drwxr-xr-x@    - rudr 29 Apr 19:48  fex64
drwxr-xr-x@    - rudr 25 Apr 17:29  micro
drwxr-xr-x@    - rudr 25 Apr 10:07 󱍙 Music
drwxr-xr-x@    - rudr 25 Apr 13:07  Notes
drwxr-xr-x@    - rudr  8 May 13:03 󰉏 Pictures
drwxr-xr-x@    - rudr  7 May 17:22  Postman
drwxr-xr-x@    - rudr 15 May 13:36  proyecto
drwxr-xr-x@    - rudr 25 Apr 10:07  Public
.rw-r--r--@ 7.0k rudr  5 May 22:29  .hyper.js
.rw-r--r--@  475 rudr  5 May 19:33  .nvidia-settings-rc
.rw-r--r--@    7 rudr 25 Apr 14:36  .python_history
.rw-r--r--@  52k rudr 21 May 12:16  .zcompdump
.rw-------@ 120k rudr 24 May 19:07 󱆃 .zsh_history
.rw-r--r--@ 8.9k rudr 15 May 13:37 󱆃 .zshrc
@rudr ➜ ~
```

Where the `drwx--x--x@` is the permission indicator for owner - group - other, and we can read it this way.

![Permissions](https://miro.medium.com/0*arXYnZrNR4cMVpwE.png)

So the first letter always say the type of the file. (these are the most common ones there are some other special ones too, but for now these are the main ones)

- `d` indicate DIR
- `-` indicate normal file
- `.` indicate normal file in `lsd` output (tool-specific styling); in standard `ls` this is shown as `-`

And then the owner, group, other (each has its own 3 letter) - `drwx--x--x@`

- `rwx` - so owner has all permissions
- `--x` - group has execute permission
- `--x` - other has execute permission

`@` the last letter indicates that the DIR has the extended attributes or security permissions.

These letters also can be represented in a numeric format.
![Numeric permission notation](https://tbhaxor.com/content/images/2021/08/image-37.png)

To change both user and group we can use `chown`.

`sudo chown user:group file/dir`

Normally when we run a program, it runs as a user. The UID is attached to the process. In the case where user run a program that tries to write on a root only file it fails. Because the process is bind to the user (you) and user don't have the permission. _setuid changes that rule for the one specific program_.

In simple words the setuid runs the program as the owner of the file not as you.

e.g. the `passwd` has the owner root
so when we run it, it has the owner root not the user.

### Pitfalls - About Permissions and security risks.

- The difference of the permission based on the context (File or DIR).

  |     | File            | Directory                          |
  | --- | --------------- | ---------------------------------- |
  | r   | read contents   | list filenames                     |
  | w   | modify contents | create/delete/rename files inside  |
  | x   | run as program  | enter (cd) and access files inside |

- Running commands with `sudo` should be done deliberately — it bypasses normal permission checks by running as root, so a bug or malicious command has full system access.

- If a setuid-root program has a bug, an attacker can use that bug to run their own code as root — without ever logging in as root.

## systemd (system daemon)

A **Daemon** or **System Daemon** on Linux is a background program that runs continuously without a GUI or direct user interaction. In simple words system daemon a program that run as a `PID 1` after kernel boots. Its main job is to start, stop and monitor any background services on the machine.

So it just starts any background service that is required such as Bluetooth, Wi-Fi, audio, ssh, docker, etc. at boot and restart them if they crash.

We can use `systemctl` command line tool to talk with systemd.

```bash
@rudr ➜ ~ systemctl status docker

● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: disabled)
    Drop-In: /usr/lib/systemd/system/service.d
             └─10-timeout-abort.conf
     Active: active (running) since Sun 2026-05-24 18:55:28 EDT; 1h 32min ago
 Invocation: 11a7988247d844169f56217d59119928
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 2007 (dockerd)
      Tasks: 21
     Memory: 87.5M (peak: 152.2M, swap: 4.2M, swap peak: 15.2M)
        CPU: 1.340s
     CGroup: /system.slice/docker.service
             └─2007 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --selinux-enabled --userland-proxy-path /usr/bin>

May 24 18:55:28 fedora dockerd[2007]: time="2026-05-24T18:55:28.085186247-04:00" level=info msg="Deleting nftables IPv4 rules" error="exit >
May 24 18:55:28 fedora dockerd[2007]: time="2026-05-24T18:55:28.109397079-04:00" level=info msg="Deleting nftables IPv6 rules" error="exit >
May 24 18:55:28 fedora dockerd[2007]: time="2026-05-24T18:55:28.112447981-04:00" level=info msg="Firewalld: docker zone already exists, ret>
May 24 18:55:28 fedora dockerd[2007]: time="2026-05-24T18:55:28.868899908-04:00" level=info msg="Loading containers: done."
May 24 18:55:28 fedora dockerd[2007]: time="2026-05-24T18:55:28.880497805-04:00" level=info msg="Docker daemon" commit=1.fc44 containerd-sn>
May 24 18:55:28 fedora dockerd[2007]: time="2026-05-24T18:55:28.880627003-04:00" level=info msg="Initializing buildkit"
May 24 18:55:28 fedora dockerd[2007]: time="2026-05-24T18:55:28.901679624-04:00" level=info msg="Completed buildkit initialization"
May 24 18:55:28 fedora dockerd[2007]: time="2026-05-24T18:55:28.911421394-04:00" level=info msg="Daemon has completed initialization"
May 24 18:55:28 fedora dockerd[2007]: time="2026-05-24T18:55:28.911539273-04:00" level=info msg="API listen on /run/docker.sock"
May 24 18:55:28 fedora systemd[1]: Started docker.service - Docker Application Container Engine.
lines 1-25/25 (END)
```

Here are some basic systemctl commands we can use to manage systemd.

- `systemctl start/stop <service>`
- `systemctl enable/disable <service>`
- `systemctl status <service>`

### journalctl

Then comes the logs for the systemd - when something goes wrong or unexpected, we have to read log to monitor the situation.

And that's where we use `journalctl` - it's a central place by systemd to manage the logs called **journal**. In the old days every system used to write their logs in their own file at `/var/log/`.

common usage:

```bash
journalctl -u docker            # all logs for docker, ever
journalctl -u docker -f         # follow live (like tail -f)
journalctl -u docker -n 50      # last 50 lines
journalctl -u docker --since "1 hour ago"
```

### Pitfalls

- If we start the service but never enable it then it will not run on system boot, to run a service on a system boot we have to enable the service so it can run on the boot after the `PID 1`.

- Typical clean debug flow when a service is broken:
  1. `systemctl status <service>`
  2. `journalctl -u <service> -n 50`
  3. `journalctl -u <service> -f`

## CRON

A **cron** is a daemon (systemd start it at boot) that reads schedule files called crontab and runs command at specific times.

Each user has their own crontab assigned. we can edit it with these commands:

```bash
crontab -e        # edit your crontab
crontab -l        # list current entries
```

Each crontab has these 6 fields:

```
<minute> <hour> <day-of-month> <month> <day-of-week>  <command>
```

So to trigger a backup at 2AM we should write a cron like this.

```
0 2 * * *  /usr/local/bin/backup-chalkdust.sh
```

Another example:

```
*/15 9-17 * * 1-5  /usr/local/bin/healthcheck.sh
```

It reads like this:
every 15 minutes, between 9 AM and 5 PM, on weekdays (mon-fri), run the health check script. Classic "business hours monitoring" pattern.

### logs

We can find most of the system logs at this place - `/var/log/`

- boot.log — messages from the boot process (services that started/failed at boot)
- dnf.log — package manager activity (installs, updates, removals — Fedora-specific; Ubuntu calls it apt/history.log)
- syslog or messages — general system log (everything from the kernel and various services)
- auth.log or secure — authentication events (logins, sudo usage, ssh attempts) — important for security
- kern.log — kernel-specific messages

### Pitfalls - some caveats of cron & logs

- When we run any command in cron it don't have a full setup it runs on a minimal setup so we have to give it a full command to run a command and shortcuts like `python3` won't work. We have to always write a full path - `/usr/bin/python3 /home/rudra/script.py`

- If cron runs any command and gets anything (output/error) it tries to email the user but most systems are not setup for the email, so output gets lost. so to fix that we can save the output for ourselves.

  ```bash
  0 2 * * *  /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
  ```

  `>>` appends output. `2>&1` merges errors into the same file.

- Since, we are on modern systems. we use `journalctl` instead of this path because most systemd based system (e.g. fedora) funnels logs through `journalctl` instead of `/var/log/`.

## Networking

### DNS

A **DNS** or a _Domain Name System_ is like a phone book of the internet. It translates the human-readable domains into numerical IP addresses, so browser can locate and load the websites.

DNS Process:

```
Your laptop
   │
   │ "What's chalkdust.com?"
   ▼
Resolver (e.g. 8.8.8.8)
   │
   │ "Who handles .com?"
   ▼
Root nameservers ──→ "Ask the .com TLD servers"
   │
   │ "Who handles chalkdust.com?"
   ▼
.com TLD servers ──→ "Ask chalkdust.com's authoritative server"
   │
   │ "What's the A record for chalkdust.com?"
   ▼
Authoritative server ──→ "It's 203.0.113.45"
   │
   ▼
Resolver caches it, hands IP back to your laptop
```

Types of DNS records:

- `A Record`: Maps a domain name to an IPv4 address (e.g., 192.0.2.1).
- `AAAA Record`: Maps a domain name to a newer, longer IPv6 address.
- `CNAME Record`: Points an alias domain to the actual, canonical domain name (e.g., pointing blog.example.com to example.com).
- `MX Record`: Directs email traffic to the correct mail server.
- `TXT Record`: Stores readable text information used for security (like SPF/DKIM) or site verification.

### DNS Cache

**Resolver Cache** - The resolver itself (8.8.8.8, ISP, or whoever) saves the detail for a while. So next time if someone asks for the same detail, It answers from the cache instead of walking `root -> TLD -> authoritative` again.

**Local OS Cache** - The device also remembers the details locally so next time if same domain is asked, it doesn't even leave the machine and - the OS hands back the cached IP instantly.

**TTL** - caches can't stays forever, otherwise we can't change the DNS. so every DNS have its own assigned TTS: a fixed number of seconds It's allowed to stay cached. e.g. TTL 300 – 5 min

### TCP & UDP

The next step is opening a TCP connection once the browser get the IP. TCP works using 3 way handshakes.

1. SYN — client → server: "I want to talk"
2. SYN-ACK — server → client: "Got it, I want to talk too"
3. ACK — client → server: "Confirmed, let's go"

Before TCP can carry any data, both sides have to agree to talk. That agreement is the TCP handshake — 3 packets exchanged before any HTTP data flows.

"`SYN`" - synchronize. "`ACK`" - acknowledge.

```
Client                                    Server
  │                                          │
  │ ─────────── SYN ────────────────────────►│
  │                                          │
  │ ◄────────── SYN-ACK ─────────────────────│
  │                                          │
  │ ─────────── ACK ────────────────────────►│
  │                                          │
  │            [connection open]             │
  │                                          │
  │ ──── HTTP GET /index.html ──────────────►│
```

The distinct characteristics of **TCP & UDP**:

- **TCP** - reliable, ordered, connection-based. Guarantees every byte arrives, in order. Slower, but trustworthy.
- **UDP** - fire-and-forget. Faster, no guarantee. Used for DNS lookups, video streaming, gaming — where losing a packet is fine.

### Security

On a TCPs raw connection **TLS** (Transport Layer Security) performs its own handshake. It used to verify the server's identity by `SSL/TLS certificate` signed by a trusted `Certificate Authority (CA)`. The browser checks its signature. Then both client and server agree on an encryption key which only they both know and all the traffic flows between them is encrypted with that key.

Full sequence of the DNS to HTTP response:

```
1. DNS lookup       → get IP
2. TCP handshake    → open connection (SYN, SYN-ACK, ACK)
3. TLS handshake    → verify cert + agree on encryption key
4. HTTP request     → GET /, encrypted with the agreed key
5. HTTP response    → server sends back HTML, also encrypted
```

The server sends an SSL/TLS certificate. And it has these details (server public key, identity info, a signature from a CA)

### HTTP

Then `HTTP` sends a request to a server, here are some of the most basic types of `HTTP` requests.

- `GET` — read a resource ("give me this page")
- `POST` — submit new data ("create this user")
- `PUT` — replace data ("update this user fully")
- `DELETE` — remove data ("delete this user")

There are two parts of HTTP request one is Method and other is Path. Together - `GET /index.html`

Some other HTTP requests `PATCH`, `HEAD`, `OPTIONS`.

All HTTP requests will have some kind of the status code based on the response they will provide.

- `2xx` — Success. 200 OK is the common one. Request worked.
- `3xx` — Redirect. "What you want is somewhere else." `301` (moved permanently), `302` (temporary redirect). Browser auto-follows.
- `4xx` — Client error. You (the client) did something wrong. `404` Not Found (resource doesn't exist), `401` Unauthorized (need to log in), `403` Forbidden (logged in but not allowed), `400` Bad Request (malformed).
- `5xx` — Server error. Server broke. `500` Internal Server Error (generic crash), `502` Bad Gateway (proxy got bad response from upstream), `503` Service Unavailable (server temporarily down/overloaded).

Simple Model from server's point of view:

```
2xx - "yes"
3xx - "go there instead"
4xx - "you messed up"
5xx - "I messed up"
```

### Flow

- **What happens when you type a `URL` _(uniform resource locator)_?**

  So the first thing when someone types a URL in the browser search box is that it tries to resolve the DNS locally (browser/OS) based on the previous DNS cache. If it's not there then it asks the DNS resolver (e.g. 1.1.1.1 / 8.8.8.8). The resolver walks the chain on your behalf: it asks the root nameserver who handles the TLD. Once the TLD is found it asks the authoritative server for the IP (IPv4) of the domain. Once the IP is found the record is cached at the OS and resolver level, and that's called DNS caching.

  Once the IP is cached a TCP connection is opened. Before any data is transmitted via HTTP, TCP performs a 3-way handshake to establish the connection, known as SYN, SYN-ACK, ACK. Why TCP and not UDP is because TCP is reliable and guarantees that the packets arrive in order since it is a must for sites that packets should arrive in order otherwise the browser can't render the HTML DOM correctly and the site will be broken.

  If the site is based on HTTPS (Hyper Text Transfer Protocol Secure) then a raw TCP connection is required to have a TLS/SSL handshake. It works by identifying the server's identity using a TLS certificate signed by a trusted certificate authority (CA), and it also negotiates a session key to encrypt the data. Once the certificate is validated by the browser, data transmission is allowed. Otherwise, the browser raises a warning that the source might be compromised or a security issue.

  Then comes the HTTP request itself, after everything is validated the client sends a GET request to the server to ask for an index.html page. Every HTTP request has at minimum a method and a path - GET /index.html.

  After the client's request the server responds with the demanded resource and a status code. There are many types of status code, but they all follow this convention: 2xx, 3xx, 4xx, 5xx. 200 shows OK, 301 shows Redirect, 401 shows Unauthorized. 500 shows the server error.

  If the server responds with the correct resource asked by the client then the status code will be 200 and the browser will render the HTML. After that, the browser parses the HTML, finds `<link>`, `<script>`, and `<img>` references, and each of those can trigger another DNS/TCP/TLS/HTTP cycle, often to different domains such as CDNs, fonts, or analytics.
