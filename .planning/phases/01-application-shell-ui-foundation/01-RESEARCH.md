# Phase 1: Application Shell & UI Foundation - Research

**Researched:** 2026-03-26
**Domain:** Wails v2 + React 19 + TypeScript + Tailwind v4 + shadcn/ui + xterm.js v6 + Zustand v5
**Confidence:** HIGH (core stack from official docs and verified sources)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Use Tailwind CSS as the CSS methodology (utility-first, fast iteration)
- Use shadcn/ui as the component base (headless, composable, TypeScript-native)
- Default to dark mode theme (fits terminal/sysadmin tool persona)
- Use Tailwind preflight as the CSS reset (comes included with Tailwind)
- Integrate xterm.js using `useRef` + `useEffect` pattern (standard imperative lifecycle for terminal libraries)
- Place terminal preview in the bottom section of the center column (~30% height, below chat) — shows terminal context in user's line of sight
- Include `@xterm/addon-fit` for automatic resize on window/column resize events
- Show a static "No terminal connected" welcome message on initial load to confirm the widget renders
- Seed mock terminal content with a realistic bash session (ls, git status output) to exercise the layout with real-looking content
- Show 2 mock tabs ("bash:1" and "bash:2") to exercise active/inactive tab states
- Mock chat response echoes the user's input text back (simple, confirms the UI flow works end-to-end)
- Pre-populate command sidebar with 3 mock commands to exercise card layout and hover tooltips

### Claude's Discretion
- Go module structure and directory layout (follow Wails v2 convention: `main.go` at root, `frontend/` subdirectory)
- Zustand store internal shape — 3 stores (chat, terminal, commands) with Immer produce-based mutations
- Zustand devtools middleware enabled in dev builds
- TypeScript strict mode enabled
- Wails window dimensions (1400x900 initial)

### Deferred Ideas (OUT OF SCOPE)
- Direct SSH client functionality (PairAdmin observes local terminal sessions including SSH; it does not initiate SSH connections itself)
- Light mode and system-preference theme switching (deferred to Phase 5 - Settings dialog)
- Resizable column splitter (deferred to Phase 5 - Settings & Config)
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| SHELL-01 | Application launches as a native desktop window using Wails v2 with React + TypeScript frontend | Wails v2.12.0 scaffold with react-ts template; build section covers exact commands |
| SHELL-02 | Three-column layout: terminal tabs (left, 160px fixed), chat area (center, flexible), command sidebar (right, 220px collapsible) | CSS Grid/Flexbox patterns documented; Tailwind v4 utility classes apply directly |
| SHELL-03 | Status bar displays active model, connection status, token usage, and settings button | Static placeholder values for Phase 1; Zustand store stub holds model/status state |
| SHELL-04 | Application builds and runs on Ubuntu 22.04+ with `libwebkit2gtk-4.1-dev` and `-tags webkit2_41` | Confirmed: Ubuntu 24.04 on dev machine; webkit2gtk-4.1-0 runtime present; dev headers MISSING (install step required) |
| CHAT-01 | User can type a question in the chat input and send it (Enter to send, Shift+Enter for newline) | Auto-expanding textarea pattern documented; keyboard handler in ChatInput component |
| CHAT-05 | Chat history is isolated per terminal tab; switching tabs shows that tab's conversation only | Zustand chatStore keyed by tabId; tab-switching clears/restores view |
| CHAT-06 | `/clear` command clears chat history for the current tab | Slash-command detection in input handler; chatStore.clearTab(tabId) action |
| CMD-01 | Every command block the AI suggests is automatically added to the command sidebar | Phase 1: mock commands pre-populated; real extraction wired in Phase 2 |
| CMD-02 | Commands in the sidebar are displayed in reverse-chronological order (newest at top) | commandStore array; unshift on add, or sort by timestamp desc |
| CMD-03 | Clicking a command in the sidebar copies it to the clipboard | Wails ClipboardSetText binding; fallback for Wayland detection |
| CMD-04 | Hovering over a sidebar command shows the original question that generated it | Tooltip component from shadcn/ui; each CommandCard stores originating question text |
| CMD-05 | "Clear History" button removes all commands from the sidebar for the current tab | commandStore.clearTab(tabId) action; button in sidebar footer |
| CLIP-01 | "Copy to Terminal" button copies the command to the system clipboard | Wails runtime.ClipboardSetText for X11; wl-clipboard exec for Wayland |
| CLIP-02 | Application detects Wayland display server at startup and warns if `wl-clipboard` is not installed | Go: os.Getenv("WAYLAND_DISPLAY") check; exec.LookPath("wl-copy"); EventsEmit warning to frontend |
</phase_requirements>

---

## Summary

Phase 1 builds a fully interactive desktop application skeleton using Wails v2 (Go backend + WebKit2GTK webview), a React 19 frontend with Tailwind CSS v4 and shadcn/ui components, xterm.js v6 for the terminal preview pane, and Zustand v5 + Immer for state management. No real terminal capture or LLM integration occurs — all data is mock/hardcoded. The goal is to verify the full rendering pipeline and interaction model before adding real data sources.

The key architectural challenge of this phase is correct integration of Wails v2's binding and event system with the React component tree, and setting up xterm.js v6 (which has significant breaking changes from v5) with the correct renderer for WebKit2GTK. On Ubuntu 24.04 (confirmed as the development machine), `libwebkit2gtk-4.1-dev` and `libgtk-3-dev` must be installed before `wails build` will succeed. The Wails CLI itself is also not yet installed.

