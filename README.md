# OpenSSL Server/Client Example

This repository demonstrates OpenSSL TLS communication between a server and client, focusing on understanding the TLS record structure and how payloads are transmitted.

## Setup

### Generate Certificates
```bash
./generate-certs.sh
```
This generates:
- CA certificate and key
- Server certificate and key (using secp384r1 curve)
- Client certificate and key (using secp384r1 curve)

### Run Server
```bash
./server.sh
```
Starts an OpenSSL server on port 4443 with debug and message logging enabled.

### Run Client
```bash
./client.sh
```
Connects to the server with client certificate authentication.

## TLS Record Structure

### Record Header (5 bytes)
```
Byte 0:    Content Type (0x17 = Application Data)
Bytes 1-2: Protocol Version (0x03 0x03 = TLS 1.2)
Bytes 3-4: Payload Length (big-endian)
```

### Example Communication

**Client sends:**
```shell
1
>>> TLS 1.2, RecordHeader [length 0005]
    17 03 03 00 13
>>> TLS 1.2, InnerContent [length 0001]
    17
write to 0x61d7f685b0d0 [0x61d7f693ad53] (24 bytes => 24 (0x18))
0000 - 17 03 03 00 13 d6 d0 a5-f9 af 0b 6b 63 e3 89 ce   ...........kc...
0010 - c1 eb 0b 24 5f 0d 83 d4-                          ...$_...
```

**Server receives:**
```shell
read from 0x585202f0ac90 [0x585202f24983] (5 bytes => 5 (0x5))
0000 - 17 03 03 00 13                                    .....
<<< TLS 1.2, RecordHeader [length 0005]
    17 03 03 00 13
read from 0x585202f0ac90 [0x585202f24988] (19 bytes => 19 (0x13))
0000 - d6 d0 a5 f9 af 0b 6b 63-e3 89 ce c1 eb 0b 24 5f   ......kc......$_
0010 - 0d 83 d4                                          ...
<<< TLS 1.3, InnerContent [length 0001]
    17
1
```

**Analysis:**
- Header: 5 bytes = 1-byte Content-Type | 2-bytes Version | 2-bytes Length
  - `17` → Content-Type=0x17 (Application Data)
  - `03 03` → Version=0x0303 (TLS 1.2)
  - `00 13` → Length=0x0013 (19 bytes)
- Payload: 19 bytes (encrypted): `d6 d0 a5 f9 af 0b 6b 63 e3 89 ce c1 eb 0b 24 5f 0d 83 d4`

## System Call Analysis

Use `strace` to observe low-level socket operations:

```bash
./strace-client.sh  # Trace client syscalls
./strace-server.sh  # Trace server syscalls
```

**Client sends:**
```shell
1
read(0, "1\n", 8192)                    = 2
write(1, ">>> TLS 1.2, RecordHeader [length 0005]\n", 40>>> TLS 1.2, RecordHeader [length 0005]
) = 40
write(1, "    17 03 03 00 13\n", 19    17 03 03 00 13
)    = 19
write(1, ">>> TLS 1.2, InnerContent [length 0001]\n", 40>>> TLS 1.2, InnerContent [length 0001]
) = 40
write(1, "    17\n", 7    17
)                 = 7
write(3, "\27\3\3\0\23\263\250e\343\377\252\334\334\366[\330(S\343p\340\214\317h", 24) = 24
```

**Server receives:**
```shell
read(4, "\27\3\3\0\23", 5)              = 5
write(1, "<<< TLS 1.2, RecordHeader [length 0005]\n", 40<<< TLS 1.2, RecordHeader [length 0005]
) = 40
write(1, "    17 03 03 00 13\n", 19    17 03 03 00 13
)    = 19
read(4, "\263\250e\343\377\252\334\334\366[\330(S\343p\340\214\317h", 19) = 19
write(1, "<<< TLS 1.3, InnerContent [length 0001]\n", 40<<< TLS 1.3, InnerContent [length 0001]
) = 40
write(1, "    17\n", 7    17
)                 = 7
write(1, "1\n", 21
)                      = 2
```

**Key observations:**
- Client uses `write(3, data, 24)` to send TLS record to socket (fd=3)
- Server performs two `read()` calls:
  1. `read(4, header, 5)` - Read 5-byte header
  2. `read(4, payload, 19)` - Read 19-byte payload (length from header)
- This demonstrates how TLS implements framing over TCP streams