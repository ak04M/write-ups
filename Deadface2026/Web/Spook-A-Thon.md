# DEADFACE CTF Write-up — Grave Encounters

**Target IP:** `170.187.156.131`  
**Web application:** Grave Encounters — Halloween Movie Marathon  
**Web port:** `8080`  
**WebSocket port:** `8765`  
**Flag format:** `deadface{flag-text}`

> **Note:** The challenge explicitly warns against using port `8443`, as it belongs to a different challenge.

## 1. Initial Nmap Scan

We started by scanning all TCP ports to discover the services running on the target.

**Command:**

```bash
nmap -Pn -sV --open -p- -T4 170.187.156.131
```

**Options explained:**

- `-Pn`: Skip host discovery and scan the target directly.
- `-sV`: Detect service versions.
- `--open`: Display open ports only.
- `-p-`: Scan all TCP ports.
- `-T4`: Use a faster timing profile.

**Finding:** During our enumeration, we identified the website running on port `8080`. The exact Nmap output was not captured in our conversation.

## 2. Retrieve the Homepage

We used `curl` to inspect the HTTP response and retrieve the HTML source.

**Command:**

```bash
curl -i http://170.187.156.131:8080/
```

**Observed output:**

```http
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.13.12
Date: Sun, 18 Oct 2026 18:40:25 GMT
Content-type: text/html
Content-Length: 4530
Last-Modified: Thu, 15 Oct 2026 02:24:02 GMT
```

The page title was:

```html
<title>Grave Encounters :: Halloween Movie Marathon</title>
```

The footer contained:

```html
Grave Encounters v0.3 &middot; built in a hurry by jerry &middot;
```

The page appeared to be a simple static HTML site served by Python's `SimpleHTTP` server.

## 3. Inspect the JavaScript

While examining the HTML source, we found a JavaScript block that revealed a WebSocket endpoint.

**Relevant source:**

```javascript
const WS_URL = (location.protocol === "https:" ? "wss://" : "ws://") + location.hostname + ":8765";
let ws;

function connect(){
  try {
    ws = new WebSocket(WS_URL);
    ws.onopen = () => { ws.send(JSON.stringify({cmd:"list"})); };
    ws.onmessage = (ev) => { /* lineup is static; socket reserved for member tools */ };
    ws.onclose = () => {};
    ws.onerror = () => {};
  } catch (e) { /* ignore */ }
}
connect();
```

**Findings:**

1. The WebSocket service was running on port `8765`.
2. The frontend automatically sent the JSON command `{"cmd":"list"}`.
3. The comment about member tools suggested that the WebSocket backend might expose additional functionality.

This gave us a new service to investigate.

## 4. Connect to the WebSocket

We initially checked whether `websocat` was installed, but it was not available on our Kali Linux system.

Instead, we used Python's `websockets` library.

**Library check:**

```bash
python3 -c "import websockets; print('websockets installed')"
```

We created a file named `ws_test.py` to connect to the WebSocket service and send the `list` command.

**Script:**

```python
import asyncio
import json
import websockets

async def main():
    uri = "ws://170.187.156.131:8765"

    async with websockets.connect(uri) as ws:
        await ws.send(json.dumps({"cmd": "list"}))

        try:
            response = await asyncio.wait_for(
                ws.recv(), timeout=5
            )
            print("Response:", response)
        except asyncio.TimeoutError:
            print("No response within 5 seconds")

asyncio.run(main())
```

**Execution:**

```bash
python3 ws_test.py
```

**Observed output:**

```text
Response: {"type": "welcome"}
```

The connection was successful, but our first script only printed the first received message.

## 5. Read Multiple WebSocket Messages

We modified `ws_test.py` to read multiple messages from the server.

The observed output was:

```text
python3 ws_test.py
Received: {"type": "welcome"}
Sent: {"cmd":"list"}
Received: {"type": "movies", "movies": [{"title": "Hereditary", "votes": 4}, {"title": "The Thing", "votes": 6}, {"title": "Scream", "votes": 3}, {"title": "Trick 'r Treat", "votes": 5}]}
No more messages.
```

The server returned the following movie data:

| Movie | Votes |
|---|---:|
| Hereditary | 4 |
| The Thing | 6 |
| Scream | 3 |
| Trick 'r Treat | 5 |

**Finding:** The `list` command was valid and returned the movie lineup.

## 6. Test Other Commands

We next tested `help`, `list`, and `secret` to understand the backend's command handling.

**Observed output:**