The Tailwind + shadcn/ui stack has changed significantly: Tailwind v4 uses a Vite plugin (`@tailwindcss/vite`) rather than PostCSS, and there is a proven community Wails template (`Mahcks/wails-vite-react-tailwind-shadcnui-ts`) that validates this combination works. Clipboard support requires special handling for Wayland: the Go backend detects `WAYLAND_DISPLAY` and checks for `wl-copy` availability, warning the user if missing.

**Primary recommendation:** Scaffold with `wails init -n pairadmin -t react-ts`, then manually layer in Tailwind v4 + shadcn/ui using the Vite plugin approach. Do NOT use the community template directly (its wails.json and Go skeleton are minimal stubs), but use it as a reference for the Tailwind v4 + shadcn integration. Install dev headers first.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| github.com/wailsapp/wails/v2 | v2.12.0 | Go desktop framework, IPC bridge, lifecycle | Locked decision; only mature Go webview framework |
| react | 19.2.4 | UI rendering | Wails react-ts template ships React 18+; 19 is latest |
| react-dom | 19.2.4 | DOM renderer | Paired with react |
| typescript | 6.0.2 | Type safety | Locked decision; strict mode enabled |
| vite | 8.0.3 | Frontend bundler + HMR | Wails react-ts template default; Vite plugin for Tailwind v4 |
| tailwindcss | 4.2.2 | Utility-first CSS | Locked decision |
| @tailwindcss/vite | 4.2.2 | Tailwind v4 Vite integration | Replaces PostCSS approach in v4; required for v4 |
| @xterm/xterm | 6.0.0 | Terminal emulator widget | Locked decision; direct writes bypass React state |
| @xterm/addon-fit | 0.11.0 | Auto-resize xterm to container | Locked decision |
| @xterm/addon-canvas | 0.7.0 | Canvas 2D renderer for xterm | WebGL may fail in WebKit2GTK; canvas is the safe fallback |
| zustand | 5.0.12 | Frontend state management | Locked decision; 3 stores |
| immer | 11.1.4 | Immutable state mutations for Zustand | Locked decision; Immer produce-based mutations |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| class-variance-authority | 0.7.1 | Variant-based className composition | Required by shadcn/ui components |
| clsx | 2.1.1 | Conditional className merging | Required by shadcn/ui |
| tailwind-merge | 3.5.0 | Merge Tailwind classes without conflicts | Required by shadcn/ui |
| lucide-react | 1.7.0 | Icon library | Default icon set for shadcn/ui |
| @vitejs/plugin-react | 6.0.1 | React JSX transform for Vite | Required by Vite config |
| @types/react | 19.2.14 | TypeScript types for React | Dev dependency |
| @types/node | 25.5.0 | TypeScript types for Node (path resolver in vite.config) | Required for `path.resolve` in vite.config.ts |
| @xterm/addon-webgl | 0.19.0 | WebGL2 renderer for xterm | Optional perf boost; use only if WebGL confirmed working in WebKit2GTK |

**shadcn/ui components to add (via CLI):**
- `button` — send button, clear history, copy buttons
- `tooltip` — command card hover tooltip (CMD-04)
- `badge` — tab active/inactive state indicator
- `separator` — column dividers
- `scroll-area` — chat message list, command sidebar

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| @xterm/addon-canvas | @xterm/addon-webgl | WebGL is faster but may fail silently in WebKit2GTK; canvas 2D is universally safe |
| Tailwind v4 Vite plugin | postcss/tailwind v3 | PostCSS works but v4 is current; Vite plugin approach is simpler and officially recommended |
| Zustand + Immer | Jotai atoms | Jotai excels at fine-grained atoms but ordered list mutations (chat messages) are cleaner with Immer; locked decision |

### Installation

```bash
# 1. Install Wails CLI
go install github.com/wailsapp/wails/v2/cmd/wails@latest

# 2. Install Ubuntu 24.04 build prerequisites
sudo apt install build-essential libgtk-3-dev libwebkit2gtk-4.1-dev

# 3. Scaffold project
wails init -n pairadmin -t react-ts
cd pairadmin

# 4. Add frontend dependencies
cd frontend
npm install tailwindcss @tailwindcss/vite
npm install zustand immer
npm install @xterm/xterm @xterm/addon-fit @xterm/addon-canvas @xterm/addon-webgl
npm install class-variance-authority clsx tailwind-merge lucide-react
npm install -D @types/node

# 5. Initialize shadcn/ui
npx shadcn@latest init
# When prompted: select "dark" style, CSS variables yes
npx shadcn@latest add button tooltip badge separator scroll-area
```

**Version verification:** All versions above verified against npm registry on 2026-03-26.

---

## Architecture Patterns

### Recommended Project Structure

