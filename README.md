# Runesynth MCP (WebAudio via Electron)

Minimal, working MCP server that lets an LLM patch a WebAudio graph through Electron.

## Components
- **MCP server (Node/TypeScript)**: exposes tools over stdio (`@modelcontextprotocol/sdk`).
- **Electron app**: hosts a browser `AudioContext` and executes graph operations.
- **WebSocket bridge**: `ws://127.0.0.1:17865` connects the MCP process to the Electron process.

## Project Layout
```
runesynth-mcp/
  package.json
  tsconfig.json
  src/
    mcp/server.ts
    bridge/electron-main.ts
    renderer/index.html
    renderer/webaudio.ts
```

## Prereqs
- Node 18+
- npm
- macOS/Windows/Linux with audio device

## Install
```bash
npm i
npm run build
```

## Develop (hot-compile + both processes)
In one terminal:
```bash
npm run dev
```
This runs `tsc -w`, starts Electron on `dist/bridge/electron-main.js`, and the MCP server on stdio.

Alternatively, run in two terminals:
```bash
npm run build
npm run dev:electron   # Terminal A (Electron + WebSocket)
npm run dev:mcp        # Terminal B (MCP stdio server)
```

## Use with an MCP client
- **Claude Desktop**: add to `mcpServers` config as a command pointing to `node` with args `dist/mcp/server.js` (or install globally and use the `runesynth-mcp` bin after a build).
- Any MCP-compliant client can talk to it over stdio.

## Tools
- `reset {}`
- `create_oscillator {"id":"vco1","type":"sine|square|sawtooth|triangle","frequency":220}`
- `create_gain {"id":"vca","gain":0.5}`
- `create_filter {"id":"lpf","type":"lowpass|highpass|bandpass|notch","frequency":1000,"Q":0.7}`
- `connect {"sourceId":"vco1","destId":"lpf"}`
- `connect_to_output {"nodeId":"vca"}`
- `set_param {"nodeId":"lpf","param":"frequency","value":800}`
- `start {"nodeId":"vco1","when":0}`
- `stop {"nodeId":"vco1","when":0}`
- `render_graph {}`

## Smoke Test (LLM or CLI issuing tool calls)
1. `reset {}`
2. Create nodes
   ```
   create_oscillator {"id":"vco1","type":"sawtooth","frequency":110}
   create_filter     {"id":"lpf","type":"lowpass","frequency":800,"Q":0.8}
   create_gain       {"id":"vca","gain":0.0}
   ```
3. Wire
   ```
   connect {"sourceId":"vco1","destId":"lpf"}
   connect {"sourceId":"lpf","destId":"vca"}
   connect_to_output {"nodeId":"vca"}
   ```
4. Play
   ```
   set_param {"nodeId":"vca","param":"gain","value":0.2}
   start {"nodeId":"vco1"}
   ```
5. Sweep filter
   ```
   set_param {"nodeId":"lpf","param":"frequency","value":200}
   set_param {"nodeId":"lpf","param":"frequency","value":2200}
   ```
6. Stop
   ```
   stop {"nodeId":"vco1"}
   ```

## Design Notes
- **Timing**: For musical scheduling, add a `clock` tool and apply offsets with `ctx.currentTime` in the renderer.
- **Persistence**: Add `save_graph` / `load_graph` tools (JSON) to serialize patches.
- **Validation**: Enforce patching constraints in the MCP layer before sending to Electron.
- **Headless alt**: If you must avoid Electron, you can experiment with `standardized-audio-context` in Node (no hardware output parity guaranteed).

## License
MIT
