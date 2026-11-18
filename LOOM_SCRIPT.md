# 5-Minute Loom Video Walkthrough Script

**Candidate:** Ashutosh Kumar Jaiswal (@Ashujais)  
**Target Video Duration:** Under 5 minutes (Strictly Unedited, Single Take)  
**Requirements:** Screen share active, microphone on, no cuts/pauses.

---

## Part 1: Prove Your Build Works (`0:00 – 1:30`)

### Action:
1. Open terminal inside Chromium's `src/` directory.
2. Run:
   ```bash
   git status
   git diff --stat
   ```
3. Open `third_party/blink/renderer/core/frame/frame_console.cc` in your IDE/editor.
4. Live edit: change `[CANARY_ACK]` to `[CANARY_ACK_LIVE]`.
5. Save the file and trigger the component build:
   ```bash
   autoninja -C out/Default chrome
   ```
   *(With `is_component_build = true` and `symbol_level = 0`, Ninja compiles only `frame_console.cc` and links in ~20 seconds).*

### What to Say (Speaking Script):
> "Hi everyone, my name is Ashutosh Kumar Jaiswal (@Ashujais on GitHub), and this is my walkthrough for the Custom Chromium Build and Telemetry Canary internship take-home assignment.  
> Right now, I'm inside the Chromium `src/` directory. Running `git status` and `git diff --stat` shows our clean telemetry patch modifying only four C++ files.  
> To demonstrate that our fast-build setup works, I'll open `frame_console.cc` where our Canary hook lives. I'm updating `[CANARY_ACK]` to `[CANARY_ACK_LIVE]`, saving the file, and running `autoninja -C out/Default chrome`.  
> Because we configured `args.gn` with `is_component_build = true` and avoided modifying any shared `.h` header files, Ninja only rebuilds this single object file and relinks in under 20 seconds. And as you can see, the build succeeds cleanly."

---

## Part 2: Show Your Features Working (`1:30 – 3:00`)

### Action:
1. Launch the custom browser with logging enabled to `stderr`:
   ```bash
   ./out/Default/chrome --inject-candidate-id="Ashujais" --enable-logging=stderr
   ```
2. Navigate to `https://httpbin.org/headers`.
3. Open Chrome DevTools (`F12` or `Ctrl + Shift + I`):
   - Switch to the **Network** tab.
   - Click on the `headers` document request.
   - Expand **Request Headers** and point out:
     ```http
     X-Candidate-Canary: Ashujais
     ```
4. Switch to the DevTools **Console** tab:
   - Type and execute:
     ```javascript
     console.warn("canary-ping")
     ```
   - Show your terminal (`stderr`), highlighting the output:
     ```text
     [CANARY_ACK_LIVE] Verified build for Ashujais at 123456789
     ```
   - Type a normal warning to demonstrate regression safety:
     ```javascript
     console.warn("ordinary test message")
     ```
   - Show that normal warnings appear in DevTools without emitting `[CANARY_ACK_LIVE]` in the terminal.

### What to Say (Speaking Script):
> "Now, let's test both features in action.  
> I am launching Chrome from the command line with our custom flag `--inject-candidate-id="Ashujais"` and `--enable-logging=stderr`.  
> First, let's verify Feature A: Network Header Injection. I navigate to `https://httpbin.org/headers`. Opening the DevTools Network panel and inspecting the outgoing request headers, we can clearly see `X-Candidate-Canary: Ashujais` sent with the HTTP request.  
> Next, let's verify Feature B: the Blink Console Canary Hook. In the DevTools Console, I type `console.warn("canary-ping")` and press Enter. Over in our terminal, we immediately observe the log line: `[CANARY_ACK_LIVE] Verified build for Ashujais at <Timestamp>`, retrieved at runtime via `base::TimeTicks::Now()`.  
> If I execute `console.warn("regular warning")`, normal console logging continues without triggering our canary acknowledgement, preserving standard browser behavior."

---

## Part 3: Architecture Deep-Dive (`3:00 – 5:00`)

### Action:
Open your `git diff` in your editor or terminal and scroll through each change while answering the two required architecture questions out loud.

### What to Say (Speaking Script):

> "Now, let's review the code in `git diff` and answer the two architectural questions.

#### Question 1: Which OS process runs Feature A, and which runs Feature B?
> "Chromium uses a multi-process architecture to isolate untrusted web code from system resources:  
> * **Feature B (Console Canary Hook):** Runs strictly inside the **Blink Renderer Process**. When a webpage or DevTools console executes JavaScript, V8 evaluates it on the Renderer main thread. V8's inspector forwards the call to Blink's `MainThreadDebugger::consoleAPIMessage`, which invokes `FrameConsole::ReportMessageToClient` inside `frame_console.cc` in the Renderer process.  
> * **Feature A (Network Header Injection):** Runs across both the **Browser Process** and the **Blink Renderer Process**. Top-level frame navigations (e.g. typing a URL in the Omnibox) are initiated by the Browser process and intercepted by `ChromeContentBrowserClient::CreateURLLoaderThrottles`. In-page subresources (scripts, images, stylesheets, and `fetch()` calls) originate inside the Renderer process and are intercepted by `URLLoaderThrottleProviderImpl::CreateThrottles`. Both use our `blink::URLLoaderThrottle` hook before requests are dispatched to the Network Service.

#### Question 2: How did the `--inject-candidate-id` flag value get from the main Browser process at startup to the process that adds the HTTP header?
> "When Chrome starts, only the main Browser process initially sees the command-line arguments passed to the binary. Chromium sandboxes child Renderer processes and does NOT automatically propagate all command-line switches to them.  
> As shown here in our diff for `content/browser/renderer_host/render_process_host_impl.cc`, we modified `RenderProcessHostImpl::PropagateBrowserCommandLineToRenderer`. When the browser prepares to spawn a new renderer process, it extracts the switch value from its own `base::CommandLine::ForCurrentProcess()` and explicitly appends `--inject-candidate-id` to the child's `renderer_cmd`.  
> When the child renderer process initializes, its own local `base::CommandLine` receives this flag. In `url_loader_throttle_provider_impl.cc`, the renderer reads the flag, instantiates our `CanaryURLLoaderThrottle`, and attaches the `X-Candidate-Canary` header in `WillStartRequest()` whenever the scheme is HTTP or HTTPS. If the flag is omitted at startup, `HasSwitch()` evaluates to false and no header is added.

#### Wrap-up:
> "To prevent massive rebuilds, all modifications were restricted to `.cc` files, avoiding any edits to shared headers like `content_switches.h`. Thank you for reviewing my submission!"
