---
title: Handling DNS-over-HTTPS Requests
---
This document outlines the process for handling DNS-over-HTTPS requests, enabling secure DNS resolution for clients. The flow receives an HTTPS DNS request, validates and processes it, supports user-specific features, and returns a DNS response to the client.

```mermaid
flowchart TD
  node1["Receiving and Preparing HTTPS Requests"]:::HeadingStyle
  click node1 goToHeading "Receiving and Preparing HTTPS Requests"
  node1 --> node2{"Is request valid and within allowed size?"}
  node2 -->|"No"| node7["Finalizing and Returning the DNS Response"]:::HeadingStyle
  click node7 goToHeading "Finalizing and Returning the DNS Response"
  node2 -->|"Yes"| node3["Building and Forwarding the HTTP Request"]:::HeadingStyle
  click node3 goToHeading "Building and Forwarding the HTTP Request"
  node3 --> node4["Dispatching the DNS Request"]:::HeadingStyle
  click node4 goToHeading "Dispatching the DNS Request"
  node4 --> node5["Processing and Routing the DNS Query"]:::HeadingStyle
  click node5 goToHeading "Processing and Routing the DNS Query"
  node5 --> node6["Executing Plugin Operations"]:::HeadingStyle
  click node6 goToHeading "Executing Plugin Operations"
  node6 --> node7
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

# Receiving and Preparing HTTPS Requests

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Receive HTTPS DNS request"] --> node2["Buffer request body"]
    click node1 openCode "src/server-node.js:1243:1244"
    click node2 openCode "src/server-node.js:1245:1254"
    node2 --> node3{"Is request a POST and is body size valid?"}
    click node3 openCode "src/server-node.js:1259:1263"
    node3 -->|"No"| node4["Reject request with error status and CORS headers (if needed, based on user agent)"]
    click node4 openCode "src/server-node.js:1260:1262"
    node4 --> node6["Log rejected request"]
    click node6 openCode "src/server-node.js:1262:1263"
    node3 -->|"Yes"| node5["Process DNS-over-HTTPS request"]
    click node5 openCode "src/server-node.js:1264:1265"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["Receive HTTPS DNS request"] --> node2["Buffer request body"]
%%     click node1 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1243:1244"
%%     click node2 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1245:1254"
%%     node2 --> node3{"Is request a POST and is body size valid?"}
%%     click node3 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1259:1263"
%%     node3 -->|"No"| node4["Reject request with error status and CORS headers (if needed, based on user agent)"]
%%     click node4 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1260:1262"
%%     node4 --> node6["Log rejected request"]
%%     click node6 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1262:1263"
%%     node3 -->|"Yes"| node5["Process DNS-over-HTTPS request"]
%%     click node5 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1264:1265"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/server-node.js" line="1243">

---

ServeHTTPS sets up the initial request handling, buffers the incoming data, and after basic validation, passes control to <SwmToken path="src/server-node.js" pos="1265:1:1" line-data="      handleHTTPRequest(b, req, res);">`handleHTTPRequest`</SwmToken> to actually process the DNS query.

```javascript
function serveHTTPS(req, res) {
  trapRequestResponseEvents(req, res);
  const ua = req.headers["user-agent"];
  const buffers = [];

  // if using for await loop, then it must be wrapped in a
  // try-catch block: stackoverflow.com/questions/69169226
  // if not, errors from reading req escapes unhandled.
  // for example: req is being read from, but the underlying
  // socket has been the closed (resulting in err_premature_close)
  req.on("data", (chunk) => buffers.push(chunk));

  req.on("end", () => {
    const b = bufutil.concatBuf(buffers);
    const bLen = b.byteLength;

    if (util.isPostRequest(req) && !dnsutil.validResponseSize(b)) {
      res.writeHead(dnsutil.dohStatusCode(b), util.corsHeadersIfNeeded(ua));
      res.end();
      log.w(`h2: req body length out of bounds: ${bLen}`);
    } else {
      log.d("----> doh request", req.method, bLen, req.url);
      handleHTTPRequest(b, req, res);
    }
  });
}
```

---

</SwmSnippet>

# Building and Forwarding the HTTP Request

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Receive HTTP request"] --> node2["Prepare request for processing"]
  click node1 openCode "src/server-node.js:1275:1278"
  click node2 openCode "src/server-node.js:1278:1295"
  node2 --> node3["Dispatching the DNS Request"]
  
  node3 --> node4{"Is response writable?"}
  click node4 openCode "src/server-node.js:1299:1301"
  node4 -->|"Yes"| node5{"Is response non-empty?"}
  click node5 openCode "src/server-node.js:1310:1318"
  node5 -->|"Yes"| node6["Return DNS answer to client"]
  click node6 openCode "src/server-node.js:1313:1315"
  node5 -->|"No"| node6
  node4 -->|"No"| node6
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
click node3 goToHeading "Dispatching the DNS Request"
node3:::HeadingStyle

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Receive HTTP request"] --> node2["Prepare request for processing"]
%%   click node1 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1275:1278"
%%   click node2 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1278:1295"
%%   node2 --> node3["Dispatching the DNS Request"]
%%   
%%   node3 --> node4{"Is response writable?"}
%%   click node4 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1299:1301"
%%   node4 -->|"Yes"| node5{"Is response non-empty?"}
%%   click node5 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1310:1318"
%%   node5 -->|"Yes"| node6["Return DNS answer to client"]
%%   click node6 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1313:1315"
%%   node5 -->|"No"| node6
%%   node4 -->|"No"| node6
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
%% click node3 goToHeading "Dispatching the DNS Request"
%% node3:::HeadingStyle
```

<SwmSnippet path="/src/server-node.js" line="1275">

---

In <SwmToken path="src/server-node.js" pos="1275:4:4" line-data="async function handleHTTPRequest(b, req, res) {">`handleHTTPRequest`</SwmToken>, we generate a unique request ID, wrap the incoming HTTP request into a standardized Request object, and add custom headers. Then, we call <SwmToken path="src/server-node.js" pos="1297:9:9" line-data="    const fRes = await handleRequest(util.mkFetchEvent(fReq));">`handleRequest`</SwmToken> from <SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath> to actually process the DNS-over-HTTPS logic using the prepared fetch event.

