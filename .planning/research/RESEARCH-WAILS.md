# Wails v2 Research: PairAdmin Desktop Application

**Project:** PairAdmin — Go desktop app with Wails v2 + React + TypeScript
**Researched:** 2026-03-25
**Wails version at research time:** v2.10.1 (latest stable as of Feb 2025); v2.11.0 expected 2025-11
**Overall confidence:** MEDIUM-HIGH — core architecture is HIGH confidence from official sources; packaging and streaming patterns are MEDIUM confidence based on verified community findings

---

## 1. Wails v2 Architecture: How Go and JavaScript Bindings Work

### The Binding Mechanism

Wails uses CGO + WebKit2GTK (Linux), WebView2 (Windows), or WKWebView (macOS) as the webview host. The Go backend communicates with the frontend through two channels:

**Synchronous bindings (RPC-style):** Public methods on Go structs bound at startup are exposed as async JavaScript functions. When the frontend calls a bound method, Wails serializes the arguments to JSON, invokes the Go function via an internal IPC bridge (not real WebSockets—a custom protocol through the webview), and returns the result as a Promise that resolves with the JSON-deserialized response.

**Asynchronous events:** The runtime's event system (`runtime.EventsEmit` / `runtime.EventsOn`) allows either side to push named messages with arbitrary JSON-serializable payloads.

### Setting Up Bindings in main.go

```go
package main

import (
    "context"
    "github.com/wailsapp/wails/v2"
    "github.com/wailsapp/wails/v2/pkg/options"
    "github.com/wailsapp/wails/v2/pkg/options/assetserver"
)

func main() {
    app := NewApp()
    chat := NewChatService()
    terminal := NewTerminalService()

    err := wails.Run(&options.App{
        Title:  "PairAdmin",
        Width:  1400,
        Height: 900,
        AssetServer: &assetserver.Options{
            Assets: assets, // embed.FS
        },
        OnStartup: func(ctx context.Context) {
            app.SetContext(ctx)
            chat.SetContext(ctx)
            terminal.SetContext(ctx)
        },
        OnShutdown:    app.shutdown,
        OnBeforeClose: app.beforeClose,
        Bind: []interface{}{
            app,
            chat,
            terminal,
        },
    })
}
```

The `ctx context.Context` saved in `OnStartup` is the **required token for all runtime calls** (`runtime.EventsEmit`, `runtime.MessageDialog`, etc.). This context must be stored on each struct and is valid for the lifetime of the app.

### Multiple Struct Bindings Pattern

For PairAdmin's three-panel architecture, bind separate service structs for clean separation:

```go
type ChatService struct { ctx context.Context }
type TerminalService struct { ctx context.Context }
type CommandService struct { ctx context.Context }

func (s *ChatService) SetContext(ctx context.Context) { s.ctx = ctx }
```

Each struct's public methods become callable from the frontend under the namespace `window.go/main/ChatService.MethodName`.

### TypeScript Binding Generation

Wails auto-generates TypeScript wrappers in `frontend/wailsjs/go/` when running `wails dev`. The structure is:

```
frontend/wailsjs/
  go/
    main/
      App.js          // JS wrapper
      App.d.ts        // TypeScript declarations
      ChatService.js
      ChatService.d.ts
      models.ts       // All Go struct types used as params/returns
  runtime/
    runtime.js        // Wails runtime (events, dialogs, etc.)
    runtime.d.ts
```

