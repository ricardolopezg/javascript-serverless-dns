---
title: Handling DNS-over-TCP Queries
---
This document describes how DNS queries sent by clients over TCP are received, assembled, and processed. Each query is forwarded to an upstream resolver, and the response is returned to the client. The flow supports handling multiple queries in a single connection for efficient DNS resolution.

```mermaid
flowchart TD
  node1["Reading and Buffering TCP DNS Queries"]:::HeadingStyle
  click node1 goToHeading "Reading and Buffering TCP DNS Queries"
  node1 --> node2{"Is there a complete DNS query to process?"}
  node2 -->|"No"| node1
  node2 -->|"Yes"| node3["Forwarding DNS Queries to Upstream"]:::HeadingStyle
  click node3 goToHeading "Forwarding DNS Queries to Upstream"
  node3 --> node4["Writing DNS Answers Back to TCP Clients"]:::HeadingStyle
  click node4 goToHeading "Writing DNS Answers Back to TCP Clients"
  node4 --> node5{"Are there additional pipelined queries?"}
  node5 -->|"Yes"| node1
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

# Where is this flow used?

This flow is used multiple times in the codebase as represented in the following diagram:

```mermaid
graph TD;
      900ccb321fe23baa523d9ad5d054beea83e670a38afa8359ce0a1dc7cf6a4b15(src/server-node.js::serveTLS) --> cf090485bf8d5fbbb13cf52382f159bf0b4f759e64dd976ef6f968d42846bd62(src/server-node.js::handleTCPData):::mainFlowStyle

3045764f3abaaafc715c65dde62e10b3bdcb2b02a11a89428d894c6bdcd647fc(src/server-node.js::serveTCP) --> cf090485bf8d5fbbb13cf52382f159bf0b4f759e64dd976ef6f968d42846bd62(src/server-node.js::handleTCPData):::mainFlowStyle


classDef mainFlowStyle color:#000000,fill:#7CB9F4
classDef rootsStyle color:#000000,fill:#00FFF4
classDef Style1 color:#000000,fill:#00FFAA
classDef Style2 color:#000000,fill:#FFFF00
classDef Style3 color:#000000,fill:#AA7CB9

