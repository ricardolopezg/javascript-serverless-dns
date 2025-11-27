---
title: Managing Incoming Network Connections
---
This document explains how incoming network sockets are managed based on their protocol. Sockets are either pooled for reuse or closed if pooling is not possible, optimizing network resource usage.

```mermaid
flowchart TD
  node1["Parking or Closing Incoming Connections"]:::HeadingStyle
  click node1 goToHeading "Parking or Closing Incoming Connections"
  node2["Checking Socket Eligibility for Pooling"]:::HeadingStyle
  click node2 goToHeading "Checking Socket Eligibility for Pooling"
  node3["Finalizing Socket Check-in"]:::HeadingStyle
  click node3 goToHeading "Finalizing Socket Check-in"
  node4["Handling UDP Socket Parking or Closure"]:::HeadingStyle
  click node4 goToHeading "Handling UDP Socket Parking or Closure"
  node1 -->|"TCP"| node2
  node1 -->|"UDP"| node4
  node2 -->|"Eligible and space available"| node3
  node2 -->|"Not eligible or no space"| node4
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

# Where is this flow used?

This flow is used multiple times in the codebase as represented in the following diagram:

```mermaid
graph TD;
      242f0198851de586303d5f7fef79f39c1d94826dfcb7265a70c579f5279962e7(src/…/node/dns-transport.js::Transport.udpquery) --> 784f1818d7b59b47fe8299f008bbd1b999dbb8f08c221516c8f9781a50ddf55e(src/…/node/dns-transport.js::Transport.parkConn)

8104e61c05fb4118e89f308f6e4820111c12d4ac1881ab1f519d3fcecc7b7698(src/…/dns-op/resolver.js::resolveDnsUpstream) --> 242f0198851de586303d5f7fef79f39c1d94826dfcb7265a70c579f5279962e7(src/…/node/dns-transport.js::Transport.udpquery)

8104e61c05fb4118e89f308f6e4820111c12d4ac1881ab1f519d3fcecc7b7698(src/…/dns-op/resolver.js::resolveDnsUpstream) --> 50bf232e8db402a016070cff7e3d3bff35a738de5a835cf04337d9014a2312b2(src/…/node/dns-transport.js::Transport.tcpquery)

6a8da81bb668375fae6262852ff83e8c70f5020063d6a85d9b663874b53bc617(src/…/command-control/cc.js::domainNameToList) --> 8104e61c05fb4118e89f308f6e4820111c12d4ac1881ab1f519d3fcecc7b7698(src/…/dns-op/resolver.js::resolveDnsUpstream)

c4284cc89cfe5167db2f44081a566a421f91dc0cbc8077528668dc3ca096dd56(src/…/command-control/cc.js::CommandControl.commandOperation) --> 6a8da81bb668375fae6262852ff83e8c70f5020063d6a85d9b663874b53bc617(src/…/command-control/cc.js::domainNameToList)

1ee3e410eb76b84533705554985e772d052147c2c3087e4d3876f8af16d4572d(src/…/command-control/cc.js::CommandControl.exec) --> c4284cc89cfe5167db2f44081a566a421f91dc0cbc8077528668dc3ca096dd56(src/…/command-control/cc.js::CommandControl.commandOperation)

4b89e52dba3ee51396b4c97ac472edf6cce97a95e3e0d450aac51807e371a072(src/…/dns-op/resolver.js::DNSResolver.resolveDns) --> 8104e61c05fb4118e89f308f6e4820111c12d4ac1881ab1f519d3fcecc7b7698(src/…/dns-op/resolver.js::resolveDnsUpstream)

66557e9a1c97b0ec60a2a495f43daef2c9343d8dd94c4edbfc7d86d8df7ba307(src/…/dns-op/resolver.js::DNSResolver.exec) --> 4b89e52dba3ee51396b4c97ac472edf6cce97a95e3e0d450aac51807e371a072(src/…/dns-op/resolver.js::DNSResolver.resolveDns)

50bf232e8db402a016070cff7e3d3bff35a738de5a835cf04337d9014a2312b2(src/…/node/dns-transport.js::Transport.tcpquery) --> 784f1818d7b59b47fe8299f008bbd1b999dbb8f08c221516c8f9781a50ddf55e(src/…/node/dns-transport.js::Transport.parkConn)


classDef mainFlowStyle color:#000000,fill:#7CB9F4
classDef rootsStyle color:#000000,fill:#00FFF4
classDef Style1 color:#000000,fill:#00FFAA
classDef Style2 color:#000000,fill:#FFFF00
classDef Style3 color:#000000,fill:#AA7CB9