```
pairadmin/
  main.go                          # wails.Run entry point, window config
  app.go                           # App struct (lifecycle, Wayland detection)
  services/
    chat.go                        # ChatService (mock echo for Phase 1)
    terminal.go                    # TerminalService (stub for Phase 1)
    commands.go                    # CommandService (clipboard copy, Wayland check)
  wails.json                       # Wails project config
  go.mod
  go.sum
  frontend/
    index.html
    src/
      main.tsx                     # React root, ThemeProvider
      App.tsx                      # Three-column layout shell
      components/
        layout/
          ThreeColumnLayout.tsx    # CSS Grid root layout
          StatusBar.tsx            # Bottom status bar (SHELL-03)
        terminal/
          TerminalTabList.tsx      # Left column: tab list (SHELL-02)
          TerminalTab.tsx          # Individual tab button (active/inactive)
          TerminalPreview.tsx      # xterm.js widget (useRef + useEffect)
        chat/
          ChatPane.tsx             # Center column wrapper
          ChatMessageList.tsx      # Scrollable message list
          ChatBubble.tsx           # Single message bubble (user/assistant)
          ChatInput.tsx            # Auto-expanding textarea (CHAT-01)
        sidebar/
          CommandSidebar.tsx       # Right column wrapper
          CommandCard.tsx          # Single command card with tooltip (CMD-04)
          ClearHistoryButton.tsx   # CMD-05
        ui/                        # shadcn/ui generated components (do not hand-edit)
      stores/
        chatStore.ts               # Zustand: messages[] keyed by tabId
        terminalStore.ts           # Zustand: tabs[], activeTabId
        commandStore.ts            # Zustand: commands[] keyed by tabId
      hooks/
        useWailsClipboard.ts       # Abstraction over ClipboardSetText
        useKeyboardShortcuts.ts    # Enter-to-send, Shift+Enter newline
      lib/
        utils.ts                   # cn() helper (shadcn/ui standard)
      theme/
        theme-provider.tsx         # Dark mode ThemeProvider
      wailsjs/                     # AUTO-GENERATED — do not edit
        go/main/
        runtime/
    package.json
    vite.config.ts
    tsconfig.json
    tsconfig.app.json
    components.json                # shadcn/ui config
```

### Pattern 1: Wails App Initialization (main.go)

**What:** Wire Go service structs, configure window, set background color to prevent white flash.
**When to use:** Project scaffold; set up once.

```go
// Source: Wails official docs + RESEARCH-WAILS.md
package main

import (
    "context"
    "embed"

    "github.com/wailsapp/wails/v2"
    "github.com/wailsapp/wails/v2/pkg/options"
    "github.com/wailsapp/wails/v2/pkg/options/assetserver"
)

//go:embed all:frontend/dist
var assets embed.FS

func main() {
    app := NewApp()
    commands := NewCommandService()

    err := wails.Run(&options.App{
        Title:  "PairAdmin",
        Width:  1400,
        Height: 900,
        // Prevents white flash before webview renders on Linux
        BackgroundColour: &options.RGBA{R: 18, G: 18, B: 18, A: 255},
        AssetServer: &assetserver.Options{
            Assets: assets,
        },
        OnStartup: func(ctx context.Context) {
            app.startup(ctx)
            commands.SetContext(ctx)
        },
        Bind: []interface{}{app, commands},
    })
    if err != nil {
        println("Error:", err.Error())
    }
}
```

### Pattern 2: Wails Background Color (Critical for Linux)

**What:** Set `BackgroundColour` to match CSS background to prevent white/grey flash on startup.
**When to use:** Always on Linux builds; optional on other platforms.

The Wails v2.12.0 release included a fix for Linux crash on panic in JS-bound Go methods. With this version, the `BackgroundColour` option remains the primary mitigation for the startup flash.

### Pattern 3: xterm.js v6 Setup with Canvas Fallback

**What:** Mount xterm.js in a React component using imperative lifecycle. Use `@xterm/addon-canvas` as the renderer (not WebGL) because WebKit2GTK does not reliably support WebGL2.
**When to use:** TerminalPreview component.

```typescript
// Source: xterm.js official docs + RESEARCH-WAILS.md adapted for v6
import { useEffect, useRef } from "react";
import { Terminal } from "@xterm/xterm";
import { FitAddon } from "@xterm/addon-fit";
import { CanvasAddon } from "@xterm/addon-canvas";
import "@xterm/xterm/css/xterm.css";

interface TerminalPreviewProps {
  tabId: string;
}

export function TerminalPreview({ tabId }: TerminalPreviewProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  const termRef = useRef<Terminal | null>(null);

  useEffect(() => {
    if (!containerRef.current) return;

    const term = new Terminal({
      theme: {
        background: "#0d0d0d",
        foreground: "#d4d4d4",
        cursor: "#d4d4d4",
      },
      fontSize: 13,
      fontFamily: "'JetBrains Mono', 'Fira Code', monospace",
      scrollback: 1000,
      convertEol: true,
    });

    const fitAddon = new FitAddon();
    const canvasAddon = new CanvasAddon();

    term.loadAddon(fitAddon);
    term.open(containerRef.current);
    term.loadAddon(canvasAddon); // Load AFTER open()
    fitAddon.fit();

    // Static mock content for Phase 1
    term.writeln("\x1b[32m$ \x1b[0mls -la /home/admin");
    term.writeln("total 48");
    term.writeln("drwxr-xr-x  6 admin admin 4096 Mar 26 09:12 .");
    term.writeln("\x1b[32m$ \x1b[0mgit status");
    term.writeln("On branch main");
    term.writeln("nothing to commit, working tree clean");
    term.writeln("\x1b[33m[No terminal connected — Phase 1 mock]\x1b[0m");

    termRef.current = term;

    const resizeObserver = new ResizeObserver(() => fitAddon.fit());
    resizeObserver.observe(containerRef.current);

    return () => {
      resizeObserver.disconnect();
      term.dispose();
    };
  }, [tabId]);

  return (
    <div
      ref={containerRef}
      className="h-full w-full overflow-hidden"
      style={{ minHeight: "120px" }}
    />
  );
}
```

