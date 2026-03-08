# Migrating Excalidraw MCP from WebSockets to Server-Sent Events (SSE)

## Overview & Architecture
The Excalidraw MCP project is distributed as two separate Docker components:
1. **MCP Excalidraw Server** (`Dockerfile`): Connects to AI agents via stdio and communicates with the Canvas Server via HTTP REST endpoints.
2. **MCP Excalidraw Canvas Server** (`Dockerfile.canvas`): Provides the visual web interface (React) and the REST API backend (Node.js).

Currently, real-time synchronization between the Node.js Canvas Server and the React Frontend utilizes WebSockets (`ws`). In strict enterprise environments, corporate proxies often block the `Upgrade: websocket` HTTP header, returning a `426 Upgrade Required` error. 

To resolve this while maintaining a real-time drawing experience, this design outlines replacing WebSockets with Server-Sent Events (SSE). SSE operates over standard HTTP (`text/event-stream`), making it proxy-friendly (especially when `NO_PROXY` is configured for localhost/internal traffic).

---

## 1. Canvas Server Component Changes (`Dockerfile.canvas`)

This component houses both the backend Node server and the frontend React application. **All code changes to support this migration will occur within this component.**

### Part A: Node.js Backend (`src/server.ts`)
The server must drop the `ws` dependency and expose an SSE endpoint to push events to the frontend.

**Phase 1: Remove WebSocket Dependencies**
- Remove `import { WebSocketServer } from 'ws';`
- Remove `import WebSocket from 'ws';`
- Remove `const wss = new WebSocketServer({ server });` and all `wss.on('connection')` logic.

**Phase 2: Implement SSE Client Tracking**
- Replace `const clients = new Set<WebSocket>();` with `const sseClients = new Set<express.Response>();`.

**Phase 3: Create the SSE Endpoint (`GET /api/stream`)**
Create a new Express route to handle SSE connections.

```typescript
app.get('/api/stream', (req: Request, res: Response) => {
  // 1. Set required SSE headers
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive'
  });

  // 2. Add client to tracking set
  sseClients.add(res);
  logger.info('New SSE connection established');

  // 3. Send initial state immediately (current elements and sync status)
  const filesObj: Record<string, ExcalidrawFile> = {};
  files.forEach((f, id) => { filesObj[id] = f; });
  
  const initialMessage = {
    type: 'initial_elements',
    elements: Array.from(elements.values()),
    ...(files.size > 0 ? { files: filesObj } : {})
  };
  
  const syncMessage = {
    type: 'sync_status',
    elementCount: elements.size,
    timestamp: new Date().toISOString()
  };

  // SSE format requires "data: <json>\n\n"
  res.write(\`data: \${JSON.stringify(initialMessage)}\\n\\n\`);
  res.write(\`data: \${JSON.stringify(syncMessage)}\\n\\n\`);

  // 4. Handle client disconnect
  req.on('close', () => {
    sseClients.delete(res);
    logger.info('SSE connection closed');
  });
});
```

**Phase 4: Update the Broadcast Function**
Update the existing `broadcast()` function to write to the SSE response streams instead of sending WebSocket frames.

```typescript
// Broadcast to all connected SSE clients
function broadcast(message: WebSocketMessage): void {
  const data = \`data: \${JSON.stringify(message)}\\n\\n\`;
  sseClients.forEach(client => {
    try {
      client.write(data);
    } catch (err) {
      logger.warn('Failed to send to SSE client, removing');
      sseClients.delete(client);
    }
  });
}
```

### Part B: React Frontend (`frontend/src/App.tsx`)
The React app must drop the `WebSocket` API and use the browser's native `EventSource` API to consume the SSE stream from the Node backend.

**Phase 1: Remove WebSocket Logic**
- Remove `websocketRef = useRef<WebSocket | null>(null)`.
- Replace the `connectWebSocket()` function hook with `connectSSE()`.

**Phase 2: Implement EventSource**
Update `connectSSE()` to use `EventSource` pointing to `/api/stream`.

```typescript
const eventSourceRef = useRef<EventSource | null>(null);

const connectSSE = (): void => {
  if (eventSourceRef.current && eventSourceRef.current.readyState === EventSource.OPEN) {
    return;
  }

  // SSE uses standard HTTP/HTTPS URLs mapping to the Node backend
  const sseUrl = '/api/stream';
  eventSourceRef.current = new EventSource(sseUrl);

  eventSourceRef.current.onopen = () => {
    setIsConnected(true);
    if (excalidrawAPI) {
      setTimeout(loadExistingElements, 100);
    }
  };

  eventSourceRef.current.onmessage = (event: MessageEvent) => {
    try {
      // EventSource provides the exact JSON payload broadcast from the server
      const data: WebSocketMessage = JSON.parse(event.data);
      handleWebSocketMessage(data); // Re-use existing business logic
    } catch (error) {
      console.error('Error parsing SSE message:', error, event.data);
    }
  };

  eventSourceRef.current.onerror = (error: Event) => {
    console.error('SSE error:', error);
    setIsConnected(false);
    if (eventSourceRef.current?.readyState === EventSource.CLOSED) {
      setTimeout(connectSSE, 3000);
    }
  };
}
```

**Phase 3: Update Lifecycle Hooks**
Ensure the `EventSource` is closed when the component unmounts.

```typescript
useEffect(() => {
  connectSSE();
  return () => {
    if (eventSourceRef.current) {
      eventSourceRef.current.close();
    }
  }
}, []);
```

---

## 2. MCP Client Component Changes (`Dockerfile`)

This component houses the core MCP Server (`src/index.ts`) which interfaces with the AI agent and the external Canvas Server component over HTTP REST.

### Required Changes: **None** (0 lines of code)

Because the MCP Client operates strictly via HTTP REST endpoints (e.g., `fetch(EXPRESS_SERVER_URL + '/api/elements')`), it is entirely agnostic to how the Canvas Server broadcasts updates to the React Frontend. 

When the MCP Client creates an element via a `POST` request, the Canvas Server processes it and automatically triggers the new SSE `broadcast()` function. Thus, the MCP Client component continues to function identically without needing any code modifications or awareness of the SSE migration.

---

## Summary of Impact
*   **Proxy Compatibility:** High. Solves the `426 Upgrade Required` issue by removing WebSockets.
*   **Canvas Server (`Dockerfile.canvas`)**: Requires backend and frontend code modifications to replace `ws` with `text/event-stream`.
*   **MCP Client (`Dockerfile`)**: Requires absolutely zero modifications.
*   **Dependency Size:** Reduced. We can uninstall the `ws` and `@types/ws` packages from the repository.
