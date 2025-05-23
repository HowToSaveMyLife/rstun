rstun
=====
**Original [rstun](https://github.com/neevek/rstun)**

**This adds parameter congestion_control, default is BBR, other options are COPA, COPAP(COPA PLUS), BBR2, CUBIC and NEW_RENO, SPROUT. Complete Options for rstund and rstunc are listed below.**

**The dependencies path of quinn and quinn-proto need to be changed. COPA、COPAP、SPROUT are implemented in [quinn-copa](https://github.com/HowToSaveMyLife/quinn-copa)**

A TCP/UDP tunnel over QUIC written in Rust.

rstun is a high-performance TCP/UDP tunneling solution. It leverages the [Quinn](https://github.com/quinn-rs/quinn) library for [QUIC](https://quicwg.org/) transport, ensuring efficient, low-latency bidirectional communication, secured by QUIC’s integrated TLS layer.

Key Features
------------

* Bidirectional TCP and UDP communication over a single QUIC tunnel.
* Encryption provided by QUIC’s inherent TLS layer.

Operating Modes
-----

* Inbound Tunneling (IN Mode)

  In `IN` mode, rstunc allows you to expose a local service (e.g., a web server or application) to the public internet securely through the QUIC tunnel. This mode is useful for scenarios where a service is running locally behind a NAT or firewall but needs to be accessible from outside.

* Outbound Tunneling (OUT Mode)

  In `OUT` mode, rstunc securely tunnels local outbound traffic through rstund, which then forwards it to the specified destination. This mode is commonly used to encrypt and route traffic from a local network to external servers, leveraging QUIC for enhanced performance and security.

Components
----------

rstun consists of two binaries:

* rstunc (client): Establishes and manages tunnels to the server.
* rstund (server): Accepts incoming connections and forwards TCP and UDP traffic according to the configured routing rules.

Example
=======

* Start the server

```
rstund \
  --addr 0.0.0.0:6060 \
  --tcp-upstream 8800 \
  --udp-upstream 8.8.8.8:53 \
  --password 123456 \
  --cert path/to/cert.der \
  --key path/to/key.der
```
  - `--addr` specifies the ip:port pair that the server will be listening on.
  - `--tcp-upstream` is the default TCP upstream for `OUT` mode tunneling if the client doesn't specify one. Tcp traffic from the client through the tunnel will be forwarded to this upstream.
  - `--udp-upstream` is the default UDP upstream for `OUT` mode tunneling if the client doesn't specify one. Udp traffic from the client through the tunnel will be forwarded to this upstream.
  - `--password`, password of the server, the client `rstunc` is required to send this password to successfully build a tunnel with the server.
  - `cert` and `key` are certificate and private key for the domain of the server, self-signed certificate is allowed, but you will have to connect to the server using IP address and the certificate will also be required by `rstunc` for verification in this case (see below). Anyway, getting a certificate for your domain from a trusted CA and connecting to the server using domain name is always recommended. Note `cert` and `key` are optional, if they are not specified, the domain `localhost` is assumed and a self-signed certificate is generated on the fly, but this is for test only, Man-In-The-Middle attack can occur with such setting, so make sure **it is only used for testing**!

* Start the client

```
rstunc
  --mode OUT \
  --server-addr 1.2.3.4:6060 \
  --password 123456 \
  --cert path/to/cert.der \
  --tcp-mapping 0.0.0.0:9900^8800 \
  --loglevel D
```
  - `--mode`, `OUT` for securing data from local to the server through the tunnel.
  - `--server-addr`, domain name or IP address of the server, port is always required.
  - `--password`, same as that for the server.
  - `--cert`, see explanation above for `rstund`. Note this is also optional if connecting to the server with a domain name, or the server `rstund` runs with an auto-generated self-signed certificate (see the TEST example below).
  - `--addr-mapping` is an address mapping between two `ip:port` pairs separated by the `^` character, the format is `[ip:]port^[ip:]port`, in the example above, a local port `9900` is mapped to the remote port `8800` of the `1.2.3.4` server that runs `rstund`. i.e. all traffic from the local port `9900` will be forwarded to the remote port `8800` through the tunnel. `--addr-mapping` also supports using `ANY` as the second part of the mapping for `OUT` mode tunneling, in which case, the server default will be used. For example `9900^ANY`.

* Simple TEST example

The following commands run a server, then a client that connects to the server in their simplest ways:


```
# Remote: run the server with auto-generated self-signed certificate
rstund -a 9000 -p 1234

# Local: connect to the server (127.0.0.1:9000) and bind both the TCP and UDP port 9900 to remote TCP and UDP port 8800
rstunc -m OUT -a 127.0.0.1:9000 -p 1234 -t 0.0.0.0:9900^8800 -u 0.0.0.0:9900^8800
```

Complete Options for `rstund` and `rstunc`
---

```
Usage: rstund [OPTIONS] --addr <ADDR> --password <PASSWORD>

Options:
  -a, --addr <ADDR>
          Address ([ip:]port pair) to listen on [default: ]
  -t, --tcp-upstream <TCP_UPSTREAM>
          The default TCP upstream for TunnelOut connections, format: [ip:]port [default: ]
  -u, --udp-upstream <UDP_UPSTREAM>
          The default UDP upstream for TunnelOut connections, format: [ip:]port [default: ]
  -p, --password <PASSWORD>
          Password of the tunnel server
  -c, --cert <CERT>
          Path to the certificate file, if empty, a self-signed certificate
          with the domain "localhost" will be used [default: ]
  -k, --key <KEY>
          Path to the key file, can be empty if no cert is provided [default: ]
  -w, --workers <WORKERS>
          Threads to run async tasks [default: 0]
      --quic-timeout-ms <QUIC_TIMEOUT_MS>
          Quic idle timeout in milliseconds for the connection [default: 40000]
      --tcp-timeout-ms <TCP_TIMEOUT_MS>
          Tcp idle timeout in milliseconds for the connection [default: 30000]
      --udp-timeout-ms <UDP_TIMEOUT_MS>
          Udp idle timeout in milliseconds for the connection [default: 30000]
  -l, --loglevel <LOGLEVEL>
          [default: I] [possible values: T, D, I, W, E]
  -g, --congestion-control <CONGESTION_CONTROL>
          Congestion control algorithm [default: BBR] [possible values: BBR, CUBIC, NEW_RENO, BBR2, COPA, COPAP, SPROUT]
  -h, --help
          Print help
  -V, --version
          Print version
```


```
Usage: rstunc [OPTIONS] --mode <MODE> --server-addr <SERVER_ADDR> --password <PASSWORD>

Options:
  -m, --mode <MODE>
          Create a tunnel running in IN or OUT mode [possible values: IN, OUT]
  -a, --server-addr <SERVER_ADDR>
          Address (<domain:ip>[:port] pair) of rstund, default port is 3515
  -p, --password <PASSWORD>
          Password to connect with rstund
  -t, --tcp-mapping <TCP_MAPPING>
          LOCAL and REMOTE mapping in [ip:]port^[ip:]port format, e.g. 8080^0.0.0.0:9090
          `8000^ANY` for not explicitly specifying the upstream on the server, the server
                     decides that port, so it depends on that the server is started with
                     explicitly setting the `--tcp-upstream` option. [default: ]
  -u, --udp-mapping <UDP_MAPPING>
          LOCAL and REMOTE mapping in [ip:]port^[ip:]port format, e.g. 8080^0.0.0.0:9090
          `8000^ANY` for not explicitly specifying the upstream on the server, the server
                     decides that port, so it depends on that the server is started with
                     explicitly setting the `--udp-upstream` option. [default: ]
  -c, --cert <CERT>
          Path to the certificate file, only needed for self signed certificate [default: ]
  -e, --cipher <CIPHER>
          Preferred cipher suite [default: chacha20-poly1305] [possible values: chacha20-poly1305, aes-256-gcm, aes-128-gcm]
  -w, --workers <WORKERS>
          Workers to run async tasks [default: 0]
  -r, --wait-before-retry-ms <WAIT_BEFORE_RETRY_MS>
          Wait time in milliseconds before trying [default: 5000]
      --quic-timeout-ms <QUIC_TIMEOUT_MS>
          Quic idle timeout in milliseconds for the connection [default: 30000]
      --tcp-timeout-ms <TCP_TIMEOUT_MS>
          Tcp idle timeout in milliseconds for the connection [default: 30000]
      --udp-timeout-ms <UDP_TIMEOUT_MS>
          Udp idle timeout in milliseconds for the connection [default: 5000]
      --dot <DOT>
          Comma separated DoT servers (domains) used to resolve the server address (domain)
          e.g. "dns.google,one.one.one.one"
          Note that DoT servers will be resolved using any available system DNS [default: ]
      --dns <DNS>
          Comma separated DNS' (IPs) used to resolve the server address (domain)
          Note that the --dot option if not empty takes precedence over this option
          e.g. "1.1.1.1,8.8.8.8" [default: ]
  -l, --loglevel <LOGLEVEL>
          Log level [default: I] [possible values: T, D, I, W, E]
  -g, --congestion-control <CONGESTION_CONTROL>
          Congestion control algorithm [default: BBR] [possible values: BBR, CUBIC, NEW_RENO, BBR2, COPA, COPAP, SPROUT]
  -h, --help
          Print help
  -V, --version
          Print version
```

License
-------

This Source Code Form is subject to the terms of the Mozilla Public
License, v. 2.0. If a copy of the MPL was not distributed with this
file, You can obtain one at http://mozilla.org/MPL/2.0/.
