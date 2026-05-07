# WebServ

A fully functional HTTP/1.1 web server built from scratch in C++98, inspired by NGINX. This is a 42 School project implementing core web server features including multi-server virtual hosting, CGI execution, file uploads, and non-blocking I/O.

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Requirements](#requirements)
- [Building](#building)
- [Usage](#usage)
- [Configuration](#configuration)
- [CGI Support](#cgi-support)
- [Project Structure](#project-structure)
- [Authors](#authors)

---

## Features

- **HTTP/1.1** compliant request/response handling
- **Multiple virtual servers** on different IPs and ports
- **Non-blocking I/O** using `select()` for concurrent client handling
- **HTTP methods**: `GET`, `POST`, `DELETE`
- **CGI execution**: PHP, Python, Perl (`.php`, `.py`, `.pl`)
- **File uploads** with configurable upload directory
- **Auto-index** (directory listing) when no index file is found
- **Custom error pages** per server or location
- **URL redirections**
- **Keep-alive** connections with configurable timeout
- **MIME type** resolution from a `mime.types` file
- **Access and error logging**
- **Client body size limiting**
- **URI length limiting**
- **Cookie tracking**
- **Default server** selection via `default_server` tag

---

## Architecture

The server is split into four main modules:

| Module | Responsibility |
|---|---|
| `config` | Parses and validates the configuration file, builds `Server` and `Location` objects |
| `server` | Sets up sockets, runs the main `select()` event loop, manages client connections and uploads |
| `Request_Response` | Parses raw HTTP requests, builds HTTP responses, handles auto-index, CGI responses, and error pages |
| `CGI` | Forks a child process, sets environment variables, executes the CGI binary, and returns its output |

**Request lifecycle:**

```
Client → socket accept → read full request → parse (Request)
       → match server/location → build response (Response)
       → [optional CGI fork] → send response → keep-alive or close
```

---

## Requirements

- A C++98-compatible compiler (`g++` or `clang++`)
- `make`
- POSIX-compliant OS (Linux or macOS)

---

## Building

```bash
make        # compile the webserv binary
make clean  # remove object files
make fclean # remove object files, binary, upload/, session/, logs/
make re     # full rebuild
```

The compiled binary is named `webserv`.

---

## Usage

```bash
./webserv [config_file]
```

- If no config file is provided, the server falls back to `ConfigFiles/default.conf`.
- The server creates `upload/`, `session/`, and `logs/` directories automatically on first run.

**Examples:**

```bash
./webserv                           # use default config
./webserv ConfigFiles/example.conf  # use a custom config
```

---

## Configuration

The configuration syntax is inspired by NGINX. A config file contains one or more `server` blocks, each optionally containing `location` blocks.

### Server Block

```nginx
server {
    listen 127.0.0.1:8080;          # IP:port (default IP: 127.0.0.1, default port: 8080)
    listen 8081 default_server;     # mark as the default server for unmatched hosts

    server_name example.com www.example.com;

    root ./www;                     # root directory for this server
    index index.html index.php;     # index files to look for

    client_max_body_size 10m;       # body size limit (k/m/g suffix, default: 1m)
    client_max_uri 2048;            # max URI length in bytes

    autoindex on;                   # enable directory listing (default: off)

    allowed_method GET POST DELETE; # restrict accepted methods

    error_page 404 /errors/404.html;
    error_page 500 /errors/500.html;

    access_log logs/access.log;
    error_log  logs/error.log;

    include ConfigFiles/mime.types; # load MIME type mappings

    location /uploads {
        upload_pass ./upload/;      # directory where uploaded files are stored
    }

    location /cgi-bin {
        cgi_pass .php /usr/bin/php-cgi;
        cgi_pass .py  /usr/bin/python3;
    }

    location /old-path {
        redirection /new-path;      # HTTP redirect
    }
}
```

### Directive Reference

| Directive | Scope | Description |
|---|---|---|
| `listen` | server | IP, port, or IP:port. Append `default_server` to use as the fallback. |
| `server_name` | server | One or more hostnames for virtual hosting. |
| `root` | server, location | Document root path. |
| `index` | server, location | Index file(s) to serve for directory requests. |
| `autoindex` | server, location | `on`/`off` — generate a directory listing when no index is found. |
| `allowed_method` | server, location | Whitelist of accepted HTTP methods (`GET`, `POST`, `DELETE`). |
| `client_max_body_size` | server, location | Max request body size. Accepts `k`, `m`, `g` suffix (default: `1m`). |
| `client_max_uri` | location | Max URI length in bytes. |
| `error_page` | server, location | Map HTTP status codes to custom error pages. |
| `access_log` | server, location | Path to the access log file. |
| `error_log` | server, location | Path to the error log file. |
| `include` | server, location | Include an external file (e.g. `mime.types`). |
| `upload_pass` | server, location | Directory to store uploaded files. |
| `cgi_pass` | server, location | Map a file extension to a CGI interpreter binary. |
| `redirection` | server, location | Redirect requests to another path. |
| `alias` | location | Replace the location prefix with a different filesystem path. |

### `listen` Formats

```
listen 8080;                  # port only  → IP defaults to 127.0.0.1
listen 192.168.1.1;           # IP only    → port defaults to 8080
listen 127.0.0.1:8080;        # IP + port
listen 8080 default_server;   # mark as default virtual server
```

---

## CGI Support

CGI scripts are executed by forking a child process and passing environment variables following the CGI/1.1 specification. Supported interpreters are configured via `cgi_pass`:

```nginx
cgi_pass .php /usr/bin/php-cgi;
cgi_pass .py  /usr/bin/python3;
cgi_pass .pl  /usr/bin/perl;
```

- A **504 Gateway Timeout** is returned if the CGI process exceeds the timeout.
- A **500 Internal Server Error** is returned on CGI execution failure.

---

## Project Structure

```
WebServe/
├── main.cpp                        # Entry point
├── Makefile
├── include/
│   └── header.hpp                  # Central include aggregator
├── ConfigFiles/
│   ├── default.conf                # Fallback configuration
│   ├── example.conf                # Annotated example configuration
│   └── mime.types                  # MIME type mappings
└── src/
    ├── config/                     # Configuration parsing
    │   ├── include/
    │   │   ├── Parser.hpp          # Tokenizer + config analyzer
    │   │   ├── Server.hpp          # Server block representation
    │   │   ├── Location.hpp        # Location block representation
    │   │   ├── CommonDirectives.hpp # Shared directives (root, index, …)
    │   │   ├── StringExtensions.hpp # String utility helpers
    │   │   └── CustomException.hpp  # Custom exception class
    │   └── src/
    │       ├── Parser.cpp
    │       ├── Server.cpp
    │       ├── Location.cpp
    │       ├── CommonDirectives.cpp
    │       ├── StringExtensions.cpp
    │       └── CustomException.cpp
    ├── server/                     # Socket & event loop
    │   ├── include/
    │   │   ├── Wb_Server.hpp       # Main server class (select loop)
    │   │   └── Upload.hpp          # File upload handler
    │   └── src/
    │       ├── Wb_Server.cpp
    │       └── upload.cpp
    ├── CGI/                        # CGI execution
    │   ├── include/CGI.hpp
    │   └── src/CGI.cpp
    └── Request_Response/           # HTTP parsing & response building
        ├── include/
        │   ├── Request.hpp
        │   └── Response.hpp
        └── src/
            ├── Request.cpp
            ├── Response.cpp
            ├── Main_Response.cpp
            ├── AutoIndex.cpp
            ├── Cgi_Response.cpp
            ├── Client_Errors.cpp
            └── serveRequestedResource.cpp
```

---

## Authors

hdagdagu
rrhnizar
kchaouki
