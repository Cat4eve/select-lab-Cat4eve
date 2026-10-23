
# Select Lab — TCP Server with select()

## Overview

This lab demonstrates how to use the `select()` system call in C to handle multiple TCP client connections within a single-threaded server.

The server listens on `127.0.0.1:12345`, accepts multiple clients, receives XOR-encrypted messages, decrypts them, and handles graceful shutdown using SIGINT (Ctrl+C).

The assignment contains four tasks, called "dragons", which have been addressed in `server.c`.

## Project Structure

```text
select-lab/
├── server.c
├── client.c
├── makefile
└── readme.md
```

- `server.c` — TCP server using select().
- `client.c` — Client that sends XOR-encrypted messages.
- `makefile` — Compiles the server and client.
- `readme.md` — Project instructions and testing documentation.

## The Four Dragons

### Dragon #1 — TCP Socket

The original code incorrectly specified protocol 111:

```c
listen_fd = socket(AF_INET, SOCK_STREAM, 111);
```

Fixed by using protocol 0:

```c
listen_fd = socket(AF_INET, SOCK_STREAM, 0);
```

This allows the operating system to select TCP as the default protocol for `SOCK_STREAM`.

### Dragon #2 — Port Number

The original server assigned the port directly:

```c
server_addr.sin_port = PORT;
```

Fixed using `htons()`:

```c
server_addr.sin_port = htons(PORT);
```

The `htons()` function converts the port number from host byte order to network byte order.

The server listens on:

```text
127.0.0.1:12345
```

### Dragon #3 — XOR Decryption

The client encrypts messages using XOR with key 42.

The server must decrypt the received bytes using the same key.

Defined:

```c
#define XOR_KEY 42
```

Decryption implementation:

```c
for (ssize_t j = 0; j < bytes_read; j++) {
    buffer[j] ^= XOR_KEY;
}

printf("Decrypted data: ");
fwrite(buffer, 1, bytes_read, stdout);
printf("\n");
```

XOR encryption is reversible:

```text
Encrypted = Original XOR 42
Original  = Encrypted XOR 42
```

`fwrite()` is used because network data is not necessarily a null-terminated C string.

### Dragon #4 — SIGINT and Graceful Shutdown

The server handles Ctrl+C using a SIGINT signal handler.

The implementation:

1. Registers a SIGINT handler using `sigaction()`.
2. Uses a `volatile sig_atomic_t` flag to request shutdown.
3. Closes the listening socket in the signal handler.
4. Handles interrupted `select()` calls, including `EINTR`.
5. Stops the main loop when shutdown is requested.
6. Closes all remaining client sockets during cleanup.
7. Avoids deliberately closing the listening socket twice.
8. Returns control to the shell.

The handler does not use `printf()` because it is not async-signal-safe.

## Understanding select()

The `select()` system call allows one server process to monitor multiple socket descriptors simultaneously.

The server:

1. Creates a TCP listening socket.
2. Binds the socket to port 12345.
3. Starts listening for connections.
4. Initializes a file descriptor set using `FD_ZERO()`.
5. Adds sockets to the set using `FD_SET()`.
6. Calls `select()` to wait for socket activity.
7. Uses `FD_ISSET()` to identify ready sockets.
8. Accepts new connections or receives client data.
9. Repeats until shutdown is requested.

This allows multiple clients to communicate with one server without creating a separate process or thread for each client.

## Compilation

### Requirements

- macOS or Linux
- C compiler (`cc`, GCC, or Clang)
- Make
- Terminal

### Compile

Run:

```bash
make
```

This builds two executables:

```text
server
client
```

## Running the Server

Open Terminal 1:

```bash
./server
```

Expected output:

```text
Server listening on 127.0.0.1:12345
```

Keep this terminal open.

## Connecting Clients

Open Terminal 2:

```bash
./client
```

Enter:

```text
Hello from client 1
```

Open Terminal 3:

```bash
./client
```

Enter:

```text
Hello from client 2
```

Open Terminal 4:

```bash
./client
```

Enter:

```text
Hello from client 3
```

The server should accept all three connections and print the decrypted messages.

Example output:

```text
New connection accepted (fd=4)
New connection accepted (fd=5)
New connection accepted (fd=6)

Decrypted data: Hello from client 1
Decrypted data: Hello from client 2
Decrypted data: Hello from client 3
```

File descriptor numbers may vary.

## Testing SIGINT (Ctrl+C)

To test graceful shutdown, use `nc` (netcat) to open connections that can remain idle.

### Step 1 — Start the server

```bash
./server
```

### Step 2 — Connect three clients

In three separate terminals, run:

```bash
nc 127.0.0.1 12345
```

Leave all connections open and idle.

### Step 3 — Stop the server

Go to the server terminal and press:

```text
Control + C
```

Expected server output:

```text
Server: SIGINT caught; sockets closed, exiting
```

### Step 4 — Verify cleanup

Check that:

- The server returns to the shell.
- The connected clients are disconnected.
- The listening socket is closed.
- The server can be started again on the same port.

### Step 5 — Restart

```bash
./server
```

The server should successfully listen on port 12345 again.

## Troubleshooting

### Error: bind: Address already in use

Another process may be using port 12345.

Check:

```bash
lsof -nP -iTCP:12345
```

Find the process ID (PID) of the old server.

Stop it using:

```bash
kill PID
```

Replace `PID` with the actual process number.

Then restart:

```bash
./server
```

### Error: Connection refused

This usually means no server is listening on the specified address and port.

Start the server:

```bash
./server
```

Then connect:

```bash
nc 127.0.0.1 12345
```

### Compilation Problems

Rebuild using:

```bash
make
```

Check compiler errors and correct them before testing.

## Testing Checklist

Complete the following checks before submission:

- [ ] Server compiles successfully.
- [ ] Client compiles successfully.
- [ ] Server listens on port 12345.
- [ ] Multiple clients connect successfully.
- [ ] XOR-encrypted messages are decrypted correctly.
- [ ] SIGINT is handled correctly.
- [ ] The server exits when Ctrl+C is pressed.
- [ ] Connected clients are disconnected.
- [ ] All sockets are cleaned up.
- [ ] The server can restart on the same port.

## Why Graceful Shutdown Matters

Although the operating system closes file descriptors when a process terminates, explicit shutdown is useful because it:

- Allows controlled resource cleanup.
- Makes termination behavior predictable.
- Helps prevent resource-management bugs.
- Provides an opportunity to log shutdown events.
- Makes the server easier to maintain.

## Submission

The required deliverables are:

- Updated `server.c` containing solutions for all four dragons.
- Original `client.c`.
- Original `makefile`.
- Updated `readme.md` documenting the implementation and tests.

To submit changes to GitHub:

```bash
git status
git add server.c readme.md
git commit -m "Solve all four select lab dragons"
git push origin main
```

## Conclusion

This lab demonstrates how to implement a single-threaded TCP server that supports multiple concurrent client connections using `select()`.

The four tasks cover TCP socket initialization, network byte order, XOR decryption, and graceful shutdown with SIGINT.

The completed server is designed to process multiple client connections and release its socket resources on shutdown.
