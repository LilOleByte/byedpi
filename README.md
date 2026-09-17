Implementation of some DPI bypass methods.
The program is a local SOCKS proxy server.

Usage example:

```
ciadpi --disorder 1 --auto=torst --tlsrec 1+s
ciadpi  --fake -1 --md5sig
then using foxyproxy ext setup with localhost: 127.0.0.1 port

then connect after running locally

```

---

### Argument Descriptions

```
-i, --ip <ip>
    Listening IP, defaults to 0.0.0.0

-p, --port <num>
    Listening port, defaults to 1080

-D, --daemon
    Run in daemon mode
    Supported only on Linux and BSD systems

-w, --pidfile <filename>
    PID file location

-E, --transparent
    Run in transparent proxy mode; SOCKS will not function
    
-c, --max-conn <count>
    Maximum number of client connections, defaults to 512

-I,  --conn-ip <ip>
    Address to bind outgoing connections to, defaults to ::
    When an IPv4 address is specified, requests to IPv6 will be rejected

-b, --buf-size <size>
    Maximum amount of data received and sent per recv/send call
    Size is specified in bytes, defaults to 16384

-g, --def-ttl <num>
    TTL value for all outgoing connections
    Can be useful to bypass detection of non-standard/reduced TTL

-N, --no-domain
    Drop requests if a domain name is specified as the address
    Since resolution is performed synchronously, it can slow down or even freeze operation

-U, --no-udp
    Do not proxy UDP
    
-F, --tfo
    Enables TCP Fast Open
    If the server supports it, the first packet will be sent immediately along with SYN
    Supported only on Linux (4.11+)
    
-A, --auto <t,r,s,n>
    Automatic mode
    If a blocking or failure-like event occurs,
    the bypass parameters following this option will be applied
    Possible events:
        torst    : Timeout occurred or the server reset the connection after the first request
        redirect : HTTP Redirect with a Location whose domain does not match the outgoing one
        ssl_err  : ServerHello did not arrive in response to ClientHello, or SH contains an invalid session_id
        none     : Previous group was skipped, e.g., due to domain or protocol limitations
    
-L, --auto-mode <0-3>
    0: cache IP only if reconnection is possible
    1: cache IP also if:
        torst - timeout/connection reset during data exchange (i.e., after the first data from the server)
        ssl_err - only one round of data exchange completed (request-response/request-response-request)
    2: sort groups by trigger activation count, from lowest to highest
    3: both 1 and 2 simultaneously
    
-u, --cache-ttl <sec>
    Cache value time-to-live, defaults to 100800 (28 hours)
    
-y, --cache-dump <file|->
    Dump cache to a file or stdout. Format: <ip> <port> <group index> <time> <host>
    
-T, --timeout <sec>
    Timeout for waiting for the first response from the server in seconds
    On Linux, this is converted to milliseconds, so a fractional number can be specified
    
-K, --proto <t,h,u,i>
    Protocol whitelist: tls,http,udp,ipv4
    
-H, --hosts <file|:string>
    Limit the scope of parameters to a domain list
    Domains must be separated by a newline or space
    
-j, --ipset <file|:str>
    Limiter by specific IPs/subnets
    
-V, --pf <port[-portr]>
    Port limiter
    
-R, --round <num[-numr]>
    Which/what request(s) to apply obfuscation to
    Defaults to 1, i.e., the first request
    
-s, --split <pos_t>
    Split the request at the specified position
    Position format: offset[:repeats:skip][+flag1[flag2]]
    Flags:
        +s: add SNI offset
        +h: add Host offset
        +n: zero offset
    Additional flags:
        +e: end; +m: middle
    Examples: 
        0+sm - split the request in the middle of SNI
        1:3:5 - split at positions 1, 6, and 11
    The switch can be specified multiple times to split the request at multiple positions
    If the offset is negative and has no flags, the packet size is added to it
    
-d, --disorder <pos_t>
    Similar to --split, but parts are sent in reverse order
    
-o, --oob <pos_t>
    Similar to --split, but a part is sent as OOB data
    
-q, --disoob <pos_t>
    Similar to --disorder, but a part is sent as OOB data
    
-f, --fake <pos_t>
    Similar to --disorder, except a fake part is sent before the first chunk is sent
    The number of bytes sent from the fake equals the size of the split part
    ! May behave unstably on Windows
 
-t, --ttl <num>
    TTL for the fake packet, defaults to 8
    You need to choose a value such that the packet does not reach the server, but is processed by the DPI

-S, --md5sig
    Set the TCP MD5 Signature option for the fake packet
    Most servers (mainly on Linux) drop packets with this option
    Supported only on Linux; may be disabled in some kernel configurations (< 3.9, Android)

-O, --fake-offset <pos_t>
    Shift the start of the fake data
    Offsets with flags are calculated relative to the original request
       
-l, --fake-data <file|:str>
    Specify custom fake packets
    The string can contain escape characters (\n,\0,\0x10)

-e, --oob-data <char>
    Byte sent out-of-band, defaults to 'a'
    ASCII or escape character can be specified
    
-n, --fake-sni <str>
    Dynamically change SNI in the fake packet
    If the fake size is larger than the request size, the fake is reduced (Padding or ECH sizes are modified, or some extensions are removed)
    The "?" character is replaced by a random Latin letter, "#" by a digit, "*" by a letter or digit
    Can be specified multiple times; a random SNI will be chosen from the specified ones for each request
    
-Q, --fake-tls-mod <flag>
    rand - fill SessionID, Random, and KeyExchange fields with random data
    orig - use the original ClientHello as the fake
    msize=n - maximum fake size; a negative number reduces the original size by -n bytes
    
-M, --mod-http <h[,d,r]>
    Various manipulations with the HTTP packet, can be combined
    hcsmix:
        "Host: name" -> "hOsT: name"
    dcsmix:
        "Host: name" -> "Host: NaMe"
    rmspace:
        "Host: name" -> "Host:name\t"

-r, --tlsrec <pos_t>
    Split ClientHello into separate records at the specified offset
    Can be specified multiple times  

-m, --tlsminor <ver>
    Changes the third byte in the TLS record to the specified one
    
-a, --udp-fake <count>
    Number of fake UDP packets

-Y, --drop-sack
    Ignore SACK, forcing the kernel to retransmit already delivered packets
    Supported only on Linux

```

