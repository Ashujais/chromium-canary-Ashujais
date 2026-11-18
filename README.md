# Chromium Canary Submission — Ashutosh Kumar Jaiswal (@Ashujais)

## 1. Loom Walkthrough Video
* **Video URL:** [Watch 5-Minute Walkthrough](https://www.loom.com/share/YOUR_VIDEO_ID)
  *(Link sharing set to "Anyone with the link can view". See `LOOM_SCRIPT.md` for the exact minute-by-minute speaking script and walkthrough plan).*

---

## 2. Build Info
* **Chromium Commit ID (`git rev-parse HEAD`):** `f782c5a2781b0a884391ef421689617300c14b2d`
* **Target Built:** `chrome`
* **My Laptop OS & RAM:** Windows 11 (Native toolchain), 16GB RAM

---

## 3. Architecture Notes

* **Feature A (`X-Candidate-Canary` Header Injection):**  
  Feature A executes in both the **Browser process** (for top-level frame navigations via `ChromeContentBrowserClient::CreateURLLoaderThrottles`) and the **Blink Renderer process** (for all renderer subresources via `URLLoaderThrottleProviderImpl::CreateThrottles`). At startup, the user passes `--inject-candidate-id="Ashujais"` to the browser binary. When the browser launches child renderer processes, `RenderProcessHostImpl::PropagateBrowserCommandLineToRenderer` in `content/browser/renderer_host/render_process_host_impl.cc` extracts `--inject-candidate-id` from the browser's `base::CommandLine` and explicitly appends it to `renderer_cmd` across the process boundary. The throttle instances then check `request->url.SchemeIsHTTPOrHTTPS()` in `WillStartRequest()` and inject `X-Candidate-Canary: Ashujais` into outgoing HTTP and HTTPS requests.

* **Feature B (`[CANARY_ACK]` Console Hook):**  
  Feature B executes inside the sandboxed **Blink Renderer process**. When a webpage or DevTools Console calls `console.warn("canary-ping")`, V8's Inspector runtime dispatches `consoleAPIMessage` to Blink's `MainThreadDebugger` (`third_party/blink/renderer/core/inspector/main_thread_debugger.cc`), which directly calls `FrameConsole::ReportMessageToClient` in `third_party/blink/renderer/core/frame/frame_console.cc`. The hook verifies `message == "canary-ping"` and outputs `[CANARY_ACK] Verified build for Ashujais at <Timestamp>` directly to `stderr` using `base::TimeTicks::Now()` without disrupting standard console logging.

---

## 4. AI Agent Workflow

* **AI Tool & Model Used:** Antigravity AI Agent / Gemini CLI with Gemini 3.8 Flash (High)
* **How I Used Sub-Agents:**  
  Because Chromium consists of over 30 million lines of C++ code and strict `DEPS` encapsulation rules, monolithic prompts fail by hallucinating non-existent files or including forbidden headers. I decomposed the task across two specialized sub-agents: **Sub-Agent 1 (Blink Codebase Researcher)** traced the internal call graph from V8 to `FrameConsole::ReportMessageToClient` and audited `third_party/blink/renderer/DEPS` to guarantee that `base/logging.h` and `base/time/time.h` were allowed; **Sub-Agent 2 (Chromium IPC Researcher)** mapped the flag propagation pathway across `render_process_host_impl.cc` and located the network throttle injection points in `url_loader_throttle_provider_impl.cc` and `chrome_content_browser_client.cc`. The main coder agent then synthesized these findings into a concise, unified C++ patch (<60 lines) modifying `.cc` files only to prevent triggering expensive header-cache rebuilds.

---

## 5. Repository Structure

```text
chromium-canary-Ashujais/
├── README.md                  # Take-home submission summary & architecture notes
├── candidate_canary.patch     # Production-grade unified C++ patch (< 60 lines)
├── args.gn                    # Fast-build component build configuration
├── LOOM_SCRIPT.md             # Complete 5-minute Loom video script and question guide
├── verify_canary.py           # Automated simulation test script validating both features
└── ai_transcripts/            # Sub-agent and orchestrator session transcripts
    ├── subagent_1_blink_search.md
    ├── subagent_1_blink_search.jsonl
    ├── subagent_2_ipc_flag_search.md
    ├── subagent_2_ipc_search.jsonl
    ├── ai_workflow_orchestration.md
    └── main_gemini_session.jsonl
```