```text
python3 ws_test.py
Welcome: {"type": "welcome"}

Sent: {'cmd': 'help'}
Received: {"error": "unknown cmd: help"}

Sent: {'cmd': 'list'}
Received: {"type": "movies", "movies": [{"title": "Hereditary", "votes": 4}, {"title": "The Thing", "votes": 6}, {"title": "Scream", "votes": 3}, {"title": "Trick 'r Treat", "votes": 5}]}

Sent: {'cmd': 'secret'}
Received: {"error": "unknown cmd: secret"}
```

**Findings:**

- `list` was a valid command.
- `help` was rejected as an unknown command.
- `secret` was also rejected as an unknown command.

The server used a custom JSON-based protocol, so the correct way to retrieve the secret could not be determined simply by guessing the command name.

## 7. Enumerate Common Web Paths

We attempted to check common paths that might expose configuration files, source code, or backups.

**Command:**

```bash
BASE="http://170.187.156.131:8080"

for path in robots.txt sitemap.xml .git/HEAD \
            openapi.json swagger.json api docs \
            .env server-status; do
  echo "===== /$path ====="
  curl -sS -i --max-time 5 "$BASE/$path" | head -30
done
```

The command failed because the shell could not find `curl` and `head`.

**Observed errors included:**

```text
Command 'head' is available in the following places
 * /bin/head
 * /usr/bin/head
The command could not be located because '/bin:/usr/bin' is not included in the PATH environment variable.
head: command not found
```

```text
Command 'curl' is available in the following places
 * /bin/curl
 * /usr/bin/curl
The command could not be located because '/bin:/usr/bin' is not included in the PATH environment variable.
curl: command not found
```

We tried another set of paths:

```bash
BASE="http://170.187.156.131:8080"

for path in robots.txt .git/HEAD backup.zip backup.tar.gz \
            config.json config.py server.py app.py; do
  printf '\n===== %s =====\n' "$path"
  curl -sS --max-time 5 -i "$BASE/$path" | head -15
done
```

This attempt encountered the same `PATH` problem, so it did not successfully display the HTTP responses.

## 8. Identify the Exposed Source File

After the path checks, we reported the following result:

```text
server.py- 200, rest 404
```

**Finding:** The `/server.py` path returned HTTP `200 OK`, while the other tested paths returned `404 Not Found`.

This was the most important discovery in the challenge: the Python source file was publicly accessible through the web server.

## 9. Retrieve and Inspect `server.py`

We prepared the following command to download the exposed source file:

```bash
/usr/bin/curl -sS \
  http://170.187.156.131:8080/server.py \
  -o leaked_server.py
```

We could then inspect the source:

```bash
cat leaked_server.py
```

To locate potentially relevant code, we used the following search pattern:

```bash
grep -inE \
'cmd|secret|flag|token|admin|password|key|def |json' \
leaked_server.py
```

The user subsequently confirmed:

```text
server.py had it
```

The exposed source file was the key lead toward retrieving the challenge secret.

## 10. Final Exploitation and Flag

The contents of `server.py`, the exact successful WebSocket request, and the final flag were not included in the conversation transcript.

To complete this section accurately, document:

1. The relevant command handler from `server.py`.
2. The exact JSON request sent to the WebSocket service.
3. The server response containing the secret.
4. The recovered flag in the format `deadface{flag-text}`.

**Flag:** Not recorded in the available transcript.

## 11. Security Lessons

This challenge demonstrates several important security lessons:

- **Prevent source-code exposure:** Internal Python files should not be served from a public web root.
- **Protect secrets:** Credentials, tokens, and flags should not be embedded in publicly accessible files.
- **Enforce server-side authorization:** Hiding functionality in the frontend does not secure backend commands.
- **Minimize exposed services:** Only necessary ports and services should be reachable.
- **Use careful error handling:** Backend errors should not reveal unnecessary implementation details.

## Conclusion

We began with Nmap-based port enumeration and discovered the web application on port `8080`. Inspecting its HTML revealed a WebSocket backend on port `8765`.

Using Python, we connected to the WebSocket, confirmed that `list` returned movie data, and observed that `help` and `secret` were invalid commands.

Our path-enumeration attempts initially failed because `curl` and `head` were unavailable through the shell's `PATH`. We later identified `/server.py` as publicly accessible, returning HTTP `200 OK`, while the other tested paths returned `404`.

The exposed Python source was the key discovery. The exact final exploitation request and flag need to be added from the actual source code and successful server response.
