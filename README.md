# Logseq CLI HTTP Server

A simple HTTP server that wraps the [Logseq CLI](https://github.com/logseq/logseq/tree/master/deps/cli) to provide REST API access to your Logseq graphs.

## Why Use This?

The Logseq CLI is powerful but has limitations when used directly from browser extensions or other applications:

- **Browser extensions can't execute CLI commands** - Chrome/Firefox extensions run in a sandbox
- **Complex setup required** - Chrome Native Messaging requires manifests, extension IDs, and configuration
- **Debugging is difficult** - Native messaging errors are opaque and hard to troubleshoot

This HTTP server solves these problems by:

✅ **Simple HTTP interface** - Just make `fetch()` requests from any browser extension
✅ **Easy testing** - Test with `curl` or your browser
✅ **No configuration** - Works immediately after starting
✅ **Clear errors** - Standard HTTP status codes and JSON error messages
✅ **Browser agnostic** - Works with Chrome, Firefox, Safari, Edge

## When to Use the CLI (via this server) vs. Logseq's Built-in API

| Feature | Logseq CLI (this server) | Logseq Built-in HTTP API |
|---------|-------------------------|--------------------------|
| **Requires Logseq app running** | ❌ No | ✅ Yes |
| **Can list graphs** | ✅ Yes | ❌ No |
| **Can query graphs** | ✅ Yes (offline) | ✅ Yes (better) |
| **Can search** | ⚠️ Limited (title only) | ✅ Yes (full content) |
| **Can modify data** | ❌ No | ✅ Yes |
| **Real-time updates** | ❌ No | ✅ Yes |

**Use this server when:**
- You need to discover available graphs without opening Logseq
- You want offline access to graph data
- You're building a tool that works independently of the Logseq app

**Use Logseq's built-in API when:**
- The Logseq app is already running
- You need to modify graph data (create/update/delete)
- You need full-text search
- You need real-time updates

## Prerequisites

1. **Logseq CLI** installed:
   ```bash
   npm install -g @logseq/cli
   ```

2. **Python 3** (macOS/Linux ships with Python 3)

3. **Logseq Desktop graphs** - The CLI only works with Desktop app graphs, not browser-based Logseq

## Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/YOUR_USERNAME/logseq-cli-server.git
   cd logseq-cli-server
   ```

2. Verify Logseq CLI is installed:
   ```bash
   logseq --version
   ```

3. Start the server:
   ```bash
   python3 logseq_server.py
   ```

The server will start on `http://localhost:8765` by default.

## Usage

### Start the Server

```bash
python3 logseq_server.py

# Custom port
python3 logseq_server.py --port 8080

# Allow remote connections (be careful!)
python3 logseq_server.py --host 0.0.0.0
```

### API Endpoints

All endpoints return JSON with this structure:
```json
{
  "success": true|false,
  "stdout": "command output",
  "stderr": "error output if any",
  "returncode": 0
}
```

#### Health Check
```bash
GET /health
```

Example:
```bash
curl http://localhost:8765/health
```

#### List Graphs
```bash
GET /list
```

Lists all available Logseq graphs (both DB and File graphs).

Example:
```bash
curl http://localhost:8765/list
```

Response:
```json
{
  "success": true,
  "stdout": "DB Graphs:\n  research-notes\n  personal\n\nFile Graphs:\n  legacy-notes\n",
  "returncode": 0
}
```

#### Show Graph Info
```bash
GET /show?graph=GRAPH_NAME
```

Shows metadata about a specific graph.

Example:
```bash
curl "http://localhost:8765/show?graph=research-notes"
```

#### Search Graph
```bash
GET /search?q=QUERY&graph=GRAPH_NAME
```

Searches for content in a graph. Note: Offline search only searches `:block/title`, not full content.

Example:
```bash
curl "http://localhost:8765/search?q=anthropology&graph=research-notes"
```

#### Execute Query
```bash
POST /query
Content-Type: application/json

{
  "graph": "GRAPH_NAME",
  "query": "QUERY"
}
```

Executes a datalog query or entity query on a graph.

**Entity query example:**
```bash
curl -X POST http://localhost:8765/query \
  -H "Content-Type: application/json" \
  -d '{"graph":"research-notes","query":":logseq.class/Task"}'
```

**Datalog query example:**
```bash
curl -X POST http://localhost:8765/query \
  -H "Content-Type: application/json" \
  -d '{"graph":"research-notes","query":"[:find (pull ?b [*]) :where [?b :block/content]]"}'
```

## Usage from Browser Extensions

### manifest.json
```json
{
  "manifest_version": 3,
  "permissions": ["storage"],
  "host_permissions": ["http://localhost:8765/*"]
}
```

### JavaScript Example
```javascript
// List graphs
const response = await fetch('http://localhost:8765/list');
const data = await response.json();
console.log(data.stdout);

// Search
const searchResponse = await fetch(
  'http://localhost:8765/search?q=keyword&graph=my-graph'
);
const searchData = await searchResponse.json();

// Query
const queryResponse = await fetch('http://localhost:8765/query', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    graph: 'my-graph',
    query: ':logseq.class/Task'
  })
});
const queryData = await queryResponse.json();
```

## Important Limitations

### 1. CLI Can't Access Database While Logseq App is Running

**Problem:** If Logseq Desktop has a graph open, the CLI cannot access its database file.

**Solution:** Close the graph in Logseq Desktop before querying it via the CLI.

**Exception:** The `list` command always works since it doesn't need database access.

### 2. Sandbox Restrictions

**Problem:** If you run this server from within certain development environments (like Claude Code, VS Code with sandbox), Python's subprocess may not have access to the Logseq database files.

**Solution:** Run the server in a regular terminal, not through a sandboxed environment.

### 3. Search Limitations

**Offline search** (without API token) only searches `:block/title`, not full block content. For full-text search, you need:
- Logseq Desktop app running
- HTTP Server enabled in Logseq settings
- API token configured

### 4. Read-Only Access

The Logseq CLI (and this server) provides **read-only** access. You cannot create, update, or delete blocks/pages through the CLI.

For write operations, use [Logseq's built-in HTTP API](https://docs.logseq.com/#/page/logseq%20api).

## Security Considerations

### CORS Policy

By default, the server allows requests from **any origin** (`Access-Control-Allow-Origin: *`). This is fine for local development.

**For production**, edit `logseq_server.py` and restrict CORS:

```python
# In _set_headers method, change:
self.send_header('Access-Control-Allow-Origin', '*')

# To:
self.send_header('Access-Control-Allow-Origin', 'chrome-extension://YOUR_EXTENSION_ID')
```

### Network Binding

By default, the server only listens on `localhost` (not accessible from other machines). Keep it this way unless you specifically need remote access.

### Command Whitelist

The server only executes safe Logseq CLI commands:
- `list`
- `show`
- `search`
- `query`
- `export-edn`

No arbitrary shell execution is possible.

## Running in Background

### macOS/Linux - Using nohup
```bash
nohup python3 logseq_server.py > /dev/null 2>&1 &
```

Stop it:
```bash
pkill -f logseq_server.py
```

### macOS - Auto-start on Login (launchd)

Create `~/Library/LaunchAgents/com.logseq.cliserver.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.logseq.cliserver</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/python3</string>
        <string>/FULL/PATH/TO/logseq_server.py</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

Load it:
```bash
launchctl load ~/Library/LaunchAgents/com.logseq.cliserver.plist
```

## Troubleshooting

### "logseq command not found"

Install the Logseq CLI:
```bash
npm install -g @logseq/cli
```

Verify installation:
```bash
which logseq
logseq --version
```

### "unable to open database file"

**Cause:** Logseq Desktop app has the graph open, locking the database.

**Solution:** Close the graph in Logseq Desktop before querying.

**Note:** `logseq list` always works since it doesn't access databases.

### Port already in use

Use a different port:
```bash
python3 logseq_server.py --port 9000
```

Update your client code to use the new port.

### No graphs listed

**Cause:** You're using browser-based Logseq (web version).

**Solution:** The CLI only works with Logseq Desktop graphs. Open your graphs in the Desktop app.

## Logging

All requests are logged to:
- **File:** `./logseq-http-server.log` (in the same directory as the script)
- **Console:** stdout

View logs in real-time:
```bash
tail -f logseq-http-server.log
```

## Comparison with Alternatives

### vs. Logseq Built-in HTTP API

**Logseq CLI Server (this project):**
- ✅ Works without Logseq app running
- ✅ Can list available graphs
- ❌ Read-only access
- ❌ Limited search (title only)

**Logseq HTTP API:**
- ❌ Requires Logseq app running
- ❌ Cannot list graphs
- ✅ Full read/write access
- ✅ Full-text search
- ✅ Real-time updates

### vs. Chrome Native Messaging

**HTTP Server (this project):**
- ✅ Simple setup (just run the script)
- ✅ Easy debugging (curl, browser)
- ✅ Works with any browser
- ❌ Manual start/stop

**Chrome Native Messaging:**
- ❌ Complex setup (manifests, extension IDs)
- ❌ Hard to debug
- ❌ Chrome only
- ✅ Auto-managed by Chrome

## Contributing

Contributions welcome! This is a simple wrapper around the Logseq CLI - feel free to submit issues or pull requests.

## License

MIT

## Acknowledgments

- [Logseq CLI](https://github.com/logseq/logseq/tree/master/deps/cli) - The official Logseq command-line interface
- [Logseq](https://logseq.com/) - A privacy-first, open-source knowledge base