**Critical:** `CanvasAddon` must be loaded AFTER `term.open()`. Loading before open results in a silent no-op and the terminal renders without acceleration.

### Pattern 4: Zustand Store with Immer (chatStore)

**What:** Zustand v5 store with Immer middleware; tab-keyed message history.
**When to use:** Chat state management.

```typescript
// Source: RESEARCH-WAILS.md + Zustand v5 docs
import { create } from "zustand";
import { immer } from "zustand/middleware/immer";
import { devtools } from "zustand/middleware";

export interface ChatMessage {
  id: string;
  role: "user" | "assistant";
  content: string;
  isStreaming: boolean;
}

interface ChatState {
  // Keyed by tabId
  messagesByTab: Record<string, ChatMessage[]>;
  addUserMessage: (tabId: string, text: string) => string;
  addAssistantMessage: (tabId: string, content: string) => string;
  clearTab: (tabId: string) => void;
}

export const useChatStore = create<ChatState>()(
  devtools(
    immer((set) => ({
      messagesByTab: {},
      addUserMessage: (tabId, text) => {
        const id = crypto.randomUUID();
        set((state) => {
          if (!state.messagesByTab[tabId]) state.messagesByTab[tabId] = [];
          state.messagesByTab[tabId].push({
            id, role: "user", content: text, isStreaming: false,
          });
        });
        return id;
      },
      addAssistantMessage: (tabId, content) => {
        const id = crypto.randomUUID();
        set((state) => {
          if (!state.messagesByTab[tabId]) state.messagesByTab[tabId] = [];
          state.messagesByTab[tabId].push({
            id, role: "assistant", content, isStreaming: false,
          });
        });
        return id;
      },
      clearTab: (tabId) => {
        set((state) => { state.messagesByTab[tabId] = []; });
      },
    })),
    { name: "chat-store" }
  )
);
```

**Zustand v5 note:** Import `useShallow` from `"zustand/react/shallow"` (not `"zustand/shallow"` — moved in v5).

### Pattern 5: Three-Column Layout (CSS)

**What:** Fixed-width left and right columns; flexible center.
**When to use:** Root App.tsx layout.

```tsx
// Tailwind v4 utility classes
<div className="flex h-screen w-screen overflow-hidden bg-zinc-950 text-zinc-100">
  {/* Left: Terminal tabs — 160px fixed */}
  <aside className="w-40 flex-none border-r border-zinc-800 overflow-y-auto">
    <TerminalTabList />
  </aside>

  {/* Center: Chat + Terminal preview — flexible */}
  <main className="flex flex-1 flex-col overflow-hidden">
    {/* Chat area — flex-1 (grows to fill) */}
    <div className="flex flex-1 flex-col overflow-hidden">
      <ChatMessageList />
      <ChatInput />
    </div>
    {/* Terminal preview — 30% of center column */}
    <div className="h-[30%] border-t border-zinc-800">
      <TerminalPreview tabId={activeTabId} />
    </div>
  </main>

  {/* Right: Command sidebar — 220px fixed */}
  <aside className="w-[220px] flex-none border-l border-zinc-800 overflow-y-auto">
    <CommandSidebar />
  </aside>
</div>
```

### Pattern 6: Tailwind v4 Configuration

**What:** Tailwind v4 uses a Vite plugin instead of PostCSS. No `tailwind.config.js` required by default.
**When to use:** `vite.config.ts` and `src/index.css` setup.

```typescript
// vite.config.ts
import path from "path";
import tailwindcss from "@tailwindcss/vite";
import react from "@vitejs/plugin-react";
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: { "@": path.resolve(__dirname, "./src") },
  },
});
```

```css
/* src/index.css — minimal v4 import */
@import "tailwindcss";
```

```json
// tsconfig.app.json — add for path alias
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

### Pattern 7: shadcn/ui Dark Mode Default

**What:** ThemeProvider that defaults to dark and persists to localStorage.
**When to use:** `src/main.tsx` wrapping the root.

```tsx
// src/theme/theme-provider.tsx
import { createContext, useContext, useEffect, useState } from "react";

type Theme = "dark" | "light" | "system";
const ThemeContext = createContext<{ theme: Theme; setTheme: (t: Theme) => void }>({
  theme: "dark",
  setTheme: () => null,
});

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>(
    () => (localStorage.getItem("pairadmin-theme") as Theme) || "dark"
  );

  useEffect(() => {
    const root = window.document.documentElement;
    root.classList.remove("light", "dark");
    root.classList.add(theme === "system"
      ? window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light"
      : theme);
  }, [theme]);

  return (
    <ThemeContext.Provider value={{ theme, setTheme: (t) => {
      localStorage.setItem("pairadmin-theme", t);
      setTheme(t);
    }}}>
      {children}
    </ThemeContext.Provider>
  );
}

export const useTheme = () => useContext(ThemeContext);
```

### Pattern 8: Wayland Clipboard Detection (Go)

**What:** At startup, detect Wayland and check for `wl-copy`; emit a warning event if missing.
**When to use:** `app.go` startup hook.

```go
// services/commands.go
import (
    "os"
    "os/exec"
    "github.com/wailsapp/wails/v2/pkg/runtime"
)