---

### In Detail

`--split`

Splits the request into parts. Example for a 30-byte request:

* Parameters: `--split 3 --split 7`
* Send order: 1-3, 3-7, 7-30

Positions should be specified in ascending order.

---

`--disorder`

The part subject to disorder will be sent with TTL=1, i.e., it will not actually be delivered anywhere.
The OS learns about this only after sending the subsequent part, when the server reports loss via SACK.
The system will have to resend the previous packet, thereby breaking the normal order.

* Parameters: `--disorder 7`
* Send order: 7-30, 1-7

The above applies only to Linux.
On Windows, retransmission starts from the position where losses began (maximum ACK received from the server):

* Parameters: `--disorder 7`
* Send order: 7-30, 1-30

Therefore, it is advisable to also use `split`:

* Parameters: `--split 7 --disorder 23`
* Send order: 1-7, 23-30, 7-30

In practice, it is optimal to use:

* Linux: `--disorder 1`
* Windows: `--split 1+s --disorder 3+s`

---

`--fake`

* Parameters: `--fake 7`
* Send order: 1-7 fake, 7-30 original, 1-7 original

The data in the first part of the request is replaced with fake data.

This part must pass through the DPI, but not reach the server.
Since the part will not reach it, the OS will send it again, thereby changing the order similarly to `disorder`.
To prevent the fake from reaching the server, the `ttl` and `md5sig` options are used.

TTL must be chosen such that the packet passes through all DPIs, but does not reach the server.

For Linux, there is `md5sig`. It sets the TCP MD5 Signature option, which prevents the packet from being accepted by many servers.
Unfortunately, `md5sig` does not work in all builds.

For Windows, there is another way to avoid server processing of the fake.
This is combining `fake` with `disorder`:

* Parameters: `--disorder 1 --fake 7`
* Send order: 2-7 fake, 7-30 original, 1-30 original

If the fake packet does reach the server, it will be overwritten due to full retransmission.

In practice, it is optimal to use:

* Linux: `--fake -1 --md5sig`
* Windows: `--disorder 1 --fake -1`

---

`--oob`

TCP can send data out-of-band using the URG flag, but only 1 byte per packet.

All data in such a packet will be delivered to the application, except for the last byte, which is out-of-band:

* Parameters: `--oob 3`
* Sending: 1-4 with the URG flag (1-3 request data + 4th byte, which will be truncated), 3-30

This byte is preferably placed in the SNI: `--oob 3+s`

---

`--disoob`

Similar to `--disorder`, but the part is sent with an OOB byte:

* Parameters: `--disoob 3`
* Sending: 3-30, 1-4 with the URG flag (1-3 request data + 4th byte, which will be truncated)

When used with `--fake` or `--disorder`, you can get a packet where the OOB byte will be located at the split point:

* Parameters: `--disoob 3 --disorder 7`
* Sending: 3-30, 1-8 with the URG flag (1-3 + byte that will be truncated + 4-8)

---

`--tlsrec`

A single TLS record can be split into multiple records by slightly modifying the header.

A new header is inserted at the split point, increasing the request size by 5 bytes.

This header can be placed in the middle of the SNI, preventing the DPI from reading it correctly:
`--tlsrec 3+s`

Although `tlsrec` and `oob` obfuscate DPI, they can also confuse middleboxes that do not support a full TCP/TLS stack.

Because of this, they should be used together with `--auto`:

`--auto=torst --timeout 3 --tlsrec 3+s`

In the example, `tlsrec` will only be applied in cases where the connection is reset or a timeout occurs, i.e., when a blockage has most likely happened.

Conversely, you can cancel `tlsrec` if the server resets the connection or drops the packet:

`--tlsrec 3+s --auto=torst --timeout 3`

---

`-Y, --drop-sack`

Forces the kernel to ignore packets with the TCP SACK extension.
This extension allows acknowledging the receipt of individual data segments.
If the first part of the request is lost and only the second reaches the server, the server can use this extension to notify the client. Then the client, knowing that the second part arrived, will send only the first.

Why ignore this extension? The second segment might be fake. If it reaches the server, but the client does not know about it, it will attempt to retransmit it. However, this segment will contain the original data, which will overwrite the fake ones, thereby preventing protocol breakage.

Since fast acknowledgment will not work, this will break `disorder` and also add a delay before retransmission (about 200ms).

---

`--auto`, `--hosts`

The `auto` parameter divides options into groups.
For each request, they are evaluated from left to right.
First, the trigger specified in `auto` is checked, then `pf`, `ipset`, `proto`, and `hosts`.

You can specify multiple option groups by separating them with this parameter.

Parameters that come below `--timeout` in the help text can be extracted into a separate group.

#### Examples:

```
--fake -1 --ttl 10 --auto=ssl_err --fake -1 --ttl 5

```

By default, use `fake` with ttl=10; in case of an error, use `fake` with ttl=5.

```
--hosts list.txt --disorder 3 --auto=none

```

Apply obfuscation only for domains from list.txt.

```
--hosts list.txt --auto=none --disorder 3

```

Do not apply obfuscation for domains from list.txt.

```
--auto=torst --hosts list.txt --disorder 3

```

Do nothing by default; use disorder provided that a block occurred and the domain is in list.txt.

```
--proto=http,tls --disorder 3 --auto=none

```

Obfuscate only HTTP and TLS.

```
--proto=http --fake -1 --fake-data=':GET /...' --auto=none --fake -1

```

Override the fake packet for HTTP.

---

### Building

Requirements for building:
`make`, `gcc/clang` for Linux, `mingw` for Windows

* Linux: `make`
* Windows: `make windows CC=x86_64-w64-mingw32-gcc`

---

### Additional DPI Information, Sources of Ideas

* [https://github.com/bol-van/zapret/blob/master/docs/readme.md](https://github.com/bol-van/zapret/blob/master/docs/readme.md)
* [https://geneva.cs.umd.edu/papers/geneva_ccs19.pdf](https://geneva.cs.umd.edu/papers/geneva_ccs19.pdf)
* [https://habr.com/ru/post/335436](https://habr.com/ru/post/335436)
