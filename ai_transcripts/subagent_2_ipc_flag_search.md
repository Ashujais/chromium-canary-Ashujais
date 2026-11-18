# Sub-Agent 2 (Chromium IPC & URLLoaderThrottle Search) Log & Findings

**Agent Role:** Chromium IPC Researcher  
**Subagent Conversation ID:** `202f56ae-72fe-468d-98f6-88db8b8b3a6d`  
**Focus:** Tracing `base::CommandLine` flag propagation and network request header injection via `URLLoaderThrottle`

---

## Task Instructions
1. Trace how `base::CommandLine` flags get copied from the Browser process to child processes (specifically `content/browser/renderer_host/render_process_host_impl.cc`).
2. Analyze how `--inject-candidate-id=<string>` can be passed to child renderer processes without requiring modifications to shared `.h` header files like `content_switches.h`.
3. Locate where `URLLoaderThrottle` runs in the network pipeline (both in renderer process via `chrome/renderer/url_loader_throttle_provider_impl.cc` and browser process via `chrome/browser/chrome_content_browser_client.cc`).
4. Detail how a custom `URLLoaderThrottle` can inspect `request->url.SchemeIsHTTPOrHTTPS()` and inject the `X-Candidate-Canary: <value>` HTTP header on all outgoing HTTP and HTTPS requests when `--inject-candidate-id` is present, while doing nothing when the flag is omitted.
5. Provide exact file paths, line references, and C++ code implementation (< 60 lines).

---

## Research Trace & Findings

### 1. Process Boundary & Flag Copying
Chromium isolates rendering in untrusted child processes. The Browser process does NOT automatically copy all startup command-line switches to child processes.
In `content/browser/renderer_host/render_process_host_impl.cc`:
* `RenderProcessHostImpl::Init()` constructs child command line arguments.
* `RenderProcessHostImpl::AppendRendererCommandLine()` triggers `PropagateBrowserCommandLineToRenderer(browser_cmd, renderer_cmd)`.
* By adding:
  ```cpp
  if (browser_cmd.HasSwitch("inject-candidate-id")) {
    renderer_cmd->AppendSwitchASCII(
        "inject-candidate-id",
        browser_cmd.GetSwitchValueASCII("inject-candidate-id"));
  }
  ```
  around line 4122 in `render_process_host_impl.cc`, the switch is safely forwarded across the IPC boundary to child renderers without editing any shared header files (`content_switches.h`).

### 2. Header Rebuild Avoidance
Editing `content_switches.h` triggers a recompile of 5,000+ files. Using string literals (`"inject-candidate-id"`) directly inside `.cc` files ensures that only the 3 modified `.cc` files recompile, completing in 15–30 seconds under component build.

### 3. URLLoaderThrottle Network Interception
Network requests originate from two places:
1. **Renderer Process (subresources, fetch, XHR, script, CSS):**  
   Handled by `URLLoaderThrottleProviderImpl::CreateThrottles` in `chrome/renderer/url_loader_throttle_provider_impl.cc`.
2. **Browser Process (top-level Omnibox navigations to https://httpbin.org/headers):**  
   Handled by `ChromeContentBrowserClient::CreateURLLoaderThrottles` in `chrome/browser/chrome_content_browser_client.cc`.

### 4. Custom Throttle Class
```cpp
class CanaryURLLoaderThrottle : public blink::URLLoaderThrottle {
 public:
  explicit CanaryURLLoaderThrottle(std::string candidate_id)
      : candidate_id_(std::move(candidate_id)) {}
  ~CanaryURLLoaderThrottle() override = default;

  void WillStartRequest(network::ResourceRequest* request,
                        bool* defer) override {
    if (request->url.SchemeIsHTTPOrHTTPS()) {
      request->headers.SetHeader("X-Candidate-Canary", candidate_id_);
    }
  }

 private:
  std::string candidate_id_;
};
```
When registered in `CreateThrottles`, it checks:
```cpp
const base::CommandLine* cmd = base::CommandLine::ForCurrentProcess();
if (cmd->HasSwitch("inject-candidate-id")) {
  std::string cid = cmd->GetSwitchValueASCII("inject-candidate-id");
  if (!cid.empty()) {
    throttles.push_back(
        std::make_unique<CanaryURLLoaderThrottle>(std::move(cid)));
  }
}
```
If the browser is started without `--inject-candidate-id`, `HasSwitch()` returns `false`, no throttle is added, and no headers are modified.