func (c *CommandService) startup(ctx context.Context) {
    c.ctx = ctx
    if os.Getenv("WAYLAND_DISPLAY") != "" {
        if _, err := exec.LookPath("wl-copy"); err != nil {
            runtime.EventsEmit(ctx, "app:warning", map[string]string{
                "code":    "WAYLAND_CLIPBOARD",
                "message": "wl-clipboard not found. Install it with: sudo apt install wl-clipboard",
            })
        }
    }
}

func (c *CommandService) CopyToClipboard(text string) error {
    if os.Getenv("WAYLAND_DISPLAY") != "" {
        cmd := exec.Command("wl-copy", text)
        return cmd.Run()
    }
    runtime.ClipboardSetText(c.ctx, text)
    return nil
}
```

### Anti-Patterns to Avoid

- **Routing terminal output through React state:** xterm.js must receive writes directly via `term.write()`. Never store terminal buffer content in Zustand — it causes severe render churn. React state is only for tab metadata (name, id, active).
- **Using zustand v4 import paths in v5:** `import { shallow } from "zustand/shallow"` was removed in v5. Use `import { useShallow } from "zustand/react/shallow"`.
- **Loading CanvasAddon before term.open():** The canvas addon attaches to the DOM node created by `open()`. Loading it before results in no-op without error.
- **Calling `fitAddon.fit()` before the container has dimensions:** The fit addon reads `offsetWidth`/`offsetHeight` from the DOM. If the container is hidden or zero-sized, fit silently fails. Always call `fit()` after the component is mounted and visible.
- **`wails dev` before wailsjs/ directory exists:** The first run generates `frontend/wailsjs/`. TypeScript will fail to compile if you import from `wailsjs/` before running `wails dev` once. Add `frontend/wailsjs/` to `.gitignore` or commit the generated stubs.
- **Importing from `"zustand/middleware"` for immer:** In v5, import as `import { immer } from "zustand/middleware/immer"` not `"zustand/middleware"`.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Terminal emulator rendering | Custom canvas/DOM terminal | `@xterm/xterm` + `@xterm/addon-canvas` | xterm handles ANSI sequences, scrollback, selection, copy — thousands of edge cases |
| Accessible UI components | Custom Button/Tooltip/ScrollArea | `shadcn/ui` via CLI | Focus management, keyboard navigation, WAI-ARIA attributes, dark mode support |
| Immutable state mutations | Manual spread/Object.assign | `immer` via zustand middleware | Immer handles nested object mutations with structural sharing; prevents false re-renders |
| Clipboard on Wayland | Custom xdg-clipboard wrapper | `wl-copy` via exec + `runtime.ClipboardSetText` for X11 | Wayland clipboard APIs are complex; wl-clipboard handles protocol negotiation |
| Container-fit terminal resize | ResizeObserver + manual width/height calc | `@xterm/addon-fit` | FitAddon correctly accounts for character cell dimensions, padding, scrollbar width |
| Icon library | SVG sprite or inline SVGs | `lucide-react` | shadcn/ui uses lucide as default; consistent icon weight/style across app |
| Class name merging | Manual string concatenation | `clsx` + `tailwind-merge` via `cn()` helper | Tailwind class conflicts (e.g., `p-4 p-2`) require merge; clsx handles conditionals |

**Key insight:** Tailwind v4 + shadcn/ui + xterm.js represent years of iteration on exactly the problems Phase 1 faces. Any custom solution for these will be worse and slower to build.

---

## Common Pitfalls

### Pitfall 1: Ubuntu 24.04 Missing Dev Headers

**What goes wrong:** `wails build` fails with `cannot find package "github.com/wailsapp/wails/v2/internal/frontend/desktop/linux"` or pkg-config error `Package 'webkit2gtk-4.0' not found`.
**Why it happens:** Ubuntu 24.04 dropped `libwebkit2gtk-4.0-dev`; only `libwebkit2gtk-4.1-dev` is available. The confirmed dev machine is Ubuntu 24.04. `libwebkit2gtk-4.1-dev` and `libgtk-3-dev` are NOT currently installed (verified 2026-03-26).
**How to avoid:** Install before building:
```bash
sudo apt install build-essential libgtk-3-dev libwebkit2gtk-4.1-dev
```
Build with tag: `wails build -tags webkit2_41`
**Warning signs:** pkg-config exits non-zero; error mentions `webkit2gtk-4.0` not found.

### Pitfall 2: xterm.js CanvasAddon v6 — Load Order

**What goes wrong:** Terminal renders without canvas acceleration, or throws "Cannot call open before constructing Terminal" (in older versions) or silently fails.
**Why it happens:** `CanvasAddon` (in v6, moved to `@xterm/addon-canvas` as a separate package, not removed from the ecosystem as the GitHub release notes suggested) must be loaded via `term.loadAddon()` AFTER `term.open()`.
**How to avoid:** Always order: `new Terminal()` → `loadAddon(fitAddon)` → `term.open(el)` → `loadAddon(canvasAddon)` → `fitAddon.fit()`.
**Warning signs:** Terminal displays but looks different from expected; no error in console.

### Pitfall 3: Wails CLI Not Installed

**What goes wrong:** `wails` command not found; build cannot start.
**Why it happens:** The Wails CLI is not in `~/go/bin` (verified: not installed on dev machine).
**How to avoid:** `go install github.com/wailsapp/wails/v2/cmd/wails@latest` and ensure `~/go/bin` is in `$PATH`.
**Warning signs:** `command not found: wails`.

### Pitfall 4: Tailwind v4 vs v3 Config Mismatch

**What goes wrong:** `tailwind.config.js` or `postcss.config.js` created for v3; v4 Vite plugin ignores them. Classes appear but theme customization doesn't apply.
**Why it happens:** Tailwind v4 (4.2.2, current) uses `@tailwindcss/vite` plugin and CSS-first configuration via `@theme` directive in CSS. There is no `tailwind.config.js` by default in v4.
**How to avoid:** Use only the Vite plugin approach. For custom theme values, use CSS `@theme` blocks in `index.css`. For shadcn/ui compatibility, run `npx shadcn@latest init` which handles the CSS variable setup.
**Warning signs:** `tailwind.config.js` content not being applied; no error, just silent style mismatch.

### Pitfall 5: Wails wailsjs/ Import Before First Dev Run

**What goes wrong:** TypeScript compile error `Cannot find module '../wailsjs/go/main/CommandService'`.
**Why it happens:** `wailsjs/` is code-generated by `wails dev` and does not exist until the first run.
**How to avoid:** Run `wails dev` once before writing frontend code that imports from `wailsjs/`. Alternatively, create stub type declarations for Phase 1 mock mode.
**Warning signs:** TypeScript errors on wailsjs imports on a fresh checkout.

### Pitfall 6: shadcn/ui `components.json` Must Match Tailwind Version

**What goes wrong:** `npx shadcn@latest init` prompts for CSS framework and generates incorrect configuration for Tailwind v4.
**Why it happens:** shadcn/ui v0.9.5+ supports Tailwind v4 but requires selecting the correct option during init. If "Tailwind v3" is selected, CSS variable generation is incompatible.
**How to avoid:** During `npx shadcn@latest init`, select Tailwind v4 style. Verify `components.json` has `"tailwind": { "version": "4" }`.
**Warning signs:** Generated `globals.css` imports fail; variables not resolved.

### Pitfall 7: Zustand v5 Shallow Import Path Change

**What goes wrong:** Runtime error or TypeScript error: `Property 'shallow' does not exist on exported object`.
**Why it happens:** Zustand v5 moved `shallow` from `"zustand/shallow"` to `"zustand/react/shallow"` as `useShallow`.
**How to avoid:** Always use `import { useShallow } from "zustand/react/shallow"` for selector memoization in v5.
**Warning signs:** Excessive re-renders on list stores; TypeScript import error on `zustand/shallow`.

### Pitfall 8: EventsOn Goroutine Leak (React useEffect)

**What goes wrong:** Memory leak; event handlers accumulate across component mounts.
**Why it happens:** Wails `EventsOn` returns a cleanup function that must be called explicitly. React's `useEffect` cleanup fires on unmount.
**How to avoid:**
```typescript
useEffect(() => {
  const off = EventsOn("app:warning", handleWarning);
  return off; // off IS the cleanup function — return directly
}, []);
```
**Warning signs:** Multiple handler calls per event; growing memory usage.

---

## Code Examples

### Chat Input with Enter-to-Send / Shift+Enter Newline

```typescript
// Source: Standard React pattern
function ChatInput({ onSend }: { onSend: (text: string) => void }) {
  const [value, setValue] = useState("");
  const textareaRef = useRef<HTMLTextAreaElement>(null);

  const handleKeyDown = (e: React.KeyboardEvent<HTMLTextAreaElement>) => {
    if (e.key === "Enter" && !e.shiftKey) {
      e.preventDefault();
      if (value.trim()) {
        onSend(value.trim());
        setValue("");
        // Reset textarea height
        if (textareaRef.current) textareaRef.current.style.height = "auto";
      }
    }
  };

  const handleInput = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    setValue(e.target.value);
    // Auto-expand height
    e.target.style.height = "auto";
    e.target.style.height = `${Math.min(e.target.scrollHeight, 200)}px`;
  };

  return (
    <div className="border-t border-zinc-800 p-3">
      <textarea
        ref={textareaRef}
        value={value}
        onChange={handleInput}
        onKeyDown={handleKeyDown}
        placeholder="Ask about the terminal output... (Enter to send, Shift+Enter for newline)"
        className="w-full resize-none bg-zinc-900 text-zinc-100 rounded-md px-3 py-2 text-sm placeholder-zinc-500 focus:outline-none focus:ring-1 focus:ring-zinc-600 min-h-[40px] max-h-[200px]"
        rows={1}
      />
    </div>
  );
}
```

### Command Sidebar Card with Tooltip (shadcn/ui)

```typescript
// Source: shadcn/ui Tooltip docs + CMD-03/CMD-04 requirements
import { Tooltip, TooltipContent, TooltipProvider, TooltipTrigger } from "@/components/ui/tooltip";

