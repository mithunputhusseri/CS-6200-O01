# GIOS Project 1 (mp303-pr1) — Completion Guide

Quick reference for **Warm-up → Part 1 (Getfile libraries) → Part 2 (multithreaded boss/worker)**. Socket patterns follow [Beej’s Guide to Network Programming](https://beej.us/guide/bgnet/) (especially `getaddrinfo`, TCP client/server flow, partial `send`/`recv`, and `SO_REUSEADDR`).

**Constraints (from `Readme.md`):**

- Use **pthreads** for Part 2 (client and server). **Do not use `fork()`** for Parts 1–2 concurrency.
- Test on the class VM / Ubuntu 20.04 where noted (`mtgf/*.o` binaries are 20.04-only).
- Gradescope grades your **last** submission before the deadline.

---

## Grading overview

| Component | Points | What you submit |
|-----------|--------|-----------------|
| Warm-up: Echo | 10 | `echo/echoclient.c`, `echo/echoserver.c` |
| Warm-up: Transfer | 10 | `transfer/transferclient.c`, `transfer/transferserver.c` |
| Part 1: Getfile client | 15 | `gflib/gfclient.c`, student headers |
| Part 1: Getfile server | 15 | `gflib/gfserver.c`, student headers |
| Part 2: MT client | 20 | `mtgf/gfclient_download.c`, `gf-student.*` |
| Part 2: MT server | 20 | `mtgf/handler.c`, `gfserver_main.c`, `gf-student.*` |
| README (Canvas PDF) | 10 (+5 EC) | Design / flow / testing / references — not a code walkthrough |

---

## Beej’s Guide — sections that matter for this project

| Topic | Beej chapter | Why you need it |
|-------|--------------|-----------------|
| `getaddrinfo` / `freeaddrinfo` | §5.1 | IPv4 **and** IPv6 without hand-packing `sockaddr_in` |
| `socket` → `connect` (client) | §5.2, §5.4, §6.2 | Echo, transfer, `gfc_perform` |
| `socket` → `bind` → `listen` → `accept` (server) | §5.2–5.6, §6.1 | Echo, transfer, `gfserver_serve` |
| `send` / `recv` | §5.7 | All parts; never assume one call moves all bytes |
| Partial sends | §7.4 | **Required** for transfer + Getfile body/headers |
| `SO_REUSEADDR` | §6.1 server example | Restart server on same port (Part 1 rubric) |
| `MSG_NOSIGNAL` (Linux) | common pattern | Avoid SIGPIPE on broken connections when sending |

**Client connection pattern (Beej):** `AF_UNSPEC` + `SOCK_STREAM`, loop `getaddrinfo` results until `connect` succeeds.

**Server listen pattern (Beej):** `AI_PASSIVE` for bind-to-all-interfaces, **`setsockopt(SO_REUSEADDR)` before `bind`**, then `listen`, then infinite `accept` loop.

---

## Warm-up 1: Echo client/server (`echo/`)

### Requirements

- Messages ≤ **15 bytes**; static buffer (e.g. `char buf[16]`) is OK.
- May assume one `send`/`recv` moves the full short message (or exit on partial I/O).
- **Not** null-terminated on the wire — do not rely on `strlen` for received data.
- **Client stdout:** only the echoed bytes (no extra newlines/debug).
- **Server:** loop forever; serve many clients sequentially (accept again after each).
- **IPv4 + IPv6:** use `hints.ai_family = AF_UNSPEC`.

### Snippet: client connect (IPv4/IPv6)

```c
struct addrinfo hints, *res, *p;
char portstr[6];
int sockfd = -1;

snprintf(portstr, sizeof portstr, "%u", portno);
memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;
hints.ai_socktype = SOCK_STREAM;

if (getaddrinfo(hostname, portstr, &hints, &res) != 0)
    exit(1);

for (p = res; p; p = p->ai_next) {
    sockfd = socket(p->ai_family, p->ai_socktype, p->ai_protocol);
    if (sockfd < 0) continue;
    if (connect(sockfd, p->ai_addr, p->ai_addrlen) == 0) break;
    close(sockfd);
    sockfd = -1;
}
freeaddrinfo(res);
if (sockfd < 0) exit(1);
```

### Snippet: server accept loop

```c
hints.ai_family = AF_UNSPEC;
hints.ai_socktype = SOCK_STREAM;
hints.ai_flags = AI_PASSIVE;
getaddrinfo(NULL, portstr, &hints, &res);
/* bind first working result; SO_REUSEADDR recommended */
listen(sockfd, BACKLOG);
for (;;) {
    int clientfd = accept(sockfd, (struct sockaddr *)&their_addr, &sin_size);
    /* recv from clientfd, send same bytes back, close(clientfd) */
}
```

---

## Warm-up 2: File transfer (`transfer/`)

### Protocol (implicit)

- Client connects; **sends nothing**.
- Server reads a file (path from argv), streams bytes on the socket, **closes** socket when done.
- Client saves until **`recv` returns 0** (EOF).
- Server keeps running after each transfer (like echo server).
- **Do not** send file length in a custom header — grader expects raw stream only.

### Partial I/O (from project Readme + Beej §7.4)

Never assume `send(s, buf, length, 0) == length` or that one `recv` fills the buffer.

```c
ssize_t send_all(int fd, const void *buf, size_t len) {
    const char *p = buf;
    size_t left = len;
    while (left > 0) {
        ssize_t n = send(fd, p, left, MSG_NOSIGNAL);
        if (n < 0) {
            if (errno == EINTR) continue;
            return -1;
        }
        if (n == 0) return -1;
        p += n;
        left -= (size_t)n;
    }
    return (ssize_t)len;
}
```

### Client receive-to-file loop (see working pattern in `transfer/transferclient.c`)

```c
while (1) {
    ssize_t n = recv(sockfd, buffer, sizeof buffer, 0);
    if (n == 0) break;          /* server closed — transfer complete */
    if (n < 0) { /* handle EINTR */ }
    fwrite(buffer, 1, (size_t)n, outfile);
}
```

### Server send-from-file loop

```c
int fd = open(path, O_RDONLY);
/* open with mode S_IRUSR | S_IWUSR if creating */
char buf[4096];
ssize_t n;
while ((n = read(fd, buf, sizeof buf)) > 0) {
    if (send_all(clientfd, buf, (size_t)n) < 0) break;
}
close(fd);
close(clientfd);
```

---

## Part 1: Getfile protocol (`gflib/`)

### Wire format

**Request (client → server):**

```http
GETFILE GET <path>\r\n\r\n
```

- Scheme: `GETFILE`; method: `GET`; path must start with `/`.
- Single spaces between tokens; **no** space before `\r\n\r\n`.

**Response (server → client):**

```http
GETFILE OK <length>\r\n\r\n<body>
GETFILE FILE_NOT_FOUND\r\n\r\n
GETFILE ERROR\r\n\r\n
GETFILE INVALID\r\n\r\n
```

| Status | Body | Length field |
|--------|------|--------------|
| `OK` | file bytes follow header | ASCII decimal size |
| `FILE_NOT_FOUND` | none | omitted |
| `ERROR` | none | omitted (server fault) |
| `INVALID` | none | omitted (bad/malformed/incomplete header) |

**Critical:** Protocol data is **not** C strings. Use `memcmp` / bounded copies / search for `\r\n\r\n` — not `strcmp`/`sscanf` on raw `recv` buffers unless you null-terminate a **copy** first.

**Status semantics for grading:**

- Malformed scheme/method/path → `INVALID`.
- Valid request, missing file → `FILE_NOT_FOUND` (client-side path error).
- Server cannot read/serve → `ERROR`.

Tests may deliver header + body in **one** TCP segment or **many**; your parser must accumulate until `\r\n\r\n`, then stream the body.

### API status types (do not confuse)

| Side | Header | Values |
|------|--------|--------|
| Client | `gfclient.h` | enum: `GF_OK=0`, `GF_FILE_NOT_FOUND=1`, … |
| Server | `gfserver.h` | `#define GF_OK 200`, `GF_FILE_NOT_FOUND 400`, … |

On the wire you always send **text** status names, not these integers.

### Files you implement

| File | Role |
|------|------|
| `gfclient.c` | Opaque `gfcrequest_t`; connect; send request; parse response; callbacks |
| `gfserver.c` | Opaque `gfserver_t`, `gfcontext_t`; accept loop; parse request; call handler |
| `gf-student.c` / `gf-student.h` | Shared helpers: find header end, `send_all`, `recv_all` (optional but recommended) |
| `gfclient-student.h`, `gfserver-student.h` | Student-only declarations |

Prebuilt **`handler.o`** in `gflib/` implements `gfs_handler` (uses `content_get`, `gfs_sendheader`, `gfs_send`). Your library must expose those `gfs_*` entry points to the handler.

### Client flow (`gfc_perform`)

1. TCP connect (same `getaddrinfo` pattern as warm-up).
2. `send_all`: `GETFILE GET <path>\r\n\r\n`.
3. Accumulate bytes until `\r\n\r\n` found.
4. Parse status line; if `OK`, parse ASCII length after `GETFILE OK `.
5. Call **header callback once** with full header bytes (if registered).
6. For each body chunk: invoke **write callback** (one call per internal `recv` is expected).
7. Track `gfc_get_status`, `gfc_get_filelen`, `gfc_get_bytesreceived`.
8. Return **0** if protocol completed (including `FILE_NOT_FOUND` / `ERROR`); **negative** if connection dropped or header invalid.

### Server flow (`gfserver_serve`)

1. Setup listening socket: `getaddrinfo` + `SO_REUSEADDR` + `bind` + `listen`.
2. Forever: `accept` → read/accumulate request until `\r\n\r\n` or detect incomplete/invalid.
3. If invalid → `gfs_sendheader(ctx, GF_INVALID, 0)` (or equivalent wire message) and close.
4. If valid → extract path → call registered **handler** with `gfcontext_t **`.
5. Handler (in `handler.o`) calls your:
   - `gfs_sendheader(ctx, status, file_len)`
   - `gfs_send(ctx, data, size)` — must block until `size` bytes sent (use loop)
   - `gfs_abort(ctx)` on failure
6. Close client socket after request; **keep listening socket open** (restart on same port).

### Snippet: find header terminator

```c
const char *gf_find_header_end(const char *buf, size_t len) {
    if (len < 4) return NULL;
    for (size_t i = 0; i + 3 < len; i++) {
        if (buf[i] == '\r' && buf[i+1] == '\n' &&
            buf[i+2] == '\r' && buf[i+3] == '\n')
            return &buf[i];
    }
    return NULL;
}
```

### Snippet: build response headers (`snprintf` → `send_all`)

```c
/* OK */
snprintf(hdr, sizeof hdr, "GETFILE OK %zu\r\n\r\n", file_len);

/* errors — no length */
snprintf(hdr, sizeof hdr, "GETFILE FILE_NOT_FOUND\r\n\r\n");
snprintf(hdr, sizeof hdr, "GETFILE INVALID\r\n\r\n");
snprintf(hdr, sizeof hdr, "GETFILE ERROR\r\n\r\n");
```

### Snippet: parse client response status (conceptual)

After header ends at `\r\n\r\n`, compare prefix with `memcmp`:

- `GETFILE OK ` → parse digits until `\r` → `filelen`, body starts after 4-byte marker.
- `GETFILE FILE_NOT_FOUND\r\n\r\n` → set status, no body.
- Same for `ERROR`, `INVALID`.

### Double-pointer API pattern

All setters take `gfcrequest_t **gfr` or `gfserver_t **gfs`:

```c
void gfc_set_port(gfcrequest_t **gfr, unsigned short port) {
    (*gfr)->port = port;
}
```

Define `struct gfcrequest_t` / `struct gfserver_t` / `struct gfcontext_t` **only** inside the corresponding `.c` file.

---

## Part 2: Multithreaded boss/worker (`mtgf/`)

### Architecture (required for grading)

**Server (`gfserver_main.c` + `handler.c`):**

- **Boss:** existing accept loop inside `gfserver.o` **or** your integration — main thread accepts connections and enqueues work.
- **Workers:** fixed pool sized by `-t nthreads`; each connection/request handled in a worker.
- Use **`steque_t`** work queue + **`pthread_mutex_t`** + **`pthread_cond_t`**.
- Implement **`gfs_handler`** in `handler.c` (Part 1 used `handler.o`; Part 2 you write it).
- Expose init/shutdown helpers via `extern` from `handler.c` to `gfserver_main.c`.

**Client (`gfclient_download.c`):**

- **Boss:** main enqueues `nrequests` jobs (from `-n`) onto a steque.
- **Workers:** pool of `-t` threads each run `gfc_create` … `gfc_perform` … `gfc_cleanup` for one job.
- Boss waits until all jobs complete, then signals workers to exit and joins threads.

`gfclient.o` / `gfserver.o` in `mtgf/` replace your Part 1 sources for protocol code unless you rebuild on non-20.04 platforms (then use your `gflib` implementations).

### Handler implementation sketch (`handler.c`)

Uses `content_get(path)` → fd; `fstat` for size; stream with `read` + `gfs_send`:

```c
gfh_error_t gfs_handler(gfcontext_t **ctx, const char *path, void *arg) {
    int fd = content_get(path);
    if (fd < 0) {
        gfs_sendheader(ctx, GF_FILE_NOT_FOUND, 0);
        return gfh_success;
    }
    struct stat st;
    if (fstat(fd, &st) < 0) {
        gfs_sendheader(ctx, GF_ERROR, 0);
        return gfh_failure;
    }
    if (gfs_sendheader(ctx, GF_OK, (size_t)st.st_size) < 0)
        return gfh_failure;

    char buf[4096];
    ssize_t n;
    while ((n = read(fd, buf, sizeof buf)) > 0) {
        if (gfs_send(ctx, buf, (size_t)n) < 0)
            return gfh_failure;
    }
    return gfh_success;
}
```

(`gfh_success` / `gfh_failure` are defined in `mtgf/gfserver.h`.)

### Boss/worker queue pattern (both sides)

```c
typedef struct {
    /* job fields: path, server, port, local_path, etc. */
} job_t;

steque_t queue;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int shutdown = 0;

void boss_enqueue(job_t *job) {
    pthread_mutex_lock(&mutex);
    steque_enqueue(&queue, job);
    pthread_cond_signal(&cond);
    pthread_mutex_unlock(&mutex);
}

void *worker(void *arg) {
    for (;;) {
        pthread_mutex_lock(&mutex);
        while (steque_isempty(&queue) && !shutdown)
            pthread_cond_wait(&cond, &mutex);
        if (shutdown && steque_isempty(&queue)) {
            pthread_mutex_unlock(&mutex);
            break;
        }
        job_t *job = steque_pop(&queue);
        pthread_mutex_unlock(&mutex);
        /* do work */
    }
    return NULL;
}
```

**Server variant:** boss enqueues accepted `clientfd` or a struct holding `gfcontext_t *` after accept (exact shape depends on how you hook into `gfserver.o` — follow comments in `gfserver_main.c`).

**Synchronization checklist:**

- Protect **steque** and any shared counters with one mutex (or split carefully).
- Use condition variable so workers sleep when queue empty.
- Boss sets `shutdown=1`, broadcasts `cond`, then `pthread_join`s all workers.

### Steque API (`mtgf/steque.h`)

- `steque_init`, `steque_enqueue`, `steque_pop`, `steque_isempty`, `steque_destroy`
- Items are `void *` — heap-allocate job structs.

---

## Debugging tips (from Readme)

- Exit status **11** → SIGSEGV (`kill -l` for signal list).
- Warm-up servers must **not** exit after one client.
- Ignore SIGPIPE or use `MSG_NOSIGNAL` when sending to closed peers.
- Autograder is not a substitute for local tests on the class VM.

---

## Suggested implementation order

1. Echo client/server with `getaddrinfo` (IPv4/IPv6).
2. Transfer with `send_all` + recv-until-EOF client.
3. Shared helpers in `gf-student.c` (header search, send/recv loops).
4. `gfserver.c` request parsing + response helpers, then `gfclient.c` mirror logic.
5. Test with `gflib/gfserver_main` + `gfclient_download` (single-threaded).
6. Part 2: `handler.c`, thread pool in `gfserver_main.c`, then MT `gfclient_download.c`.

---

## References to cite in your Canvas README

- [Beej’s Guide to Network Programming](https://beej.us/guide/bgnet/)
- [POSIX Threads tutorial (LLNL)](https://hpc-tutorials.llnl.gov/posix/)
- Course `Readme.md` and `readme-student-template.md`
- libcurl easy interface (client API inspiration): linked from project Readme

---

## Directory map

```
mp303-pr1/
├── echo/              Warm-up 1
├── transfer/          Warm-up 2
├── gflib/             Part 1 — implement gfclient.c, gfserver.c
├── mtgf/              Part 2 — handler.c, MT main files; steque; *.o libs
└── Readme.md          Full assignment specification
```
