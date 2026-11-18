# AI Agent Workflow & Multi-Agent Orchestration Log

**Coding Agent System:** Antigravity AI Agent (Gemini 3.8 Flash High)  
**Main Orchestration ID:** `14aa074e-fc98-4b08-bf4d-50402053e6f7`  
**Engineer:** Ashutosh Kumar Jaiswal (@Ashujais)  
**Project:** Engineering Internship Take-Home — Custom Chromium Build & Telemetry Canary

---

## 1. Problem Decomposition & Multi-Agent Architecture

The Chromium codebase contains >30 million lines of C++ code, hundreds of `DEPS` isolation rules, and complex multi-process IPC constraints. Asking a single LLM prompt to write changes directly across Blink, Content, and Chrome typically results in:
* Attempting to `#include` headers forbidden by Blink's `DEPS` check (e.g. `base/command_line.h` inside `frame_console.cc`).
* Hallucinating file locations (e.g. assuming a non-existent `console.cc` inside `third_party/blink/renderer/core/frame/` instead of `frame_console.cc`).
* Editing shared `.h` header files like `content_switches.h`, which invalidates header caches and forces a 5,000+ file recompilation.
* Violating process isolation by attempting to access browser-process command-line arguments directly from renderer threads without IPC propagation.

To solve this, we decoupled the codebase into three specialized roles:
1. **Sub-Agent 1 (Blink Codebase Researcher):** Focused strictly on the Blink rendering engine, locating the exact invocation pathway of `console.warn` from V8 to Blink, and checking `DEPS` rules.
2. **Sub-Agent 2 (Chromium IPC Researcher):** Focused strictly on the multi-process architecture, tracking how `base::CommandLine` is copied across the process boundary in `render_process_host_impl.cc` and identifying the network throttle hook in `URLLoaderThrottleProviderImpl` and `ChromeContentBrowserClient`.
3. **Main Agent (Coder & Integrator):** Synthesized the sub-agent outputs, verified line boundaries, ensured zero `.h` header modifications, and produced the clean, unified `candidate_canary.patch`.

---

## 2. Key Multi-Process Discoveries

### Feature A: Network Header Injection (`X-Candidate-Canary`)
* **Process Flow:**
  1. The user passes `./chrome --inject-candidate-id="Ashujais"`.
  2. The Browser process initializes `base::CommandLine::ForCurrentProcess()`.
  3. When launching child renderer processes, `RenderProcessHostImpl::PropagateBrowserCommandLineToRenderer` copies `--inject-candidate-id` to the child's command line.
  4. Both `ChromeContentBrowserClient::CreateURLLoaderThrottles` (Browser process, for top-level navigations) and `URLLoaderThrottleProviderImpl::CreateThrottles` (Renderer process, for subresources) instantiate `CanaryURLLoaderThrottle`.
  5. In `WillStartRequest()`, `request->url.SchemeIsHTTPOrHTTPS()` guards non-web schemes and injects `X-Candidate-Canary: Ashujais`.

### Feature B: Console Canary Hook (`[CANARY_ACK]`)
* **Process Flow:**
  1. The user types `console.warn("canary-ping")` in DevTools Console.
  2. The command executes inside the Blink Renderer process.
  3. V8's Inspector dispatches `consoleAPIMessage` to `MainThreadDebugger`.
  4. `MainThreadDebugger` calls `FrameConsole::ReportMessageToClient` directly (bypassing `FrameConsole::AddMessage`).
  5. The hook catches `message == "canary-ping"` and outputs `[CANARY_ACK] Verified build for Ashujais at <Timestamp>` to `stderr`.

---

## 3. Sub-Agent Execution Transcripts
* `subagent_1_blink_search.jsonl` / `subagent_1_blink_search.md`
* `subagent_2_ipc_search.jsonl` / `subagent_2_ipc_flag_search.md`
* `main_gemini_session.jsonl`