%% Swimm:
%% graph TD;
%%       900ccb321fe23baa523d9ad5d054beea83e670a38afa8359ce0a1dc7cf6a4b15(<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>::<SwmToken path="src/server-node.js" pos="463:14:14" line-data="    const dot1 = tls.createServer(secOpts, serveTLS).listen(dot1Opts, () =&gt; {">`serveTLS`</SwmToken>) --> cf090485bf8d5fbbb13cf52382f159bf0b4f759e64dd976ef6f968d42846bd62(<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>::<SwmToken path="src/server-node.js" pos="1073:4:4" line-data="async function handleTCPData(socket, chunk, sb, host, flag) {">`handleTCPData`</SwmToken>):::mainFlowStyle
%% 
%% 3045764f3abaaafc715c65dde62e10b3bdcb2b02a11a89428d894c6bdcd647fc(<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>::<SwmToken path="src/server-node.js" pos="425:14:14" line-data="    const dotct = net.createServer(serverOpts, serveTCP).listen(dotOpts, () =&gt; {">`serveTCP`</SwmToken>) --> cf090485bf8d5fbbb13cf52382f159bf0b4f759e64dd976ef6f968d42846bd62(<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>::<SwmToken path="src/server-node.js" pos="1073:4:4" line-data="async function handleTCPData(socket, chunk, sb, host, flag) {">`handleTCPData`</SwmToken>):::mainFlowStyle
%% 
%% 
%% classDef mainFlowStyle color:#000000,fill:#7CB9F4
%% classDef rootsStyle color:#000000,fill:#00FFF4
%% classDef Style1 color:#000000,fill:#00FFAA
%% classDef Style2 color:#000000,fill:#FFFF00
%% classDef Style3 color:#000000,fill:#AA7CB9
```

# Reading and Buffering TCP DNS Queries

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Receive TCP data"] --> node2{"Is there data to process?"}
  click node1 openCode "src/server-node.js:1073:1075"
  node2 -->|"No"| node7["Return (no data processed)"]
  click node2 openCode "src/server-node.js:1075:1075"
  node2 -->|"Yes"| node3{"Is DNS header fully read and query size valid?"}
  click node3 openCode "src/server-node.js:1078:1095"
  node3 -->|"No"| node7
  node3 -->|"Yes"| node4["Extract and reset DNS query buffer"]
  click node4 openCode "src/server-node.js:1118:1118"
  node4 --> node5["Process DNS query"]
  click node5 openCode "src/server-node.js:1119:1119"
  node5 --> node6{"Is there pipelined (out-of-band) data?"}
  click node6 openCode "src/server-node.js:1122:1125"
  node6 -->|"Yes"| node8["Recursively process additional pipelined data"]
  click node8 openCode "src/server-node.js:1124:1125"
  node6 -->|"No"| node7
  node8 --> node7
  click node7 openCode "src/server-node.js:1127:1128"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Receive TCP data"] --> node2{"Is there data to process?"}
%%   click node1 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1073:1075"
%%   node2 -->|"No"| node7["Return (no data processed)"]
%%   click node2 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1075:1075"
%%   node2 -->|"Yes"| node3{"Is DNS header fully read and query size valid?"}
%%   click node3 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1078:1095"
%%   node3 -->|"No"| node7
%%   node3 -->|"Yes"| node4["Extract and reset DNS query buffer"]
%%   click node4 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1118:1118"
%%   node4 --> node5["Process DNS query"]
%%   click node5 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1119:1119"
%%   node5 --> node6{"Is there pipelined (out-of-band) data?"}
%%   click node6 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1122:1125"
%%   node6 -->|"Yes"| node8["Recursively process additional pipelined data"]
%%   click node8 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1124:1125"
%%   node6 -->|"No"| node7
%%   node8 --> node7
%%   click node7 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1127:1128"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/server-node.js" line="1073">

---

HandleTCPData kicks off the flow by reading incoming TCP data and assembling DNS queries using a custom scratch buffer (sb). It reads the length header, validates it, and copies query data into sb, handling partial reads and pipelined queries. Once a full query is assembled, it resets sb and calls <SwmToken path="src/server-node.js" pos="1119:7:7" line-data="    n += await handleTCPQuery(b, socket, host, flag);">`handleTCPQuery`</SwmToken> to process the DNS query. Recursive calls handle any extra data in the chunk.

```javascript
async function handleTCPData(socket, chunk, sb, host, flag) {
  const cl = chunk.byteLength;
  if (cl <= 0) return 0;

  // read header first which contains length(dns-query)
  const rem = dnsutil.dnsHeaderSize - sb.qlenBufOffset;
  if (rem > 0) {
    const seek = Math.min(rem, cl);
    const read = chunk.slice(0, seek);
    sb.qlenBuf.fill(read, sb.qlenBufOffset);
    sb.qlenBufOffset += seek;
  }

  // header has not been read fully, yet; expect more data
  // www.rfc-editor.org/rfc/rfc7766#section-8
  if (sb.qlenBufOffset !== dnsutil.dnsHeaderSize) return 0;

  const qlen = sb.qlenBuf.readUInt16BE();
  if (!dnsutil.validateSize(qlen)) {
    log.w(`tcp: query size err: ql:${qlen} cl:${cl} rem:${rem}`);
    close(socket);
    return 0;
  }

  // rem bytes already read, is any more left in chunk?
  const size = cl - rem;
  if (size <= 0) return 0;
  // gobble up at most qlen bytes from chunk starting rem-th byte
  const qlimit = rem + Math.min(qlen - sb.qBufOffset, size);
  // hopefully fast github.com/nodejs/node/issues/20130#issuecomment-382417255
  // chunk out dns-query starting rem-th byte
  const data = chunk.slice(rem, qlimit);
  // out of band data, if any
  const oob = qlimit < cl ? chunk.slice(qlimit) : null;

  sb.allocOnce(qlen);

  sb.qBuf.fill(data, sb.qBufOffset);
  sb.qBufOffset += data.byteLength;

  log.d(`tcp: q: ${qlen}, sb.q: ${sb.qBufOffset}, cl: ${cl}, sz: ${size}`);
  let n = 0;
  // exactly qlen bytes read till now, handle the dns query
  if (sb.qBufOffset === qlen) {
    // extract out the query and reset the scratch-buffer
    const b = sb.reset();
    n += await handleTCPQuery(b, socket, host, flag);

    // if there is any out of band data, handle it
    if (!bufutil.emptyBuf(oob)) {
      log.d(`tcp: pipelined, handle oob: ${oob.byteLength}`);
      n += await handleTCPData(socket, oob, sb, host, flag);
    }
  } // continue reading from socket
  return n;
}
```

---

</SwmSnippet>

# Processing Assembled DNS Queries

<SwmSnippet path="/src/server-node.js" line="1137">

---

In <SwmToken path="src/server-node.js" pos="1137:4:4" line-data="async function handleTCPQuery(q, socket, host, flag) {">`handleTCPQuery`</SwmToken>, we validate the query and socket, set up a transaction ID, and then call <SwmToken path="src/server-node.js" pos="1148:7:7" line-data="    r = await resolveQuery(rxid, q, host, flag);">`resolveQuery`</SwmToken> to actually resolve the DNS query. This separates query assembly from resolution, keeping the flow modular.

```javascript
async function handleTCPQuery(q, socket, host, flag) {
  heartbeat();

  let n = 0;
  let ok = true;
  if (bufutil.emptyBuf(q) || !tcpOkay(socket)) return 0;

  /** @type {Uint8Array?} */
  let r = null;
  const rxid = util.xid();
  try {
    r = await resolveQuery(rxid, q, host, flag);
```

---

</SwmSnippet>

## Forwarding DNS Queries to Upstream

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Forward DNS query to upstream DNS-over-HTTPS server (host)"] --> node2["Proxying DNS Requests"]
  click node1 openCode "src/server-node.js:1204:1219"
  node2 --> node3{"Is the DNS response empty?"}
  
  node3 -->|"No"| node4["Return successful DNS answer to client"]
  click node3 openCode "src/server-node.js:1223:1225"
  node3 -->|"Yes"| node5["Return DNS server failure to client"]
  click node5 openCode "src/server-node.js:1226:1228"
  click node4 openCode "src/server-node.js:1224:1225"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
click node2 goToHeading "Proxying DNS Requests"
node2:::HeadingStyle

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Forward DNS query to upstream DNS-over-HTTPS server (host)"] --> node2["Proxying DNS Requests"]
%%   click node1 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1204:1219"
%%   node2 --> node3{"Is the DNS response empty?"}
%%   
%%   node3 -->|"No"| node4["Return successful DNS answer to client"]
%%   click node3 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1223:1225"
%%   node3 -->|"Yes"| node5["Return DNS server failure to client"]
%%   click node5 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1226:1228"
%%   click node4 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1224:1225"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
%% click node2 goToHeading "Proxying DNS Requests"
%% node2:::HeadingStyle
```

<SwmSnippet path="/src/server-node.js" line="1204">

---

In <SwmToken path="src/server-node.js" pos="1204:4:4" line-data="async function resolveQuery(rxid, q, host, flag) {">`resolveQuery`</SwmToken>, we package the DNS query as a POST request with the right headers and send it to the upstream resolver. We then call <SwmToken path="src/server-node.js" pos="1219:9:9" line-data="  const r = await handleRequest(util.mkFetchEvent(freq));">`handleRequest`</SwmToken> from <SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath> to actually proxy the request and get the response.

```javascript
async function resolveQuery(rxid, q, host, flag) {
  // Using POST, since GET requests cannot be greater than 2KB,
  // where-as DNS-over-TCP msgs could be upto 64KB in size.
  const freq = new Request(`https://${host}/${flag}`, {
    method: "POST",
    // TODO: populate req ip in x-nile-client-ip header
    // TODO: add host header
    headers: util.concatHeaders(
      util.dnsHeaders(),
      util.contentLengthHeader(q),
      util.rxidHeader(rxid)
    ),
    body: q,
  });

  const r = await handleRequest(util.mkFetchEvent(freq));