%% Swimm:
%% graph TD;
%%       242f0198851de586303d5f7fef79f39c1d94826dfcb7265a70c579f5279962e7(<SwmPath>[src/…/node/dns-transport.js](src/core/node/dns-transport.js)</SwmPath>::Transport.udpquery) --> 784f1818d7b59b47fe8299f008bbd1b999dbb8f08c221516c8f9781a50ddf55e(<SwmPath>[src/…/node/dns-transport.js](src/core/node/dns-transport.js)</SwmPath>::Transport.parkConn)
%% 
%% 8104e61c05fb4118e89f308f6e4820111c12d4ac1881ab1f519d3fcecc7b7698(<SwmPath>[src/…/dns-op/resolver.js](src/plugins/dns-op/resolver.js)</SwmPath>::resolveDnsUpstream) --> 242f0198851de586303d5f7fef79f39c1d94826dfcb7265a70c579f5279962e7(<SwmPath>[src/…/node/dns-transport.js](src/core/node/dns-transport.js)</SwmPath>::Transport.udpquery)
%% 
%% 8104e61c05fb4118e89f308f6e4820111c12d4ac1881ab1f519d3fcecc7b7698(<SwmPath>[src/…/dns-op/resolver.js](src/plugins/dns-op/resolver.js)</SwmPath>::resolveDnsUpstream) --> 50bf232e8db402a016070cff7e3d3bff35a738de5a835cf04337d9014a2312b2(<SwmPath>[src/…/node/dns-transport.js](src/core/node/dns-transport.js)</SwmPath>::Transport.tcpquery)
%% 
%% 6a8da81bb668375fae6262852ff83e8c70f5020063d6a85d9b663874b53bc617(<SwmPath>[src/…/command-control/cc.js](src/plugins/command-control/cc.js)</SwmPath>::domainNameToList) --> 8104e61c05fb4118e89f308f6e4820111c12d4ac1881ab1f519d3fcecc7b7698(<SwmPath>[src/…/dns-op/resolver.js](src/plugins/dns-op/resolver.js)</SwmPath>::resolveDnsUpstream)
%% 
%% c4284cc89cfe5167db2f44081a566a421f91dc0cbc8077528668dc3ca096dd56(<SwmPath>[src/…/command-control/cc.js](src/plugins/command-control/cc.js)</SwmPath>::CommandControl.commandOperation) --> 6a8da81bb668375fae6262852ff83e8c70f5020063d6a85d9b663874b53bc617(<SwmPath>[src/…/command-control/cc.js](src/plugins/command-control/cc.js)</SwmPath>::domainNameToList)
%% 
%% 1ee3e410eb76b84533705554985e772d052147c2c3087e4d3876f8af16d4572d(<SwmPath>[src/…/command-control/cc.js](src/plugins/command-control/cc.js)</SwmPath>::CommandControl.exec) --> c4284cc89cfe5167db2f44081a566a421f91dc0cbc8077528668dc3ca096dd56(<SwmPath>[src/…/command-control/cc.js](src/plugins/command-control/cc.js)</SwmPath>::CommandControl.commandOperation)
%% 
%% 4b89e52dba3ee51396b4c97ac472edf6cce97a95e3e0d450aac51807e371a072(<SwmPath>[src/…/dns-op/resolver.js](src/plugins/dns-op/resolver.js)</SwmPath>::DNSResolver.resolveDns) --> 8104e61c05fb4118e89f308f6e4820111c12d4ac1881ab1f519d3fcecc7b7698(<SwmPath>[src/…/dns-op/resolver.js](src/plugins/dns-op/resolver.js)</SwmPath>::resolveDnsUpstream)
%% 
%% 66557e9a1c97b0ec60a2a495f43daef2c9343d8dd94c4edbfc7d86d8df7ba307(<SwmPath>[src/…/dns-op/resolver.js](src/plugins/dns-op/resolver.js)</SwmPath>::DNSResolver.exec) --> 4b89e52dba3ee51396b4c97ac472edf6cce97a95e3e0d450aac51807e371a072(<SwmPath>[src/…/dns-op/resolver.js](src/plugins/dns-op/resolver.js)</SwmPath>::DNSResolver.resolveDns)
%% 
%% 50bf232e8db402a016070cff7e3d3bff35a738de5a835cf04337d9014a2312b2(<SwmPath>[src/…/node/dns-transport.js](src/core/node/dns-transport.js)</SwmPath>::Transport.tcpquery) --> 784f1818d7b59b47fe8299f008bbd1b999dbb8f08c221516c8f9781a50ddf55e(<SwmPath>[src/…/node/dns-transport.js](src/core/node/dns-transport.js)</SwmPath>::Transport.parkConn)
%% 
%% 
%% classDef mainFlowStyle color:#000000,fill:#7CB9F4
%% classDef rootsStyle color:#000000,fill:#00FFF4
%% classDef Style1 color:#000000,fill:#00FFAA
%% classDef Style2 color:#000000,fill:#FFFF00
%% classDef Style3 color:#000000,fill:#AA7CB9
```

# Parking or Closing Incoming Connections

<SwmSnippet path="/src/core/node/dns-transport.js" line="118">

---

In <SwmToken path="src/core/node/dns-transport.js" pos="118:1:1" line-data="  parkConn(sock, proto) {">`parkConn`</SwmToken>, we decide how to handle a socket based on its protocol. For TCP, we try to hand it off to the TCP connection pool; for UDP, we use the UDP pool. If the pool can't accept the socket, we close it. Next, we call into <SwmPath>[src/…/dns/conns.js](src/core/dns/conns.js)</SwmPath> to see if the pool will take the socket, which determines if we keep or drop the connection.

```javascript
  parkConn(sock, proto) {
    if (proto === "tcp") {
      const ok = this.tcpconns.give(sock);
      if (!ok) this.closeTcp(sock);
    } else if (proto === "udp") {
      const ok = this.udpconns.give(sock);
```

---

</SwmSnippet>

## Checking Socket Eligibility for Pooling

<SwmSnippet path="/src/core/dns/conns.js" line="47">

---

In <SwmToken path="src/core/dns/conns.js" pos="47:1:1" line-data="  give(socket) {">`give`</SwmToken>, we check if the socket is in a good state for pooling—it's not pending, it's writable, and it's ready. If the pool is full, we try to free up space with a sweep. If all checks pass, we move on to actually parking the socket.

```javascript
  give(socket) {
    if (socket.pending) return false;
    if (!socket.writable) return false;
    if (!this.ready(socket)) return false;

    if (this.pool.has(socket)) return true;

    const free = this.pool.size < this.size || this.sweep();
    if (!free) return false;

```

---

</SwmSnippet>

### Freeing Up Pool Space and Logging

See <SwmLink doc-title="Sweeping DNS Connections">[Sweeping DNS Connections](/.swm/sweeping-dns-connections.9ncirpbh.sw.md)</SwmLink>

### Finalizing Socket Check-in

<SwmSnippet path="/src/core/dns/conns.js" line="57">

---

Back in <SwmToken path="src/core/node/dns-transport.js" pos="120:11:11" line-data="      const ok = this.tcpconns.give(sock);">`give`</SwmToken>, after eligibility and space checks, we call <SwmToken path="src/core/dns/conns.js" pos="57:5:5" line-data="    return this.checkin(socket);">`checkin`</SwmToken> to actually add the socket to the pool. This is where the socket gets parked and managed for future use.

```javascript
    return this.checkin(socket);
  }
```

---

</SwmSnippet>

<SwmSnippet path="/src/core/dns/conns.js" line="115">

---

<SwmToken path="src/core/dns/conns.js" pos="115:1:1" line-data="  checkin(sock) {">`checkin`</SwmToken> sets up the socket for pooling: it attaches a report, enables keepalive, pauses the socket, and binds cleanup handlers for close and error. It then logs the check-in and adds the socket to the pool.

```javascript
  checkin(sock) {
    const report = this.mkreport();

    sock.setKeepAlive(true, this.keepalive);
    sock.pause();
    sock.on("close", this.evict.bind(this));
    sock.on("error", this.evict.bind(this));

    this.pool.set(sock, report);

    log.d(report.id, "checkin, size:", this.pool.size);
    return true;
  }
```

---

</SwmSnippet>

## Handling UDP Socket Parking or Closure

<SwmSnippet path="/src/core/node/dns-transport.js" line="124">

---

Back in <SwmToken path="src/core/node/dns-transport.js" pos="118:1:1" line-data="  parkConn(sock, proto) {">`parkConn`</SwmToken>, if the UDP pool can't take the socket, we close it right away. This prevents resource leaks since UDP sockets aren't reused.

```javascript
      if (!ok) this.closeUdp(sock);
    }
  }
```

---

</SwmSnippet>

<SwmSnippet path="/src/core/node/dns-transport.js" line="173">

---

<SwmToken path="src/core/node/dns-transport.js" pos="173:1:1" line-data="  closeUdp(sock) {">`closeUdp`</SwmToken> adds a stub error listener to the socket before disconnecting and closing it. This avoids unhandled error events, which is a repo-specific detail since sockets here don't have error listeners by default.

```javascript
  closeUdp(sock) {
    if (!sock || sock.destroyed) return;
    // the socket is expected to not have any error-listeners
    // so we add one just in case to avoid unhandled errors
    sock.on("error", util.stub);
    sock.disconnect();
    sock.close();
  }
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBamF2YXNjcmlwdC1zZXJ2ZXJsZXNzLWRucyUzQSUzQXJpY2FyZG9sb3Blemc=" repo-name="javascript-serverless-dns"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