```javascript
async function handleHTTPRequest(b, req, res) {
  heartbeat();

  const rxid = util.xid();
  try {
    let host = req.headers.host || req.headers[":authority"];
    if (isIPv6(host)) host = `[${host}]`;

    // nb: req.url is a url-path, for ex: /a/b/c
    const fReq = new Request(new URL(req.url, `https://${host}`), {
      // Note: In a VM container, Object spread may not be working for all
      // properties, especially of "hidden" Symbol values!? like "headers"?
      ...req,
      // TODO: populate req ip in x-nile-client-ip header
      headers: util.concatHeaders(
        util.rxidHeader(rxid),
        nodeutil.copyNonPseudoHeaders(req.headers)
      ),
      method: req.method,
      body: req.method === "POST" ? b : null,
    });

    const fRes = await handleRequest(util.mkFetchEvent(fReq));

    if (!resOkay(res)) {
      throw new Error("res not writable 1");
    }

```

---

</SwmSnippet>

## Dispatching the DNS Request

<SwmSnippet path="/src/core/doh.js" line="25">

---

HandleRequest just forwards the fetch event to <SwmToken path="src/core/doh.js" pos="26:3:3" line-data="  return proxyRequest(event);">`proxyRequest`</SwmToken>, which does all the heavy lifting for DNS-over-HTTPS request handling.

```javascript
export function handleRequest(event) {
  return proxyRequest(event);
}
```

---

</SwmSnippet>

## Processing and Routing the DNS Query

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Receive DNS-over-HTTPS request"] --> node2{"Is request OPTIONS or HEAD?"}
  click node1 openCode "src/core/doh.js:33:34"
  node2 -->|"Yes"| node5["Respond with CORS headers"]
  click node2 openCode "src/core/doh.js:34:35"
  node2 -->|"No"| node3{"Did plugin set early response?"}
  click node5 openCode "src/core/doh.js:59:59"
  node3 -->|"Yes"| node5
  click node3 openCode "src/core/doh.js:45:47"
  node3 -->|"No"| node4["Executing Plugin Operations"]
  
  node4 --> node5["Respond with CORS headers"]
  click node5 openCode "src/core/doh.js:59:59"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
click node4 goToHeading "Executing Plugin Operations"
node4:::HeadingStyle

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Receive DNS-over-HTTPS request"] --> node2{"Is request OPTIONS or HEAD?"}
%%   click node1 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:33:34"
%%   node2 -->|"Yes"| node5["Respond with CORS headers"]
%%   click node2 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:34:35"
%%   node2 -->|"No"| node3{"Did plugin set early response?"}
%%   click node5 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:59:59"
%%   node3 -->|"Yes"| node5
%%   click node3 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:45:47"
%%   node3 -->|"No"| node4["Executing Plugin Operations"]
%%   
%%   node4 --> node5["Respond with CORS headers"]
%%   click node5 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:59:59"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
%% click node4 goToHeading "Executing Plugin Operations"
%% node4:::HeadingStyle
```

<SwmSnippet path="/src/core/doh.js" line="33">

---

In <SwmToken path="src/core/doh.js" pos="33:4:4" line-data="async function proxyRequest(event) {">`proxyRequest`</SwmToken>, we skip OPTIONS and HEAD requests, set up <SwmToken path="src/core/doh.js" pos="37:9:9" line-data="  const io = new IOState();">`IOState`</SwmToken> for tracking, and initialize the <SwmToken path="src/core/doh.js" pos="41:9:9" line-data="    const plugin = new RethinkPlugin(event);">`RethinkPlugin`</SwmToken>. If the plugin sets an early response, we return it; otherwise, we execute the plugin logic to process the DNS query.

```javascript
async function proxyRequest(event) {
  if (optionsRequest(event.request)) return util.respond204();
  if (headRequest(event.request)) return util.respond204();

  const io = new IOState();
  const ua = event.request.headers.get("User-Agent");

  try {
    const plugin = new RethinkPlugin(event);
    await plugin.initIoState(io);

    // if an early response has been set by plugin.initIoState, return it
    if (io.httpResponse) {
      return withCors(io, ua);
    }

    await util.timedSafeAsyncOp(
      /* op*/ () => plugin.execute(),
```

---

</SwmSnippet>

### Executing Plugin Operations

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Start plugin execution"]
    click node1 openCode "src/core/plugin.js:143:160"
    subgraph loop1["For each plugin"]
      node1 --> node2{"stopProcessing & not allowed to continue?"}
      click node2 openCode "src/core/plugin.js:147:149"
      node2 -->|"Yes"| node7["Next plugin"]
      node2 -->|"No"| node3{"exception & should bail?"}
      click node3 openCode "src/core/plugin.js:150:152"
      node3 -->|"Yes"| node7
      node3 -->|"No"| node4["Run plugin"]
      click node4 openCode "src/core/plugin.js:154:154"
      node4 --> node5{"Has callback?"}
      click node5 openCode "src/core/plugin.js:156:158"
      node5 -->|"Yes"| node6["Run callback"]
      click node6 openCode "src/core/plugin.js:157:158"
      node6 --> node7
      node5 -->|"No"| node7
      node7["Next plugin"] --> node2
    end
    node7 --> node8["End of plugin execution"]
    click node8 openCode "src/core/plugin.js:160:160"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["Start plugin execution"]