```

---

</SwmSnippet>

### Proxying DNS Requests

<SwmSnippet path="/src/core/doh.js" line="25">

---

HandleRequest just delegates the DNS request event to <SwmToken path="src/core/doh.js" pos="26:3:3" line-data="  return proxyRequest(event);">`proxyRequest`</SwmToken>, which does the actual proxying to the upstream resolver. This keeps the flow modular and focused.

```javascript
export function handleRequest(event) {
  return proxyRequest(event);
}
```

---

</SwmSnippet>

### Relaying DNS Queries to Upstream Servers

See <SwmLink doc-title="Processing DNS-over-HTTPS Requests">[Processing DNS-over-HTTPS Requests](/.swm/processing-dns-over-https-requests.je20mtdr.sw.md)</SwmLink>

### Formatting and Sending Error Responses

<SwmSnippet path="/src/core/doh.js" line="75">

---

ErrorResponse formats an error using <SwmToken path="src/core/doh.js" pos="76:7:9" line-data="  const eres = pres.errResponse(&quot;doh.js&quot;, err);">`pres.errResponse`</SwmToken> with the context <SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>, then sends it out using <SwmToken path="src/core/doh.js" pos="77:1:3" line-data="  io.dnsExceptionResponse(eres);">`io.dnsExceptionResponse`</SwmToken>. This ties errors to the <SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath> module for easier tracing.

```javascript
function errorResponse(io, err = null) {
  const eres = pres.errResponse("doh.js", err);
  io.dnsExceptionResponse(eres);
}
```

---

</SwmSnippet>

### Sending DNS Error Responses to Clients

See <SwmLink doc-title="Handling DNS Exception Responses">[Handling DNS Exception Responses](/.swm/handling-dns-exception-responses.b69x7qnt.sw.md)</SwmLink>

### Handling Upstream DNS Responses

<SwmSnippet path="/src/server-node.js" line="1221">

---

Back in <SwmToken path="src/server-node.js" pos="1148:7:7" line-data="    r = await resolveQuery(rxid, q, host, flag);">`resolveQuery`</SwmToken>, after getting the response from <SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>, we check if the answer is empty. If not, we normalize and return it; if it is, we log and send a SERVFAIL response.

```javascript
  const ans = await r.arrayBuffer();

  if (!bufutil.emptyBuf(ans)) {
    return bufutil.normalize8(ans);
  } else {
    log.w(rxid, host, "empty ans, send servfail; flags?", flag);
    return dnsutil.servfailQ(q);
  }
}
```

---

</SwmSnippet>

## Writing DNS Answers Back to TCP Clients

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Receive DNS response from resolver"]
  click node1 openCode "src/server-node.js:1149:1157"
  node1 --> node2{"Is response empty?"}
  click node2 openCode "src/server-node.js:1149:1152"
  node2 -->|"Yes"| node3["Close connection"]
  click node3 openCode "src/server-node.js:1163:1165"
  node2 -->|"No"| node4["Send response to client"]
  click node4 openCode "src/server-node.js:1153:1156"
  node4 --> node5{"Did sending succeed?"}
  click node5 openCode "src/server-node.js:1157:1162"
  node5 -->|"No"| node3
  node5 -->|"Yes"| node6["Keep connection open for more queries"]
  click node6 openCode "src/server-node.js:1165:1166"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Receive DNS response from resolver"]
%%   click node1 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1149:1157"
%%   node1 --> node2{"Is response empty?"}
%%   click node2 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1149:1152"
%%   node2 -->|"Yes"| node3["Close connection"]
%%   click node3 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1163:1165"
%%   node2 -->|"No"| node4["Send response to client"]
%%   click node4 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1153:1156"
%%   node4 --> node5{"Did sending succeed?"}
%%   click node5 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1157:1162"
%%   node5 -->|"No"| node3
%%   node5 -->|"Yes"| node6["Keep connection open for more queries"]
%%   click node6 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1165:1166"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/server-node.js" line="1149">

---

Back in <SwmToken path="src/server-node.js" pos="1119:7:7" line-data="    n += await handleTCPQuery(b, socket, host, flag);">`handleTCPQuery`</SwmToken>, after getting the answer from <SwmToken path="src/server-node.js" pos="1148:7:7" line-data="    r = await resolveQuery(rxid, q, host, flag);">`resolveQuery`</SwmToken>, we encode and write the response to the TCP socket. If the answer is empty or an error occurs, we log and close the socket.

```javascript
    if (bufutil.emptyBuf(r)) {
      log.w(rxid, "tcp: empty ans from resolver");
      ok = false;
    } else {
      const rlBuf = bufutil.encodeUint8ArrayBE(r.byteLength, 2);
      const data = new Uint8Array([...rlBuf, ...r]);
      n = measuredWrite(rxid, socket, data);
    }
  } catch (e) {
    ok = false;
    log.w(rxid, "tcp: send fail, err", e);
  }

  // close socket when !ok
  if (!ok) {
    close(socket);
  } // else: expect pipelined queries on the same socket

  return n;
}
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBamF2YXNjcmlwdC1zZXJ2ZXJsZXNzLWRucyUzQSUzQXJpY2FyZG9sb3Blemc=" repo-name="javascript-serverless-dns"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