**Critical rules for TypeScript generation:**
- Only methods on **bound struct instances** are exported — not package-level functions
- Methods must start with an **uppercase letter** (exported in Go)
- Return types must be JSON-serializable (no channels, no funcs, no complex numbers)
- Struct fields **require `json` tags** (or must be exported — since late 2024, exported fields without json tags are included using the field name as-is per PR #3678)
- Regeneration happens automatically in `wails dev`; for production builds run `wails generate module` or rely on the build pipeline

**Known limitation (HIGH confidence):** TypeScript generation in v2 uses reflection during dev mode — variable names in generated bindings can differ from Go source in edge cases. The `models.ts` output is generally reliable for plain structs.

### Event System API

**Go side (emitting events to frontend):**
```go
import "github.com/wailsapp/wails/v2/pkg/runtime"

// From within any method that has access to ctx
runtime.EventsEmit(ctx, "chat:token", map[string]interface{}{
    "messageId": id,
    "token":     token,
    "done":      false,
})

// One-off events
runtime.EventsEmit(ctx, "terminal:output", line)
```

**JavaScript side (receiving events from Go):**
```typescript
import { EventsOn, EventsOff } from "../wailsjs/runtime/runtime";

// In a React useEffect:
useEffect(() => {
    const cleanup = EventsOn("chat:token", (data: ChatTokenPayload) => {
        // data is the deserialized JSON payload
        appendToken(data.messageId, data.token);
    });
    return () => EventsOff("chat:token");
}, []);
```

**Go side listening to frontend events:**
```go
runtime.EventsOn(ctx, "user:command", func(data ...interface{}) {
    // handle
})
```

**Frontend emitting to Go:**
```typescript
import { EventsEmit } from "../wailsjs/runtime/runtime";
EventsEmit("user:command", { cmd: "deploy" });
```

---

## 2. React + TypeScript Template Setup

### Project Initialization

```bash
# Install Wails CLI
go install github.com/wailsapp/wails/v2/cmd/wails@latest

# Create project with React-TS template (uses Vite under the hood)
wails init -n pairadmin -t react-ts

# The official react-ts template uses:
# - Vite (bundler)
# - React 18
# - TypeScript
# - No CSS framework (add your own)
```

### Recommended Project Structure for PairAdmin

```
pairadmin/
  main.go                     # wails.Run entry point
  app.go                      # App struct (lifecycle hooks)
  services/
    chat.go                   # ChatService (LLM streaming, history)
    terminal.go               # TerminalService (exec, output streaming)
    commands.go               # CommandService (sidebar commands)
  wails.json                  # Wails project config
  frontend/
    src/
      App.tsx                 # Root layout: three-column
      components/
        Terminal/
          TerminalPane.tsx    # Tab container
          TerminalTab.tsx     # Single terminal tab with xterm.js
        Chat/
          ChatPane.tsx        # Chat area
          ChatBubble.tsx      # Individual message bubble
          CodeBlock.tsx       # Syntax-highlighted code (uses react-shiki)
        Sidebar/
          CommandSidebar.tsx  # Command palette / sidebar
      stores/
        chatStore.ts          # Zustand store for chat state
        terminalStore.ts      # Zustand store for terminal tabs
        commandStore.ts       # Zustand store for sidebar
      hooks/
        useWailsEvents.ts     # Typed wrapper around Wails EventsOn
        useChatStream.ts      # Streaming chat hook
      wailsjs/                # Auto-generated — do not edit
        go/main/
        runtime/
    package.json
    vite.config.ts
    tsconfig.json
```

### wails.json Configuration

```json
{
  "$schema": "https://wails.io/schemas/config.v2.json",
  "name": "pairadmin",
  "outputfilename": "pairadmin",
  "frontend:install": "npm install",
  "frontend:build": "npm run build",
  "frontend:dev:watcher": "npm run dev",
  "frontend:dev:serverUrl": "http://localhost:5173",
  "author": {
    "name": "Your Name"
  }
}
```

The `frontend:dev:serverUrl` points to the Vite dev server. When `wails dev` is running, the webview loads from Vite (with HMR) instead of the embedded assets. Go source changes trigger a full rebuild; frontend changes hot-reload instantly.

### Calling Go Functions from React

After `wails dev` runs once, the `frontend/wailsjs/` directory is populated. Import and call:

```typescript
// Direct RPC call — returns a Promise
import { SendMessage } from "../wailsjs/go/main/ChatService";

async function handleSend(text: string) {
    try {
        const result = await SendMessage(text);
        // result is typed per the Go return value
    } catch (err) {
        // Go errors become rejected Promises
        console.error(err);
    }
}
```

**Important:** Bound Go methods that return `(T, error)` — the error becomes a rejected Promise. Methods returning only `error` resolve with null. Methods returning only `T` resolve with the value.

---

## 3. Real-Time / Streaming Patterns for Chat

### The Core Challenge

Wails' event system is **asynchronous and unordered** under high emit rates. Sending one event per token (50-200 events/second for fast LLMs) will produce out-of-order delivery on the frontend due to async webview script injection. This is a **confirmed and documented issue** (GitHub Issue #2759, maintainer-acknowledged).

### Recommended Pattern: Sequence-Numbered Batched Streaming

The best approach for PairAdmin combines a streaming Go goroutine with batching and sequence numbers:

**Go side — streaming handler:**
```go
type TokenChunk struct {
    MessageID string `json:"messageId"`
    Seq       int    `json:"seq"`
    Tokens    string `json:"tokens"` // batch multiple tokens
    Done      bool   `json:"done"`
}

func (c *ChatService) StreamMessage(prompt string) error {
    messageID := generateID()
    stream := c.llmClient.StreamCompletion(prompt)

    go func() {
        var buf strings.Builder
        seq := 0
        ticker := time.NewTicker(50 * time.Millisecond) // 50ms batch window
        defer ticker.Stop()

        flush := func() {
            if buf.Len() == 0 {
                return
            }
            runtime.EventsEmit(c.ctx, "chat:token", TokenChunk{
                MessageID: messageID,
                Seq:       seq,
                Tokens:    buf.String(),
                Done:      false,
            })
            buf.Reset()
            seq++
        }

        for {
            select {
            case token, ok := <-stream:
                if !ok {
                    flush()
                    runtime.EventsEmit(c.ctx, "chat:token", TokenChunk{
                        MessageID: messageID,
                        Seq:       seq,
                        Done:      true,
                    })
                    return
                }
                buf.WriteString(token)
            case <-ticker.C:
                flush()
            }
        }
    }()

    return nil
}
```

**Why this works:**
- 50ms batching aligns with the Go polling loop cadence mentioned in PairAdmin's requirements
- Sequence numbers allow the frontend to detect and handle out-of-order delivery
- The channel select + ticker pattern avoids blocking the LLM stream reader

**React side — useChatStream hook:**
```typescript
interface TokenChunk {
    messageId: string;
    seq: number;
    tokens: string;
    done: boolean;
}

export function useChatStream() {
    const appendChunk = useChatStore(s => s.appendChunk);
    const pendingChunks = useRef<Map<string, { seq: number; buffer: TokenChunk[] }>>(new Map());

    useEffect(() => {
        const cleanup = EventsOn("chat:token", (chunk: TokenChunk) => {
            const state = pendingChunks.current.get(chunk.messageId) ?? { seq: 0, buffer: [] };

            if (chunk.done) {
                appendChunk(chunk.messageId, "", true);
                pendingChunks.current.delete(chunk.messageId);
                return;
            }

            // Sequence reordering: buffer out-of-order chunks
            if (chunk.seq === state.seq) {
                appendChunk(chunk.messageId, chunk.tokens, false);
                state.seq++;
                // Drain any buffered future chunks
                let next = state.buffer.findIndex(c => c.seq === state.seq);
                while (next !== -1) {
                    appendChunk(chunk.messageId, state.buffer[next].tokens, false);
                    state.buffer.splice(next, 1);
                    state.seq++;
                    next = state.buffer.findIndex(c => c.seq === state.seq);
                }
            } else {
                state.buffer.push(chunk);
            }
            pendingChunks.current.set(chunk.messageId, state);
        });

        return cleanup;
    }, [appendChunk]);
}
```

### Alternative: Polling via Bound Method

An alternative to events is polling: Go buffers tokens in a string accumulator, and React calls a bound method like `GetPendingTokens(messageId)` every 100ms. This is simpler (no ordering issues, no EventsEmit) but introduces 100ms latency and requires a polling interval. **Recommended for status updates; not for real-time chat rendering** where the event approach with batching is better UX.

### Approach Comparison

| Approach | Ordering | Latency | Complexity | Recommendation |
|----------|----------|---------|------------|----------------|
| EventsEmit per-token | UNRELIABLE | ~10ms | Low | Avoid for high rate |
| EventsEmit batched (50ms) + seqnum | Reliable | ~50ms | Medium | **Use for chat streaming** |
| Bound method polling (100ms) | Reliable | ~100ms | Low | Use for status/progress |
| Long-polling (bound method blocks) | N/A | ~0ms | High | Avoid — blocks goroutine |

---

## 4. State Management Recommendation

### Recommendation: Zustand (with Jotai for fine-grained UI atoms)

**For PairAdmin's complexity, use Zustand as the primary store.** The app has three distinct data domains (terminal tabs, chat messages, command sidebar) that are largely independent. Zustand's store-per-domain pattern maps cleanly to this:

```typescript
// stores/chatStore.ts
import { create } from "zustand";
import { immer } from "zustand/middleware/immer";

interface ChatMessage {
    id: string;
    role: "user" | "assistant";
    content: string;
    isStreaming: boolean;
}

interface ChatStore {
    messages: ChatMessage[];
    appendChunk: (messageId: string, tokens: string, done: boolean) => void;
    addUserMessage: (text: string) => string; // returns messageId
}

export const useChatStore = create<ChatStore>()(
    immer((set) => ({
        messages: [],
        addUserMessage: (text) => {
            const id = crypto.randomUUID();
            set(state => {
                state.messages.push({ id, role: "user", content: text, isStreaming: false });
            });
            return id;
        },
        appendChunk: (messageId, tokens, done) => {
            set(state => {
                const msg = state.messages.find(m => m.id === messageId);
                if (!msg) {
                    // Create assistant message on first token
                    state.messages.push({
                        id: messageId, role: "assistant", content: tokens, isStreaming: !done
                    });
                } else {
                    msg.content += tokens;
                    msg.isStreaming = !done;
                }
            });
        },
    }))
);
```

**Why Zustand over alternatives:**
- **vs Redux:** Redux Toolkit is overkill for a single-window desktop app. 3-4KB vs ~50KB bundle. No need for middleware, devtools ecosystem is less critical.
- **vs Jotai:** Jotai's atomic model excels when many small independent UI atoms need to update in isolation. For chat messages (an ordered list that appends), a single Zustand store with Immer is simpler and avoids atom-derivation complexity.
- **vs Context:** React Context causes full subtree re-renders on any state change — unacceptable for streaming token updates hitting the store 20x/second.

**Use Jotai alongside Zustand for UI micro-state:**
```typescript
// Fine-grained UI state that doesn't belong in the business store
const activeTabAtom = atom<string>("tab-1");
const sidebarOpenAtom = atom(true);
const commandSearchAtom = atom("");
```

**Install:**
```bash
npm install zustand
npm install zustand/middleware  # immer, devtools, persist
npm install jotai               # for fine-grained UI atoms
```

**Zustand version:** 5.x (major release in late 2024, breaking changes from v4 — use `useShallow` instead of shallow import, `create` API unchanged)

---

## 5. Syntax Highlighting for Code Blocks in Chat

### Recommendation: react-shiki for Chat Code Blocks

For a programmer-focused tool like PairAdmin, **code highlighting quality matters.** Use `react-shiki` (Shiki-powered) for chat code blocks because:

1. Shiki uses TextMate grammars — the same engine as VS Code — for accurate highlighting of TypeScript generics, JSX, complex Go types
2. `react-shiki` provides built-in streaming support with a `delay` prop for throttling re-highlights during token streaming
3. The WASM bundle cost (~280KB gzipped for full, ~700KB for web bundle) is acceptable in a desktop app where you control the environment — no CDN latency concerns

**Installation:**
```bash
npm install react-shiki
```

**Usage in ChatBubble:**
```typescript
import ShikiHighlighter from "react-shiki";

interface CodeBlockProps {
    language: string;
    code: string;
    isStreaming: boolean;
}

export function CodeBlock({ language, code, isStreaming }: CodeBlockProps) {
    return (
        <ShikiHighlighter
            language={language || "text"}
            theme="github-dark"
            delay={isStreaming ? 100 : 0}  // throttle re-highlight during stream
        >
            {code}
        </ShikiHighlighter>
    );
}
```

**Bundle optimization — use web bundle for smaller footprint:**
```typescript
// Instead of default full bundle, import web-focused languages only
import ShikiHighlighter from "react-shiki/web";
```

Or for maximum control (only the languages PairAdmin uses):
```typescript
import { createHighlighterCore } from "react-shiki/core";
// Manually import only go, typescript, tsx, bash, json, yaml, etc.
```

### Alternative: react-syntax-highlighter (Prism)

If the WASM startup cost is a concern (e.g., app must be responsive immediately on launch), use `react-syntax-highlighter` with the Prism backend:

```bash
npm install react-syntax-highlighter @types/react-syntax-highlighter
```

```typescript
import { Prism as SyntaxHighlighter } from "react-syntax-highlighter";
import { vscDarkPlus } from "react-syntax-highlighter/dist/esm/styles/prism";

<SyntaxHighlighter language={language} style={vscDarkPlus}>
    {code}
</SyntaxHighlighter>
```

Prism is 11.7KB gzipped, starts in 0.5ms, but has weaker TypeScript/Go accuracy. Acceptable as a fallback.

### Parsing Code Blocks from Markdown

For the chat area, markdown responses contain fenced code blocks. Use `react-markdown` with a custom `components` prop to route code blocks to ShikiHighlighter:

```bash
npm install react-markdown
```

```typescript
import ReactMarkdown from "react-markdown";

<ReactMarkdown
    components={{
        code({ node, inline, className, children, ...props }) {
            const match = /language-(\w+)/.exec(className || "");
            const language = match ? match[1] : "text";
            return !inline ? (
                <CodeBlock language={language} code={String(children).replace(/\n$/, "")} />
            ) : (
                <code className={className} {...props}>{children}</code>
            );
        },
    }}
>
    {message.content}
</ReactMarkdown>
```

---

## 6. Packaging and Distribution on Linux

### Wails v2 Build Output

`wails build` produces a single binary at `build/bin/pairadmin`. The binary embeds the frontend assets via Go's `embed.FS`. It **does not** bundle WebKit — it depends on the system's `libwebkit2gtk-4.0` or `libwebkit2gtk-4.1`.

### AppImage (Recommended for Distribution)

Wails v2 includes an `AppImage` build directory scaffolded at `build/linux/`. The generated `build.sh` uses `linuxdeploy` and `linuxdeploy-plugin-gtk`:

```
build/
  linux/
    appimage/
      build.sh          # linuxdeploy-based AppImage builder
      build/
        usr/
          share/
            applications/
              pairadmin.desktop
            icons/
              hicolor/512x512/apps/
                pairadmin.png
```

**Build AppImage:**
```bash
# Prerequisites on build machine
wget https://github.com/linuxdeploy/linuxdeploy/releases/download/continuous/linuxdeploy-x86_64.AppImage
wget https://raw.githubusercontent.com/linuxdeploy/linuxdeploy-plugin-gtk/master/linuxdeploy-plugin-gtk.sh
chmod +x linuxdeploy-x86_64.AppImage linuxdeploy-plugin-gtk.sh

# Build the Wails binary first
wails build

# Run the AppImage builder
cd build/linux/appimage
./build.sh
```

**Known AppImage issue (MEDIUM confidence):** The AppImage will NOT bundle `WebKitNetworkProcess` and `WebKitGPUProcess` in some configurations (GitHub Issue #4313). The app will work if the target system has matching webkit2gtk installed, but a fully self-contained AppImage is difficult. Workaround: document webkit2gtk as a system dependency, or target a base system (Ubuntu 20.04 LTS) and build on it for maximum compatibility.

### .deb Package (nFPM)

Wails v2 does not have native `wails build --package` for deb/rpm — that's v3 alpha only. Use nFPM directly:

```bash
# Install nFPM
go install github.com/goreleaser/nfpm/v2/cmd/nfpm@latest
```

`nfpm.yaml`:
```yaml
name: pairadmin
arch: amd64
platform: linux
version: "0.1.0"
section: utils
maintainer: Your Name <you@example.com>
description: PairAdmin desktop application
homepage: https://github.com/yourname/pairadmin

depends:
  - libwebkit2gtk-4.1-0
  - libgtk-3-0

deb:
  compression: xz

contents:
  - src: build/bin/pairadmin
    dst: /usr/bin/pairadmin
  - src: build/linux/appimage/build/usr/share/applications/pairadmin.desktop
    dst: /usr/share/applications/pairadmin.desktop
  - src: build/linux/appimage/build/usr/share/icons/hicolor/512x512/apps/pairadmin.png
    dst: /usr/share/icons/hicolor/512x512/apps/pairadmin.png
```

```bash
# Build .deb
nfpm package --packager deb --target dist/

# Build .rpm
nfpm package --packager rpm --target dist/
```

**WebKit dependency declaration:** Use `libwebkit2gtk-4.1-0` (runtime, not `-dev`) for Ubuntu 22.04+ and Fedora 38+. For Ubuntu 20.04/22.04 compatibility, declare `libwebkit2gtk-4.0-0`. The nfpm.yaml template from Wails v3 PR #4481 corrected this — use runtime packages, not dev packages.

### Build Tag for Ubuntu 24.04

```bash
# If building on or targeting Ubuntu 24.04+
wails build -tags webkit2_41
```

This switches the build to use `libwebkit2gtk-4.1` instead of `libwebkit2gtk-4.0`.

### Summary of Linux Distribution Strategy

| Format | Tool | System Dep Required | Self-Contained |
|--------|------|---------------------|----------------|
| AppImage | linuxdeploy + plugin-gtk | gtk3, glib only | Mostly (WebKit excluded) |
| .deb | nFPM | libwebkit2gtk-4.1-0 | No |
| .rpm | nFPM | webkit2gtk4.1 | No |
| Binary only | wails build | libwebkit2gtk + libgtk3 | No |

**Recommendation:** Distribute as both AppImage (for universal Linux) and .deb (for apt-based distros). Document webkit as a runtime dependency. Build on Ubuntu 22.04 LTS for broadest AppImage compatibility.

---

## 7. Known Wails v2 Gotchas

### Critical: Ubuntu 24.04 webkit2gtk Version Split

- Ubuntu 24.04 dropped `libwebkit2gtk-4.0-dev`; only `libwebkit2gtk-4.1-dev` is available
- Fix: Install `libwebkit2gtk-4.1-dev` and build with `-tags webkit2_41`
- Status: Fixed in Wails master (via PR #3461), but may not be in the latest tagged release
- Symlink workaround if on v2.8.x: `ln -s webkit2gtk-4.1.pc webkit2gtk-4.0.pc`

### Critical: Signal Handler Interference (Linux)

WebKit2GTK installs its own signal handlers that block Go's `SIGSEGV` → panic conversion. A nil pointer dereference in a goroutine will crash the process rather than being recoverable:

```go
// If you need recoverable panics, call before the risky code:
import goruntime "runtime"
goruntime.ResetSignalHandlers() // Re-registers Go's signal handlers
```

This is a WebKit architectural constraint — not fixable in application code. Design defensively: never rely on `recover()` for nil dereferences in Go code that runs alongside WebKit.

### Moderate: Out-of-Order Event Delivery

High-frequency `runtime.EventsEmit` calls (>10/second) may arrive out of order on the frontend. Confirmed data race issue (#2448) in the events system. Mitigations:
- Use sequence numbers in event payloads (see Section 3)
- Batch events on a timer (50ms)
- For critical ordering, use bound method polling instead of events

### Moderate: EventsOn Goroutine Leak

`EventsOn` in Go registers a listener but the cleanup function returned by `EventsOn` in JavaScript must be called to deregister. In React, always call the cleanup in `useEffect` return:

```typescript
useEffect(() => {
    const off = EventsOn("my:event", handler);
    return off; // This IS the cleanup function
}, []);
```

On the Go side, `runtime.EventsOff(ctx, "my:event")` removes all listeners for that event name — use carefully if multiple components listen to the same event.

### Moderate: CSP and Inline Scripts

Wails injects two inline scripts into `index.html` at startup (`/wails/ipc.js` and `/wails/runtime.js`). If you add a Content Security Policy with `script-src 'self'`, these injected scripts will be blocked.

Options:
1. Do not add a `<meta http-equiv="Content-Security-Policy">` tag in your `index.html` — Wails has no built-in CSP header injection for the asset server
2. If you need CSP, use `'unsafe-inline'` for scripts or the Wails-specific nonce (not officially documented for v2)
3. For a local desktop app, CSP is lower priority than a web app — the threat model is different

### Minor: Webview Flickering on Resize (Linux/GTK)

GTK's webview flickers when the window is resized. No application-level workaround exists — this is a WebKit2GTK rendering behavior. Avoid layouts that resize frequently or use fixed minimum window dimensions.

### Minor: Window Background Color Race

On Linux, the window may briefly show white before the webview renders (Issue #2852). Mitigate by setting `BackgroundColour` in options to match your app's background color:

```go
options.App{
    BackgroundColour: &options.RGBA{R: 18, G: 18, B: 18, A: 255}, // match CSS background
}
```

### Minor: Audio/Video GStreamer Dependencies

If PairAdmin ever needs to play audio or video in the webview, install GStreamer plugins:
```bash
sudo apt install gstreamer1.0-plugins-good gstreamer1.0-plugins-bad
```

### Minor: BrowserOpenURL Silent Failures

`runtime.BrowserOpenURL` uses `github.com/pkg/browser` and can fail silently if no default browser is configured on a minimal Linux system (Issue #3261). Add error handling or implement a fallback using `xdg-open` directly via `os/exec`.

---

## 8. Performance: Handling 500ms Go→React Updates Without Re-Render Churn

### The Problem

A Go polling loop firing every 500ms (or faster for LLM streaming) will call `runtime.EventsEmit` repeatedly. If React components subscribe broadly to state or events, every emit triggers a full component tree re-render.

### Strategy 1: Zustand Selector Granularity

The most important optimization. Components must subscribe only to the slice of state they use:

```typescript
// BAD — re-renders on any store change
const store = useChatStore();

// GOOD — re-renders only when messages array length changes
const messageCount = useChatStore(s => s.messages.length);

// GOOD — re-renders only when specific message's content changes
const content = useChatStore(s => s.messages.find(m => m.id === id)?.content);

// GOOD — for lists, use useShallow (Zustand v5)
import { useShallow } from "zustand/react/shallow";
const messageIds = useChatStore(useShallow(s => s.messages.map(m => m.id)));
```

### Strategy 2: React.memo for Stable List Items

Chat bubbles that are "done" (not streaming) should not re-render when new tokens arrive for a different message:

```typescript
export const ChatBubble = React.memo(function ChatBubble({ messageId }: { messageId: string }) {
    // Only subscribe to this message's content, not the whole list
    const content = useChatStore(s => s.messages.find(m => m.id === messageId)?.content ?? "");
    const isStreaming = useChatStore(s => s.messages.find(m => m.id === messageId)?.isStreaming ?? false);

    return <div className="chat-bubble">{/* render content */}</div>;
}, (prev, next) => prev.messageId === next.messageId);
```

The list itself renders only stable `messageId` items; each ChatBubble subscribes to its own slice.

### Strategy 3: Separate Streaming State from Committed State

Use a `useRef` for in-progress streaming content, and only commit to Zustand when "done" or on a throttled interval:

```typescript
function StreamingBubble({ messageId }: { messageId: string }) {
    const bufferRef = useRef("");
    const [displayContent, setDisplayContent] = useState("");

    useEffect(() => {
        const interval = setInterval(() => {
            setDisplayContent(bufferRef.current); // flush to DOM every 50ms
        }, 50);

        const off = EventsOn("chat:token", (chunk: TokenChunk) => {
            if (chunk.messageId === messageId) {
                bufferRef.current += chunk.tokens;
                if (chunk.done) {
                    clearInterval(interval);
                    setDisplayContent(bufferRef.current);
                    // Commit to store
                    useChatStore.getState().finalizeMessage(messageId, bufferRef.current);
                }
            }
        });

        return () => { off(); clearInterval(interval); };
    }, [messageId]);

    return <div>{displayContent}</div>;
}
```

This pattern reduces React state updates from N tokens/sec to 20 updates/sec (50ms interval) regardless of LLM output speed.

### Strategy 4: Terminal Output — Use xterm.js, Not React State

For the terminal tab panel, **do not route terminal output through React state at all**. Use `xterm.js` directly:

```bash
npm install @xterm/xterm @xterm/addon-fit @xterm/addon-web-links
```

```typescript
import { Terminal } from "@xterm/xterm";
import { FitAddon } from "@xterm/addon-fit";
import "@xterm/xterm/css/xterm.css";

function TerminalTab({ tabId }: { tabId: string }) {
    const termRef = useRef<HTMLDivElement>(null);
    const xtermRef = useRef<Terminal | null>(null);

    useEffect(() => {
        const term = new Terminal({ theme: { background: "#1a1a1a" } });
        const fitAddon = new FitAddon();
        term.loadAddon(fitAddon);
        term.open(termRef.current!);
        fitAddon.fit();
        xtermRef.current = term;

        // Write terminal output directly to xterm — bypasses React entirely
        const off = EventsOn(`terminal:output:${tabId}`, (line: string) => {
            term.writeln(line);
        });

        return () => { off(); term.dispose(); };
    }, [tabId]);

    return <div ref={termRef} style={{ height: "100%", width: "100%" }} />;
}
```

xterm.js manages its own virtual DOM and GPU-accelerated canvas renderer — it handles thousands of lines/second without React re-renders.

### Strategy 5: 500ms Go Polling Loop Architecture

For the Go side, structure the polling loop to minimize unnecessary events:

```go
func (t *TerminalService) StartPolling() {
    go func() {
        var lastOutput string
        ticker := time.NewTicker(500 * time.Millisecond)
        defer ticker.Stop()

        for range ticker.C {
            output := t.getLatestOutput()
            if output != lastOutput { // Only emit on change
                runtime.EventsEmit(t.ctx, "terminal:update", output)
                lastOutput = output
            }
        }
    }()
}
```

**Key rule:** Always diff before emitting. A Go loop that emits on every tick regardless of changes will cause unnecessary React work.

### Performance Summary

| Technique | Impact | Effort |
|-----------|--------|--------|
| Zustand selectors (granular) | HIGH — prevents cascade re-renders | Low |
| React.memo on list items | HIGH — stable items skip render | Low |
| ref buffer + 50ms flush for streaming | HIGH — reduces state updates 10-20x | Medium |
| xterm.js for terminal output | CRITICAL — terminal bypass React entirely | Medium |
| Diff before EventsEmit in Go | MEDIUM — reduces event frequency | Low |
| Immer in Zustand store | MEDIUM — structural sharing prevents false rerenders | Low |

---

## 9. Dependency Summary

### Go Dependencies

```go
// go.mod
require (
    github.com/wailsapp/wails/v2 v2.10.1
    // LLM client — depends on provider
)
```

### Frontend Dependencies

```json
{
  "dependencies": {
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "zustand": "^5.0.0",
    "jotai": "^2.10.0",
    "react-shiki": "^0.6.0",
    "react-markdown": "^9.0.0",
    "@xterm/xterm": "^5.5.0",
    "@xterm/addon-fit": "^0.10.0",
    "@xterm/addon-web-links": "^0.11.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "vite": "^5.4.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.0"
  }
}
```

### Build Prerequisites (Linux)

```bash
# Ubuntu 22.04
sudo apt install build-essential libgtk-3-dev libwebkit2gtk-4.0-dev

# Ubuntu 24.04
sudo apt install build-essential libgtk-3-dev libwebkit2gtk-4.1-dev
# Then build with: wails build -tags webkit2_41

# Also for AppImage packaging:
sudo apt install libfuse2  # required to run AppImage tools
```

---

## 10. Open Questions and Gaps

1. **xterm.js + WebKit2GTK GPU renderer:** The xterm.js WebGL renderer may not work in WebKit2GTK on all Linux configurations. Test with `rendererType: 'canvas'` as a fallback. The canvas renderer is slower but universally supported.

2. **Wails v3 migration timeline:** Wails v3 is in alpha (v3.0.0-alpha.74 as of early 2025). It features native nFPM packaging, a redesigned binding system, and multi-window support. For a production app starting now, use v2 — v3 is not yet GA and the API is still changing. Plan for eventual migration.

3. **Go→React file content streaming:** If PairAdmin needs to display file contents loaded from Go's filesystem, be aware that large payloads (>1MB) through EventsEmit or bound methods may stall the webview. Use chunked streaming via the event approach, or expose a static asset handler via Wails' `AssetServer` custom handler.

4. **react-shiki WASM initialization latency:** On first render, Shiki loads a WASM blob. In a Wails app this is loaded from embedded assets — typically 100-300ms. Use a fallback `<code>` block with CSS-based coloring while Shiki initializes to prevent a flash of unstyled code.

5. **CSP for security:** If PairAdmin executes user-provided commands, review whether the webview needs CSP hardening. Wails does not enforce CSP by default. Add a `<meta http-equiv="Content-Security-Policy">` with at minimum `default-src 'self'; script-src 'self' 'unsafe-inline'` (inline required for Wails runtime injection) to prevent XSS from LLM-generated HTML injection.

---

## Sources

- [Wails v2 — How Does It Work](https://wails.io/docs/howdoesitwork/) (official docs — HIGH confidence)
- [Wails v2 — Events Reference](https://wails.io/docs/reference/runtime/events/) (official docs — HIGH confidence)
- [Wails v2 — Application Development Guide](https://wails.io/docs/guides/application-development/) (official docs — HIGH confidence)
- [Wails v2 — Linux Distro Support](https://wails.io/docs/guides/linux-distro-support/) (official docs — HIGH confidence)
- [Wails v2 — Options Reference](https://wails.io/docs/reference/options/) (official docs — HIGH confidence)
- [GitHub Issue #2759 — EventsOn Inconsistent Data](https://github.com/wailsapp/wails/issues/2759) (MEDIUM confidence — community-confirmed bug)
- [GitHub Issue #2448 — Data Race in Events System](https://github.com/wailsapp/wails/issues/2448) (MEDIUM confidence)
- [GitHub Issue #3513 — libwebkit2gtk-4.0 Ubuntu 24](https://github.com/wailsapp/wails/issues/3513) (HIGH confidence — resolved)
- [GitHub Issue #4313 — AppImage WebKitNetworkProcess](https://github.com/wailsapp/wails/issues/4313) (MEDIUM confidence — open issue)
- [GitHub Discussion #758 — TypeScript Binding Generation](https://github.com/wailsapp/wails/discussions/758) (MEDIUM confidence — older discussion)
- [GitHub PR #3678 — Exported Fields Without JSON Tags](https://github.com/wailsapp/wails/pull/3678) (MEDIUM confidence)
- [GitHub PR #4481 — Correct nfpm.yaml Template Dependencies](https://github.com/wailsapp/wails/pull/4481) (MEDIUM confidence)
- [react-shiki GitHub README](https://github.com/AVGVSTVS96/react-shiki) (MEDIUM confidence — library docs)
- [Comparing Web Code Highlighters (chsm.dev, Jan 2025)](https://chsm.dev/blog/2025/01/08/comparing-web-code-highlighters) (MEDIUM confidence — independent benchmark)
- [Zustand vs Jotai vs Valtio Performance Guide (ReactLibraries, 2025)](https://www.reactlibraries.com/blog/zustand-vs-jotai-vs-valtio-performance-guide-2025) (MEDIUM confidence)
- [State Management in 2025 (DEV Community)](https://dev.to/hijazi313/state-management-in-2025-when-to-use-context-redux-zustand-or-jotai-2d2k) (LOW-MEDIUM confidence — community article)
- [Wails v2 Installation and Setup (DeepWiki)](https://deepwiki.com/wailsapp/wails/2.1-installation) (MEDIUM confidence)
- [Wails v3 Linux Packaging PR #3909](https://github.com/wailsapp/wails/pull/3909) (MEDIUM confidence — v3 alpha)
- [xterm.js Official Site](https://xtermjs.org/) (HIGH confidence — official)
