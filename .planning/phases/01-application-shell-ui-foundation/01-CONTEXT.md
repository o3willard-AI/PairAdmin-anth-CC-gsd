# Phase 1: Application Shell & UI Foundation - Context

**Gathered:** 2026-03-26
**Status:** Ready for planning

<domain>
## Phase Boundary

Build a working Wails v2 + React + TypeScript desktop application with the three-column layout (terminal tabs left, chat center, command sidebar right), mock terminal tabs, static chat UI, xterm.js terminal preview pane, and clipboard support. No real terminal capture or LLM integration yet — the skeleton is interactive and visually correct.

Exit criteria: App launches, tabs are clickable, user can type in the chat input and see a hardcoded echo response, clicking a mock command card copies text to clipboard.

</domain>

<decisions>
## Implementation Decisions

### CSS & Visual Design
- Use Tailwind CSS as the CSS methodology (utility-first, fast iteration)
- Use shadcn/ui as the component base (headless, composable, TypeScript-native)
- Default to dark mode theme (fits terminal/sysadmin tool persona)
- Use Tailwind preflight as the CSS reset (comes included with Tailwind)

### xterm.js Integration
- Integrate xterm.js using `useRef` + `useEffect` pattern (standard imperative lifecycle for terminal libraries)
- Place terminal preview in the bottom section of the center column (~30% height, below chat) — shows terminal context in user's line of sight
- Include `@xterm/addon-fit` for automatic resize on window/column resize events
- Show a static "No terminal connected" welcome message on initial load to confirm the widget renders

### Mock Data & State Bootstrap
- Seed mock terminal content with a realistic bash session (ls, git status output) to exercise the layout with real-looking content
- Show 2 mock tabs ("bash:1" and "bash:2") to exercise active/inactive tab states
- Mock chat response echoes the user's input text back (simple, confirms the UI flow works end-to-end)
- Pre-populate command sidebar with 3 mock commands to exercise card layout and hover tooltips

### Claude's Discretion
- Go module structure and directory layout (follow Wails v2 convention: `main.go` at root, `frontend/` subdirectory)
- Zustand store internal shape — 3 stores (chat, terminal, commands) with Immer produce-based mutations
- Zustand devtools middleware enabled in dev builds
- TypeScript strict mode enabled
- Wails window dimensions (1400×900 initial)

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- None — greenfield project, no existing components

### Established Patterns
- Research confirmed: Wails v2 uses `useRef`-stored xterm.js instances with direct `.write()` calls (bypassing React state) for performance
- Research confirmed: Wails EventsEmit/EventsOn pattern for Go→React communication; bound methods for React→Go calls
- Research confirmed: `@xterm/addon-fit` is the standard resize solution for xterm.js in contained layouts

### Integration Points
- Wails `main.go` will bind Go service structs; frontend calls via generated TypeScript bindings
- Zustand stores initialized with mock data; Phase 2/3 will replace mock data sources with real Go service calls
- xterm.js instance owned by a `TerminalPreview` component; Phase 3 will wire `runtime.EventsOn("terminal:update")` to feed real content

</code_context>

<specifics>
## Specific Ideas

- Mocks are accepted as scaffolding for Phase 1 only; they will be fully replaced by Phase 3 (tmux capture) and Phase 2 (LLM streaming)
- The app captures content from LOCAL terminal sessions (tmux panes, which can themselves contain SSH sessions to remote servers) — PairAdmin is an observer, not a direct SSH client

</specifics>

<deferred>
## Deferred Ideas

- Direct SSH client functionality (PairAdmin observes local terminal sessions including SSH; it does not initiate SSH connections itself — this distinction is by design)
- Light mode and system-preference theme switching deferred to Phase 5 (Settings dialog)
- Resizable column splitter deferred to Phase 5 (Settings & Config)

</deferred>