interface CommandCardProps {
  command: string;
  originalQuestion: string;
  onCopy: (cmd: string) => void;
}

export function CommandCard({ command, originalQuestion, onCopy }: CommandCardProps) {
  return (
    <TooltipProvider>
      <Tooltip>
        <TooltipTrigger asChild>
          <button
            onClick={() => onCopy(command)}
            className="w-full text-left px-3 py-2 text-xs font-mono bg-zinc-900 hover:bg-zinc-800 rounded border border-zinc-800 hover:border-zinc-700 transition-colors truncate"
          >
            {command}
          </button>
        </TooltipTrigger>
        <TooltipContent side="left" className="max-w-[200px] text-xs">
          <p className="font-medium text-zinc-400">Generated from:</p>
          <p>{originalQuestion}</p>
        </TooltipContent>
      </Tooltip>
    </TooltipProvider>
  );
}
```

### Mock Echo Response (Phase 1 Hardcoded AI Response)

```typescript
// In App.tsx or ChatPane.tsx — mock chat flow for Phase 1 exit criteria
function handleUserMessage(tabId: string, text: string) {
  const { addUserMessage, addAssistantMessage } = useChatStore.getState();

  // Add user message
  addUserMessage(tabId, text);

  // Detect /clear command
  if (text.trim() === "/clear") {
    useChatStore.getState().clearTab(tabId);
    return;
  }

  // Hardcoded echo response — replaced by real LLM in Phase 2
  setTimeout(() => {
    addAssistantMessage(tabId, `Echo: ${text}`);
  }, 200);
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Tailwind PostCSS config + `tailwind.config.js` | `@tailwindcss/vite` plugin + CSS-first `@theme` directive | Tailwind v4.0 (Jan 2025) | No config file needed; significantly simpler setup |
| `xterm.js` with built-in canvas renderer (`rendererType: 'canvas'`) | `@xterm/addon-canvas` as separate addon | xterm.js v6.0.0 (breaking change) | Must install `@xterm/addon-canvas` separately; load after `open()` |
| `import { shallow } from "zustand/shallow"` | `import { useShallow } from "zustand/react/shallow"` | Zustand v5 (late 2024) | Import path changed; old path no longer exists |
| Wails v2.10.1 | Wails v2.12.0 | Nov 2024 → Feb 2025 | Linux crash on panic in JS-bound methods fixed (Issue #3965); macOS WebView crash fix |
| React 18 | React 19.2.4 | React 19 GA (Dec 2024) | Wails react-ts template may still scaffold React 18; upgrade manually if needed |

**Deprecated/outdated:**
- `rendererType: 'canvas'` option in Terminal constructor: removed in xterm.js v6; use `@xterm/addon-canvas` addon instead
- `tailwind.config.js` content detection: not used in v4; auto-detected via Vite plugin
- `zustand/shallow`: removed in v5; use `zustand/react/shallow`

---

## Open Questions

1. **xterm.js v6 + WebKit2GTK WebGL availability**
   - What we know: `@xterm/addon-webgl` requires WebGL2; WebKit2GTK on Ubuntu 24.04 may support it (ANGLE-based), but this is unconfirmed for this specific setup
   - What's unclear: Whether `@xterm/addon-webgl` works in the specific WebKit2GTK build shipped with Ubuntu 24.04's webkit2gtk-4.1-0 (v2.50.4)
   - Recommendation: Start with `@xterm/addon-canvas` (always works). After Phase 1 is functional, optionally test WebGL addon with try/catch fallback. Do not block Phase 1 on this.

2. **Wails react-ts template React version**
   - What we know: Wails react-ts template has shipped React 18 historically; React 19.2.4 is current
   - What's unclear: Whether the official react-ts template scaffold generates React 18 or 19 package.json
   - Recommendation: After scaffolding, update `package.json` to React 19 explicitly. No breaking changes for the patterns used in Phase 1.

3. **shadcn/ui `init` prompts with Tailwind v4**
   - What we know: `shadcn@latest` (v0.9.5) supports Tailwind v4 but the init flow has prompts that must be answered correctly
   - What's unclear: Whether the prompts auto-detect installed Tailwind version or require explicit selection
   - Recommendation: After running init, verify `components.json` shows Tailwind v4 configuration. If CSS variables are generated as v3 format, re-run init.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Go | Wails backend | ✓ | go1.22.2 | — |
| Node.js | Frontend build | ✓ | v24.12.0 | — |
| npm | Frontend deps | ✓ | 11.6.2 | — |
| wails CLI | Project scaffold + build | ✗ | — | Install via `go install github.com/wailsapp/wails/v2/cmd/wails@latest` |
| libwebkit2gtk-4.1-0 (runtime) | WebKit2GTK webview | ✓ | 2.50.4 | — |
| libwebkit2gtk-4.1-dev (headers) | `wails build` compilation | ✗ | — | Install via `sudo apt install libwebkit2gtk-4.1-dev` |
| libgtk-3-dev | `wails build` compilation | ✗ | — | Install via `sudo apt install libgtk-3-dev` |
| build-essential | C compilation for CGO | Unknown | — | Install via `sudo apt install build-essential` |
| wl-clipboard (wl-copy) | Wayland clipboard (CLIP-01/02) | Unknown | — | App warns user at startup if missing (CLIP-02 requirement) |

**Missing dependencies with no fallback:**
- `wails CLI` — must be installed before any scaffold or build step
- `libwebkit2gtk-4.1-dev` — required for CGO compilation; no fallback
- `libgtk-3-dev` — required for CGO compilation; no fallback

**Missing dependencies with fallback:**
- `wl-clipboard` — only required on Wayland sessions; CLIP-02 implements detection + user warning

**Wave 0 install task:** The plan MUST include a task to install system build prerequisites before any Go/Wails compilation task.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Vitest (via Vite ecosystem; standard for Wails React-TS projects) |
| Config file | `frontend/vite.config.ts` (add `test` block) or `frontend/vitest.config.ts` |
| Quick run command | `cd frontend && npm run test -- --run` |
| Full suite command | `cd frontend && npm run test -- --run --coverage` |

**Note:** Go backend tests use `go test ./...`. Wails app integration testing (does the window launch) is a manual smoke test for Phase 1 — no automated headless webview testing framework is standard for Wails v2.

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SHELL-01 | App launches as native window | smoke | Manual: `wails dev` + visual check | ❌ Wave 0 (manual only) |
| SHELL-02 | Three-column layout correct proportions | unit | `npm run test -- --run layout` | ❌ Wave 0 |
| SHELL-03 | Status bar renders placeholder content | unit | `npm run test -- --run status-bar` | ❌ Wave 0 |
| SHELL-04 | Build succeeds on Ubuntu 24.04 | smoke | `wails build -tags webkit2_41` (exit 0) | ❌ Wave 0 (build script) |
| CHAT-01 | Enter sends; Shift+Enter inserts newline | unit | `npm run test -- --run chat-input` | ❌ Wave 0 |
| CHAT-05 | Tab switch restores isolated history | unit | `npm run test -- --run chat-store` | ❌ Wave 0 |
| CHAT-06 | /clear empties current tab history | unit | `npm run test -- --run chat-store` | ❌ Wave 0 |
| CMD-01 | Mock commands pre-populated in sidebar | unit | `npm run test -- --run command-store` | ❌ Wave 0 |
| CMD-02 | Commands display newest-first | unit | `npm run test -- --run command-store` | ❌ Wave 0 |
| CMD-03 | Click card calls clipboard copy | unit | `npm run test -- --run command-card` | ❌ Wave 0 |
| CMD-04 | Hover shows originating question | unit | `npm run test -- --run command-card` | ❌ Wave 0 |
| CMD-05 | Clear History button empties sidebar | unit | `npm run test -- --run command-store` | ❌ Wave 0 |
| CLIP-01 | CopyToClipboard called with command text | unit | Go: `go test ./services/... -run TestCopyToClipboard` | ❌ Wave 0 |
| CLIP-02 | Wayland warning event emitted when wl-copy missing | unit | Go: `go test ./services/... -run TestWaylandDetection` | ❌ Wave 0 |

### Sampling Rate
- **Per task commit:** `cd frontend && npm run test -- --run`
- **Per wave merge:** `cd frontend && npm run test -- --run && go test ./...`
- **Phase gate:** Full suite green + manual smoke test (`wails dev` launches, tabs clickable, echo response works, clipboard copy works) before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `frontend/src/stores/__tests__/chatStore.test.ts` — covers CHAT-05, CHAT-06
- [ ] `frontend/src/stores/__tests__/commandStore.test.ts` — covers CMD-01, CMD-02, CMD-05
- [ ] `frontend/src/components/__tests__/ChatInput.test.tsx` — covers CHAT-01
- [ ] `frontend/src/components/__tests__/CommandCard.test.tsx` — covers CMD-03, CMD-04
- [ ] `frontend/src/components/__tests__/ThreeColumnLayout.test.tsx` — covers SHELL-02
- [ ] `services/commands_test.go` — covers CLIP-01, CLIP-02
- [ ] Vitest install: `npm install -D vitest @testing-library/react @testing-library/user-event @testing-library/jest-dom jsdom`
- [ ] `frontend/vite.config.ts` test block: add `test: { environment: "jsdom", globals: true }`

---

## Sources

### Primary (HIGH confidence)
- Wails v2 official docs (https://wails.io/docs/) — binding system, event API, options reference
- Wails v2.12.0 release notes (https://github.com/wailsapp/wails/releases/tag/v2.12.0) — Linux crash fix confirmed
- npm registry — all package versions verified 2026-03-26: zustand@5.0.12, immer@11.1.4, @xterm/xterm@6.0.0, @xterm/addon-fit@0.11.0, @xterm/addon-canvas@0.7.0, @xterm/addon-webgl@0.19.0, tailwindcss@4.2.2, @tailwindcss/vite@4.2.2, react@19.2.4, vite@8.0.3, typescript@6.0.2
- shadcn/ui Vite installation docs (https://ui.shadcn.com/docs/installation/vite) — Tailwind v4 Vite plugin, path alias config
- shadcn/ui dark mode docs (https://ui.shadcn.com/docs/dark-mode/vite) — ThemeProvider setup

### Secondary (MEDIUM confidence)
- RESEARCH-WAILS.md (`.planning/research/RESEARCH-WAILS.md`) — verified against official sources; xterm.js renderer pattern updated for v6
- Wails community template (https://github.com/Mahcks/wails-vite-react-tailwind-shadcnui-ts) — validated Tailwind v4 + shadcn/ui + Wails combination works; versions slightly behind current
- xterm.js v6.0.0 release notes (GitHub) — canvas renderer moved to `@xterm/addon-canvas` as separate package

### Tertiary (LOW confidence)
- WebSearch results on xterm.js v6 renderer behavior in WebKit2GTK — needs runtime validation in Phase 1

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all versions verified against npm registry 2026-03-26
- Architecture: HIGH — patterns from RESEARCH-WAILS.md + official Wails docs + shadcn/ui docs
- Pitfalls: HIGH — Wails pitfalls from official docs/issues; xterm v6 breaking change verified; Ubuntu 24.04 environment verified on dev machine
- Test framework: MEDIUM — Vitest is standard for Vite React projects but not Wails-specific; Go test patterns are standard

**Research date:** 2026-03-26
**Valid until:** 2026-06-26 (stable stack; Tailwind/shadcn move fast but core APIs stable for 90 days)