%%     click node1 openCode "<SwmPath>[src/core/plugin.js](src/core/plugin.js)</SwmPath>:143:160"
%%     subgraph loop1["For each plugin"]
%%       node1 --> node2{"<SwmToken path="src/core/plugin.js" pos="147:6:6" line-data="      if (io.stopProcessing &amp;&amp; !p.continueOnStopProcess) {">`stopProcessing`</SwmToken> & not allowed to continue?"}
%%       click node2 openCode "<SwmPath>[src/core/plugin.js](src/core/plugin.js)</SwmPath>:147:149"
%%       node2 -->|"Yes"| node7["Next plugin"]
%%       node2 -->|"No"| node3{"exception & should bail?"}
%%       click node3 openCode "<SwmPath>[src/core/plugin.js](src/core/plugin.js)</SwmPath>:150:152"
%%       node3 -->|"Yes"| node7
%%       node3 -->|"No"| node4["Run plugin"]
%%       click node4 openCode "<SwmPath>[src/core/plugin.js](src/core/plugin.js)</SwmPath>:154:154"
%%       node4 --> node5{"Has callback?"}
%%       click node5 openCode "<SwmPath>[src/core/plugin.js](src/core/plugin.js)</SwmPath>:156:158"
%%       node5 -->|"Yes"| node6["Run callback"]
%%       click node6 openCode "<SwmPath>[src/core/plugin.js](src/core/plugin.js)</SwmPath>:157:158"
%%       node6 --> node7
%%       node5 -->|"No"| node7
%%       node7["Next plugin"] --> node2
%%     end
%%     node7 --> node8["End of plugin execution"]
%%     click node8 openCode "<SwmPath>[src/core/plugin.js](src/core/plugin.js)</SwmPath>:160:160"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/core/plugin.js" line="143">

---

Execute iterates plugins, runs their exec logic with context, and skips or continues based on processing state.

```javascript
  async execute() {
    const io = this.io;
    // const rxid = this.ctx.get("rxid");
    for (const p of this.plugin) {
      if (io.stopProcessing && !p.continueOnStopProcess) {
        continue;
      }
      if (io.isException && p.bailOnException) {
        continue;
      }

      const res = await p.module.exec(makectx(this.ctx, p.pctx));

      if (typeof p.callback === "function") {
        await p.callback.call(this, res, io);
      }
    }
  }
```

---

</SwmSnippet>

### Authenticating and Loading User Data

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Prepare empty response"]
  click node1 openCode "src/plugins/users/user-op.js:35:36"
  node1 --> node2["Authenticate user operation"]
  click node2 openCode "src/plugins/users/auth-token.js:63:101"
  node2 --> node3{"Authentication successful?"}
  click node3 openCode "src/plugins/users/user-op.js:39:39"
  node3 -->|"Yes"| node4["Load user data"]
  click node4 openCode "src/plugins/users/user-op.js:42:42"
  node3 -->|"No"| node5["Set authentication error"]
  click node5 openCode "src/plugins/users/user-op.js:40:41"
  node4 --> node6["Attach authentication result"]
  click node6 openCode "src/plugins/users/user-op.js:44:44"
  node5 --> node6
  node6 --> node7["Return response"]
  click node7 openCode "src/plugins/users/user-op.js:49:50"
  node2 -.-> node8["Exception occurs?"]
  click node8 openCode "src/plugins/users/user-op.js:45:47"
  node8 --> node9["Set error response"]
  click node9 openCode "src/plugins/users/user-op.js:46:47"
  node9 --> node7
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Prepare empty response"]
%%   click node1 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:35:36"
%%   node1 --> node2["Authenticate user operation"]
%%   click node2 openCode "<SwmPath>[src/…/users/auth-token.js](src/plugins/users/auth-token.js)</SwmPath>:63:101"
%%   node2 --> node3{"Authentication successful?"}
%%   click node3 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:39:39"
%%   node3 -->|"Yes"| node4["Load user data"]
%%   click node4 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:42:42"
%%   node3 -->|"No"| node5["Set authentication error"]
%%   click node5 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:40:41"
%%   node4 --> node6["Attach authentication result"]
%%   click node6 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:44:44"
%%   node5 --> node6
%%   node6 --> node7["Return response"]
%%   click node7 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:49:50"
%%   node2 -.-> node8["Exception occurs?"]
%%   click node8 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:45:47"
%%   node8 --> node9["Set error response"]
%%   click node9 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:46:47"
%%   node9 --> node7
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/plugins/users/user-op.js" line="34">

---

In exec, we start with an empty response and call <SwmToken path="src/plugins/users/user-op.js" pos="38:9:11" line-data="      const out = await token.auth(ctx.rxid, ctx.request.url);">`token.auth`</SwmToken> to check if the request is authorized. The result determines if we load user data or return an error. Next up is the actual auth logic in <SwmPath>[src/…/users/auth-token.js](src/plugins/users/auth-token.js)</SwmPath>.

```javascript
  async exec(ctx) {
    let res = pres.emptyResponse();

    try {
      const out = await token.auth(ctx.rxid, ctx.request.url);
```

---

</SwmSnippet>

<SwmSnippet path="/src/plugins/users/auth-token.js" line="63">

---

Auth checks access keys from the environment, extracts a message key from the URL, and compares generated keys for each domain against the access keys. If any match, it passes; otherwise, it logs partials and fails.

```javascript
export async function auth(rxid, url) {
  const accesskeys = envutil.accessKeys();

  // empty access key, allow all
  if (util.emptySet(accesskeys)) {
    return Outcome.none();
  }
  const msg = rdnsutil.msgkeyFromUrl(url);
  // if missing msg-key in url, deny
  if (util.emptyString(msg)) {
    log.w(rxid, "auth: stop! missing access-key in", url);
    return Outcome.miss();
  }

  let ok = false;
  let a6 = "";
  // eval [s2.domain.tld, domain.tld] from a hostname
  // like s0.s1.s2.domain.tld
  for (const dom of util.domains(url)) {
    if (util.emptyString(dom)) continue;

    const [hex, hexcat] = await gen(msg, dom);

    log.d(rxid, msg, dom, "<= msg/h :auth: hex/k =>", hexcat, accesskeys);

    // allow if access-key (upto its full len) matches calculated hex
    for (const ak of accesskeys) {
      ok = hexcat.startsWith(ak);
      if (ok) {
        return Outcome.pass();
      } else {
        const [d, h] = ak.split(akdelim);
        a6 += d + akdelim + h.slice(0, 6) + " ";
      }
    }

    const h6 = dom + akdelim + hex.slice(0, 6);
    log.w(rxid, "auth: key mismatch want:", a6, "have:", h6);
  }
```

---

</SwmSnippet>

<SwmSnippet path="/src/plugins/users/user-op.js" line="39">

---

We just got back from <SwmPath>[src/…/users/auth-token.js](src/plugins/users/auth-token.js)</SwmPath>. If auth failed, exec returns an error response; if it passed, it loads user data and attaches auth info to the response. This wraps up the <SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath> logic.

```javascript
      if (!out.ok) {
        res = pres.errResponse("UserOp:Auth", new Error("auth failed"));
      } else {
        res = this.loadUser(ctx);
      }
      res.data.userAuth = out;
    } catch (ex) {
      res = pres.errResponse("UserOp", ex);
    }

    return res;
  }
```

---

</SwmSnippet>

### Extracting User and DNS Context

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Receive DNS request"] --> node2{"Is DNS message?"}
  click node1 openCode "src/plugins/users/user-op.js:56:57"
  click node2 openCode "src/plugins/users/user-op.js:59:62"
  node2 -->|"No"| node3["Return empty response"]
  click node3 openCode "src/plugins/users/user-op.js:57:62"
  node2 -->|"Yes"| node4["Process domains for delegation"]
  click node4 openCode "src/plugins/users/user-op.js:65:72"
  subgraph loop1["For each domain in DNS packet"]
    node4 --> node5{"Is domain delegated?"}
    click node5 openCode "src/plugins/users/user-op.js:68:71"
    node5 -->|"Yes"| node6["Set resolver preference in response"]
    click node6 openCode "src/plugins/users/user-op.js:70:71"
    node5 -->|"No"| node4
  end
  node4 --> node7["Check blocklist flag in request URL"]
  click node7 openCode "src/plugins/users/user-op.js:74:76"
  node7 --> node8{"Is blocklist flag present?"}
  click node8 openCode "src/plugins/users/user-op.js:75:76"
  node8 -->|"No"| node9["Return response"]
  click node9 openCode "src/plugins/users/user-op.js:109:110"
  node8 -->|"Yes"| node10{"Is blocklist data available?"}
  click node10 openCode "src/plugins/users/user-op.js:80:81"
  node10 -->|"No"| node11["Retrieve blocklist data from flag"]
  click node11 openCode "src/plugins/users/user-op.js:84:85"
  node11 --> node12{"Is blocklist data now available?"}
  click node12 openCode "src/plugins/users/user-op.js:87:93"
  node12 -->|"Yes"| node13["Enrich response with blocklist info and flag"]
  click node13 openCode "src/plugins/users/user-op.js:99:102"
  node12 -->|"No"| node9
  node10 -->|"Yes"| node13
  node13 --> node9
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Receive DNS request"] --> node2{"Is DNS message?"}
%%   click node1 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:56:57"
%%   click node2 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:59:62"
%%   node2 -->|"No"| node3["Return empty response"]
%%   click node3 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:57:62"
%%   node2 -->|"Yes"| node4["Process domains for delegation"]
%%   click node4 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:65:72"
%%   subgraph loop1["For each domain in DNS packet"]
%%     node4 --> node5{"Is domain delegated?"}
%%     click node5 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:68:71"
%%     node5 -->|"Yes"| node6["Set resolver preference in response"]
%%     click node6 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:70:71"
%%     node5 -->|"No"| node4
%%   end
%%   node4 --> node7["Check blocklist flag in request URL"]
%%   click node7 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:74:76"
%%   node7 --> node8{"Is blocklist flag present?"}
%%   click node8 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:75:76"
%%   node8 -->|"No"| node9["Return response"]
%%   click node9 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:109:110"
%%   node8 -->|"Yes"| node10{"Is blocklist data available?"}
%%   click node10 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:80:81"
%%   node10 -->|"No"| node11["Retrieve blocklist data from flag"]
%%   click node11 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:84:85"
%%   node11 --> node12{"Is blocklist data now available?"}
%%   click node12 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:87:93"
%%   node12 -->|"Yes"| node13["Enrich response with blocklist info and flag"]
%%   click node13 openCode "<SwmPath>[src/…/users/user-op.js](src/plugins/users/user-op.js)</SwmPath>:99:102"
%%   node12 -->|"No"| node9
%%   node10 -->|"Yes"| node13
%%   node13 --> node9
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/plugins/users/user-op.js" line="56">

---

In <SwmToken path="src/plugins/users/user-op.js" pos="56:1:1" line-data="  loadUser(ctx) {">`loadUser`</SwmToken>, we bail early if the context isn't a DNS message. For DNS queries, we extract domains and check for delegation, setting the resolver URL if needed. Next, we handle blocklist flags and user config.

```javascript
  loadUser(ctx) {
    const response = pres.emptyResponse();

    if (!ctx.isDnsMsg) {
      this.log.w(ctx.rxid, "not a dns-msg, ignore");
      return response;
    }

    try {
      const dnsPacket = ctx.requestDecodedDnsPacket;
      const domains = dnsutil.extractDomains(dnsPacket);
      for (const d of domains) {
        if (delegated.has(d)) {
          // may be overriden by user-preferred doh upstream
          response.data.dnsResolverUrl = envutil.primaryDohResolver();
        }
      }
```

---

</SwmSnippet>

<SwmSnippet path="/src/plugins/users/user-op.js" line="74">

---

After handling delegation, <SwmToken path="src/plugins/users/user-op.js" pos="104:13:13" line-data="      this.log.e(ctx.rxid, &quot;loadUser&quot;, e);">`loadUser`</SwmToken> extracts the blocklist flag from the URL, checks the cache, and decodes it if needed. If valid, it attaches user blocklist info and the flag to the response data.

```javascript
      const blocklistFlag = rdnsutil.blockstampFromUrl(ctx.request.url);
      const hasflag = !util.emptyString(blocklistFlag);
      if (!hasflag) {
        this.log.d(ctx.rxid, "empty blocklist-flag", ctx.request.url);
      }
      // blocklistFlag may be invalid, ref rdnsutil.blockstampFromUrl
      let r = this.userConfigCache.get(blocklistFlag);
      let hasdata = rdnsutil.hasBlockstamp(r);
      if (hasflag && !hasdata) {
        // r not in cache
        r = rdnsutil.unstamp(blocklistFlag); // r is never null, may throw ex
        hasdata = rdnsutil.hasBlockstamp(r);

        if (hasdata) {
          this.log.d(ctx.rxid, "new cfg cache kv", blocklistFlag, r);
          // TODO: blocklistFlag is not normalized, ie b32 used for dot isn't
          // converted to its b64 form (which doh and rethinkdns modules use)
          // example, b32: 1-AABABAA / equivalent b64: 1:AAIAgA==
          this.userConfigCache.put(blocklistFlag, r);
        }
      } else {
        this.log.d(ctx.rxid, "cfg cache hit?", hasdata, blocklistFlag, r);
      }

      if (hasdata) {
        response.data.userBlocklistInfo = r;
        response.data.userBlocklistFlag = blocklistFlag;
        // TODO: override response.data.dnsResolverUrl
      }
    } catch (e) {
      this.log.e(ctx.rxid, "loadUser", e);
      // avoid erroring out on invalid blocklist info & flag
      // response = pres.errResponse("UserOp:loadUser", e);
    }

    return response;
  }
```

---

</SwmSnippet>

### Finalizing and Returning the DNS Response

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Handle DNS-over-HTTPS request"] --> node2{"Timeout or error during DNS request? (waitMs)"}
    click node1 openCode "src/core/doh.js:51:53"
    node2 -->|"Yes"| node3["Generate error response"]
    click node2 openCode "src/core/doh.js:52:56"
    node2 -->|"No"| node4["Generate DNS response"]
    click node3 openCode "src/core/doh.js:52:56"
    click node4 openCode "src/core/doh.js:51:53"
    node3 --> node5["Apply CORS headers and return response"]
    node4 --> node5
    click node5 openCode "src/core/doh.js:59:60"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["Handle DNS-over-HTTPS request"] --> node2{"Timeout or error during DNS request? (<SwmToken path="src/core/doh.js" pos="51:3:3" line-data="      /* waitMs*/ dnsutil.requestTimeout(),">`waitMs`</SwmToken>)"}
%%     click node1 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:51:53"
%%     node2 -->|"Yes"| node3["Generate error response"]
%%     click node2 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:52:56"
%%     node2 -->|"No"| node4["Generate DNS response"]
%%     click node3 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:52:56"
%%     click node4 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:51:53"
%%     node3 --> node5["Apply CORS headers and return response"]
%%     node4 --> node5
%%     click node5 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:59:60"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/core/doh.js" line="51">

---

We just got back from <SwmPath>[src/core/plugin.js](src/core/plugin.js)</SwmPath>. <SwmToken path="src/core/doh.js" pos="26:3:3" line-data="  return proxyRequest(event);">`proxyRequest`</SwmToken> wraps up by returning the response with CORS headers, or calls <SwmToken path="src/core/doh.js" pos="52:15:15" line-data="      /* onTimeout*/ () =&gt; Promise.resolve(errorResponse(io))">`errorResponse`</SwmToken> if something failed. This is the last step before sending the response back.

```javascript
      /* waitMs*/ dnsutil.requestTimeout(),
      /* onTimeout*/ () => Promise.resolve(errorResponse(io))
    );
  } catch (err) {
    log.e("doh", "proxy-request error", err.stack);
    errorResponse(io, err);
  }

  return withCors(io, ua);
}
```

---

</SwmSnippet>

## Handling DNS Exceptions

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Receive DNS-over-HTTPS request with possible error"]
    click node1 openCode "src/core/doh.js:75:78"
    node1 --> node2{"Is error information provided?"}
    click node2 openCode "src/core/doh.js:75:78"
    node2 -->|"Yes"| node3["Generate error response including error details"]
    click node3 openCode "src/core/doh.js:76:77"
    node2 -->|"No"| node4["Generate generic error response"]
    click node4 openCode "src/core/doh.js:76:77"
    node3 --> node5["Send error response to requester"]
    click node5 openCode "src/core/doh.js:77:78"
    node4 --> node5
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["Receive DNS-over-HTTPS request with possible error"]
%%     click node1 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:75:78"
%%     node1 --> node2{"Is error information provided?"}
%%     click node2 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:75:78"
%%     node2 -->|"Yes"| node3["Generate error response including error details"]
%%     click node3 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:76:77"
%%     node2 -->|"No"| node4["Generate generic error response"]
%%     click node4 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:76:77"
%%     node3 --> node5["Send error response to requester"]
%%     click node5 openCode "<SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>:77:78"
%%     node4 --> node5
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/core/doh.js" line="75">

---

ErrorResponse creates an error response object and calls <SwmToken path="src/core/doh.js" pos="77:3:3" line-data="  io.dnsExceptionResponse(eres);">`dnsExceptionResponse`</SwmToken> on <SwmToken path="src/core/doh.js" pos="37:9:9" line-data="  const io = new IOState();">`IOState`</SwmToken> to build and log the DNS exception details.

```javascript
function errorResponse(io, err = null) {
  const eres = pres.errResponse("doh.js", err);
  io.dnsExceptionResponse(eres);
}
```

---

</SwmSnippet>

## Building the DNS Exception Response

<SwmSnippet path="/src/core/io-state.js" line="79">

---

In <SwmToken path="src/core/io-state.js" pos="79:1:1" line-data="  dnsExceptionResponse(res) {">`dnsExceptionResponse`</SwmToken>, we set flags for error state, extract exception info, generate a SERVFAIL DNS packet, decode it, log the packet, and build the HTTP response with the right status code.

```javascript
  dnsExceptionResponse(res) {
    this.initDecodedDnsPacketIfNeeded();

    this.stopProcessing = true;
    this.isException = true;

    if (util.emptyObj(res)) {
      this.exceptionStack = "no-res";
      this.exceptionFrom = "no-res";
    } else {
      this.exceptionStack = res.exceptionStack || "no-stack";
      this.exceptionFrom = res.exceptionFrom || "no-origin";
    }

    try {
      const qid = this.decodedDnsPacket.id; // may be null
      const questions = this.decodedDnsPacket.questions; // may be null
      const servfail = dnsutil.servfail(qid, questions); // may be empty
      const hasServfail = !bufutil.emptyBuf(servfail);
      const ex = {
        exceptionFrom: this.exceptionFrom,
        exceptionStack: this.exceptionStack,
      };

      if (hasServfail) {
        // TODO: try-catch as decode may throw?
        this.decodedDnsPacket = dnsutil.decode(servfail);
      }

      this.logDnsPkt();
```

---

</SwmSnippet>

### Logging DNS Packet Details

<SwmSnippet path="/src/core/io-state.js" line="175">

---

LogDnsPkt dumps domains, query type, answer data, and TTL from the decoded DNS packet for debugging. Next, we call <SwmPath>[src/commons/dnsutil.js](src/commons/dnsutil.js)</SwmPath> to extract answer data.

```javascript
  logDnsPkt() {
    if (this.isProd) return;
    this.log.d(
      "domains",
      dnsutil.extractDomains(this.decodedDnsPacket),
      dnsutil.getQueryType(this.decodedDnsPacket) || "",
      "data",
      dnsutil.getInterestingAnswerData(this.decodedDnsPacket),
      dnsutil.ttl(this.decodedDnsPacket)
    );
  }
```

---

</SwmSnippet>

### Extracting Key DNS Answer Data

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1{"Does DNS packet have answers?"}
  click node1 openCode "src/commons/dnsutil.js:435:436"
  node1 -->|"No"| node2["Return error code or status"]
  click node2 openCode "src/commons/dnsutil.js:436:437"
  node1 -->|"Yes"| node3["Summarize answer data"]
  click node3 openCode "src/commons/dnsutil.js:439:441"

  subgraph loop1["For each answer in DNS packet"]
    node3 --> node4{"What is the answer type?"}
    click node4 openCode "src/commons/dnsutil.js:442:525"
    node4 -->|"IP (A/AAAA)"| node5["Prepend IP to summary and set flag"]
    click node5 openCode "src/commons/dnsutil.js:450:455"
    node4 -->|"NS/TXT/OPTION"| node6["Append data to summary"]
    click node6 openCode "src/commons/dnsutil.js:455:461"
    node4 -->|"SOA/SRV/CAA/MX/CNAME"| node7["Append specific field to summary"]
    click node7 openCode "src/commons/dnsutil.js:461:477,519:520"
    node4 -->|"HINFO/RP/DNSKEY/DS/RRSIG/Other"| node8["Append field and break loop"]
    click node8 openCode "src/commons/dnsutil.js:464:467,479:481,508:510,512:514,516:518,523:524"
    node4 -->|"HTTPS"| node9["Special: Append target or IP hints"]
    click node9 openCode "src/commons/dnsutil.js:481:506"
    node5 --> node10{"Enough data collected?"}
    node6 --> node10
    node7 --> node10
    node9 --> node10
    node10 -->|"Yes"| node11["Break loop"]
    node10 -->|"No"| node3
    node8 --> node11
  end
  node3 --> node12["Truncate summary to maxlen and return"]
  click node12 openCode "src/commons/dnsutil.js:527:529"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1{"Does DNS packet have answers?"}
%%   click node1 openCode "<SwmPath>[src/commons/dnsutil.js](src/commons/dnsutil.js)</SwmPath>:435:436"
%%   node1 -->|"No"| node2["Return error code or status"]
%%   click node2 openCode "<SwmPath>[src/commons/dnsutil.js](src/commons/dnsutil.js)</SwmPath>:436:437"
%%   node1 -->|"Yes"| node3["Summarize answer data"]
%%   click node3 openCode "<SwmPath>[src/commons/dnsutil.js](src/commons/dnsutil.js)</SwmPath>:439:441"
%% 
%%   subgraph loop1["For each answer in DNS packet"]
%%     node3 --> node4{"What is the answer type?"}
%%     click node4 openCode "<SwmPath>[src/commons/dnsutil.js](src/commons/dnsutil.js)</SwmPath>:442:525"
%%     node4 -->|"IP (<SwmToken path="src/core/io-state.js" pos="158:13:15" line-data="    // gw responses only assigned on A/AAAA/HTTPS/SVCB records">`A/AAAA`</SwmToken>)"| node5["Prepend IP to summary and set flag"]
%%     click node5 openCode "<SwmPath>[src/commons/dnsutil.js](src/commons/dnsutil.js)</SwmPath>:450:455"
%%     node4 -->|"NS/TXT/OPTION"| node6["Append data to summary"]
%%     click node6 openCode "<SwmPath>[src/commons/dnsutil.js](src/commons/dnsutil.js)</SwmPath>:455:461"
%%     node4 -->|"SOA/SRV/CAA/MX/CNAME"| node7["Append specific field to summary"]
%%     click node7 openCode "<SwmPath>[src/commons/dnsutil.js](src/commons/dnsutil.js)</SwmPath>:461:477,519:520"
%%     node4 -->|"HINFO/RP/DNSKEY/DS/RRSIG/Other"| node8["Append field and break loop"]
%%     click node8 openCode "<SwmPath>[src/commons/dnsutil.js](src/commons/dnsutil.js)</SwmPath>:464:467,479:481,508:510,512:514,516:518,523:524"
%%     node4 -->|"HTTPS"| node9["Special: Append target or IP hints"]
%%     click node9 openCode "<SwmPath>[src/commons/dnsutil.js](src/commons/dnsutil.js)</SwmPath>:481:506"
%%     node5 --> node10{"Enough data collected?"}
%%     node6 --> node10
%%     node7 --> node10
%%     node9 --> node10
%%     node10 -->|"Yes"| node11["Break loop"]
%%     node10 -->|"No"| node3
%%     node8 --> node11
%%   end
%%   node3 --> node12["Truncate summary to maxlen and return"]
%%   click node12 openCode "<SwmPath>[src/commons/dnsutil.js](src/commons/dnsutil.js)</SwmPath>:527:529"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/commons/dnsutil.js" line="434">

---

GetInterestingAnswerData scans DNS answers, grabs <SwmToken path="src/commons/dnsutil.js" pos="446:5:5" line-data="    // capturing IPs in A / AAAA records appearing later in ans">`IPs`</SwmToken> first, and breaks early if enough data is found. It handles each answer type specifically, then truncates and cleans up the result for logging or display.

```javascript
export function getInterestingAnswerData(packet, maxlen = 80, delim = "|") {
  if (!hasAnswers(packet)) {
    return !util.emptyObj(packet) ? packet.rcode || "WTF1" : "WTF2";
  }

  // set to true if at least one ip has been captured from ans
  let atleastoneip = false;
  let str = "";
  for (const a of packet.answers) {
    // gather twice the maxlen to capture as much as possible:
    // ips are usually prepend to the front, and going 2 times
    // over maxlen (chosen arbitrarily) maximises chances of
    // capturing IPs in A / AAAA records appearing later in ans
    if (atleastoneip && str.length > maxlen) break;
    if (!atleastoneip && str.length > maxlen * 2) break;

    if (isAnswerA(a) || isAnswerAAAA(a)) {
      const dat = a.data || "";
      // prepend A / AAAA data
      if (!util.emptyString(dat)) str = dat + delim + str;
      atleastoneip = true;
    } else if (isAnswerOPTION(a) || isAnswerNS(a) || isAnswerTXT(a)) {
      // ns: github.com/mafintosh/dns-packet/blob/31d3caf3/index.js#L249
      // txt: github.com/mafintosh/dns-packet/blob/31d3caf3/index.js#L370
      // opt: github.com/mafintosh/dns-packet/blob/31d3caf3/index.js#L773
      const dat = a.data || "";
      if (!util.emptyString(dat)) str += dat + delim;
    } else if (isAnswerSOA(a)) {
      // github.com/mafintosh/dns-packet/blob/31d3caf3/index.js#L284
      str += a.data.mname + delim;
    } else if (isAnswerHINFO(a)) {
      // github.com/mafintosh/dns-packet/blob/31d3caf3/index.js#L450
      str += a.data.os + delim;
      break;
    } else if (isAnswerSRV(a)) {
      // github.com/mafintosh/dns-packet/blob/31d3caf3/index.js#L521
      str += a.data.target + delim;
    } else if (isAnswerCAA(a)) {
      // github.com/mafintosh/dns-packet/blob/31d3caf3/index.js#L574
      str += a.data.value + delim;
    } else if (isAnswerMX(a)) {
      // github.com/mafintosh/dns-packet/blob/31d3caf3/index.js#L618
      str += a.data.exchange + delim;
    } else if (isAnswerRP(a)) {
      // github.com/mafintosh/dns-packet/blob/31d3caf3/index.js#L1027
      str += a.data.mbox + delim;
      break;
    } else if (isAnswerHttps(a)) {
      // https/svcb answers may have a A / AAAA records
      // github.com/serverless-dns/dns-parser/blob/b7d73b3d/index.js#L1381
      const t = a.data.targetName;
      const kv = a.data.svcParams;
      if (t === ".") {
        if (util.emptyObj(kv)) continue;
        // if svcb/https is self-referential, then prepend ip hints, if any
        if (
          !util.emptyArray(kv.ipv4hint) &&
          !util.emptyString(kv.ipv4hint[0])
        ) {
          str = kv.ipv4hint[0] + delim + str;
          atleastoneip = true;
        }
        if (
          !util.emptyArray(kv.ipv6hint) &&
          !util.emptyString(kv.ipv6hint[0])
        ) {
          str = kv.ipv6hint[0] + delim + str;
          atleastoneip = true;
        }
      } else {
        str += t + delim;
      }
    } else if (isAnswerDNSKEY(a)) {
      // github.com/mafintosh/dns-packet/blob/31d3caf3/index.js#L914
      str += bufutil.bytesToBase64Url(a.data.key) + delim;
      break;
    } else if (isAnswerDS(a)) {
      // ds: github.com/mafintosh/dns-packet/blob/31d3caf3/index.js#L1279
      str += bufutil.bytesToBase64Url(a.data.digest) + delim;
      break;
    } else if (isAnswerRRSIG(a)) {
      // rrsig: github.com/mafintosh/dns-packet/blob/31d3caf3/index.js#L984
      str += bufutil.bytesToBase64Url(a.data.signature) + delim;
      break;
    } else if (isAnswerCname(a)) {
      str += a.data + delim;
    } else {
      // unhanlded types:
      // null, ptr, ds, nsec, nsec3, nsec3param, tlsa, sshfp, spf, dname
      break;
    }
  }
```

---

</SwmSnippet>

<SwmSnippet path="/src/commons/dnsutil.js" line="527">

---

After extracting answer data, we truncate the result to maxlen, then cut off at the last delimiter for clean output. This keeps logs and displays tidy.

```javascript
  const trunc = util.strstr(str, 0, maxlen);
  const idx = trunc.lastIndexOf(delim);
  return idx >= 0 ? util.strstr(trunc, 0, idx) : trunc;
}
```

---

</SwmSnippet>

### Setting the HTTP Error Response

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["DNS exception occurs"] --> node2{"Is this a SERVFAIL exception?"}
  click node1 openCode "src/core/io-state.js:109:134"
  node2 -->|"Yes"| node3["Return HTTP response with SERVFAIL (status 200 if SERVFAIL, 408 otherwise)"]
  click node2 openCode "src/core/io-state.js:114:115"
  click node3 openCode "src/core/io-state.js:109:115"
  node2 -->|"No"| node4["Log exception details"]
  click node4 openCode "src/core/io-state.js:118:119"
  node4 --> node5{"Is exceptionStack no-res or no-stack?"}
  click node5 openCode "src/core/io-state.js:120:125"
  node5 -->|"Yes"| node6["Update exceptionStack and exceptionFrom"]
  click node6 openCode "src/core/io-state.js:123:125"
  node6 --> node7["Return HTTP response with Service Unavailable (status 503)"]
  click node7 openCode "src/core/io-state.js:126:132"
  node5 -->|"No"| node7
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["DNS exception occurs"] --> node2{"Is this a SERVFAIL exception?"}
%%   click node1 openCode "<SwmPath>[src/core/io-state.js](src/core/io-state.js)</SwmPath>:109:134"
%%   node2 -->|"Yes"| node3["Return HTTP response with SERVFAIL (status 200 if SERVFAIL, 408 otherwise)"]
%%   click node2 openCode "<SwmPath>[src/core/io-state.js](src/core/io-state.js)</SwmPath>:114:115"
%%   click node3 openCode "<SwmPath>[src/core/io-state.js](src/core/io-state.js)</SwmPath>:109:115"
%%   node2 -->|"No"| node4["Log exception details"]
%%   click node4 openCode "<SwmPath>[src/core/io-state.js](src/core/io-state.js)</SwmPath>:118:119"
%%   node4 --> node5{"Is <SwmToken path="src/core/io-state.js" pos="86:3:3" line-data="      this.exceptionStack = &quot;no-res&quot;;">`exceptionStack`</SwmToken> <SwmToken path="src/core/io-state.js" pos="86:8:10" line-data="      this.exceptionStack = &quot;no-res&quot;;">`no-res`</SwmToken> or <SwmToken path="src/core/io-state.js" pos="89:14:16" line-data="      this.exceptionStack = res.exceptionStack || &quot;no-stack&quot;;">`no-stack`</SwmToken>?"}
%%   click node5 openCode "<SwmPath>[src/core/io-state.js](src/core/io-state.js)</SwmPath>:120:125"
%%   node5 -->|"Yes"| node6["Update <SwmToken path="src/core/io-state.js" pos="86:3:3" line-data="      this.exceptionStack = &quot;no-res&quot;;">`exceptionStack`</SwmToken> and <SwmToken path="src/core/io-state.js" pos="87:3:3" line-data="      this.exceptionFrom = &quot;no-res&quot;;">`exceptionFrom`</SwmToken>"]
%%   click node6 openCode "<SwmPath>[src/core/io-state.js](src/core/io-state.js)</SwmPath>:123:125"
%%   node6 --> node7["Return HTTP response with Service Unavailable (status 503)"]
%%   click node7 openCode "<SwmPath>[src/core/io-state.js](src/core/io-state.js)</SwmPath>:126:132"
%%   node5 -->|"No"| node7
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/core/io-state.js" line="109">

---

We just finished <SwmToken path="src/core/io-state.js" pos="118:8:8" line-data="      this.log.e(&quot;dnsExceptionResponse&quot;, pktjson, e.stack);">`dnsExceptionResponse`</SwmToken>. Based on whether SERVFAIL is present, we set the HTTP response status to 200 or 408, and fall back to 503 if something blows up. Headers include debug info for traceability.

```javascript
      this.httpResponse = new Response(servfail, {
        headers: util.concatHeaders(
          this.headers(servfail),
          this.debugHeaders(JSON.stringify(ex))
        ),
        status: hasServfail ? 200 : 408, // rfc8484 section-4.2.1
      });
    } catch (e) {
      const pktjson = JSON.stringify(this.decodedDnsPacket || {});
      this.log.e("dnsExceptionResponse", pktjson, e.stack);
      if (
        this.exceptionStack === "no-res" ||
        this.exceptionStack === "no-stack"
      ) {
        this.exceptionStack = e.stack;
        this.exceptionFrom = "IOState:errorResponse";
      }
      this.httpResponse = new Response(null, {
        headers: util.concatHeaders(
          this.headers(),
          this.debugHeaders(JSON.stringify(this.exceptionStack))
        ),
        status: 503,
      });
    }
  }
```

---

</SwmSnippet>

## Sending the Final Response to the Client

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Set HTTP status and headers"] --> node2{"Is response writable?"}
  click node1 openCode "src/server-node.js:1303:1304"
  node2 -->|"No"| node5["Handle error: Cannot write response"]
  click node2 openCode "src/server-node.js:1310:1311"
  node2 -->|"Yes"| node3{"Is response body size > 0?"}
  click node3 openCode "src/server-node.js:1312:1315"
  node3 -->|"Yes"| node4["Send response body"]
  click node4 openCode "src/server-node.js:1313:1314"
  node3 -->|"No"| node6["Send empty response"]
  click node6 openCode "src/server-node.js:1317:1318"
  node5 --> node7["Send 400 Bad Request, log error, close response"]
  click node7 openCode "src/server-node.js:1320:1325"
  node4 --> node8["Finish"]
  node6 --> node8
  node7 --> node8
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Set HTTP status and headers"] --> node2{"Is response writable?"}
%%   click node1 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1303:1304"
%%   node2 -->|"No"| node5["Handle error: Cannot write response"]
%%   click node2 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1310:1311"
%%   node2 -->|"Yes"| node3{"Is response body size > 0?"}
%%   click node3 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1312:1315"
%%   node3 -->|"Yes"| node4["Send response body"]
%%   click node4 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1313:1314"
%%   node3 -->|"No"| node6["Send empty response"]
%%   click node6 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1317:1318"
%%   node5 --> node7["Send 400 Bad Request, log error, close response"]
%%   click node7 openCode "<SwmPath>[src/server-node.js](src/server-node.js)</SwmPath>:1320:1325"
%%   node4 --> node8["Finish"]
%%   node6 --> node8
%%   node7 --> node8
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/server-node.js" line="1303">

---

We just got back from <SwmPath>[src/core/doh.js](src/core/doh.js)</SwmPath>. <SwmToken path="src/server-node.js" pos="1265:1:1" line-data="      handleHTTPRequest(b, req, res);">`handleHTTPRequest`</SwmToken> writes the response headers and body to the client, normalizing the buffer if there's data, or ending the response if not. Errors are handled with fallback status codes and logging.

```javascript
    res.writeHead(fRes.status, util.copyHeaders(fRes));

    // ans may be null on non-2xx responses, such as redirects (3xx) by cc.js
    // or 4xx responses on timeouts or 5xx on invalid http method
    const ans = await fRes.arrayBuffer();
    const sz = bufutil.len(ans);

    if (!resOkay(res)) {
      throw new Error("res not writable 2");
    } else if (sz > 0) {
      adjustTLSFragAfterWrites(res.socket, sz);
      res.end(bufutil.normalize8(ans));
    } else {
      // expect fRes.status to be set to non 2xx above
      res.end();
    }
  } catch (e) {
    const ok = resOkay(res);
    if (ok && !res.headersSent) res.writeHead(400); // bad request
    if (ok && !res.writableEnded) res.end();
    if (!ok) resClose(res);
    log.w(e);
  }
}
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBamF2YXNjcmlwdC1zZXJ2ZXJsZXNzLWRucyUzQSUzQXJpY2FyZG9sb3Blemc=" repo-name="javascript-serverless-dns"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
