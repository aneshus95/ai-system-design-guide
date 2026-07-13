# Computer-Use Agents

Computer-use agents let an LLM see a screen, reason about it, and act through mouse clicks and keystrokes -- the same way a human operates a computer. Instead of calling structured APIs, the model works with raw pixels. This chapter covers how they work, when they beat traditional automation, and how to design production systems around them.

## Table of Contents

- [What Are Computer-Use Agents?](#what-are-computer-use-agents)
- [The Screenshot-Reason-Act Loop](#the-screenshot-reason-act-loop)
- [Claude Computer Use: Tools and API](#claude-computer-use-tools-and-api)
- [Architecture: Sandboxed Environments](#architecture-sandboxed-environments)
- [Browser vs Desktop Automation](#browser-vs-desktop-automation)
- [Comparison with Traditional Automation](#comparison-with-traditional-automation)
- [When Computer-Use Beats API Calls](#when-computer-use-beats-api-calls)
- [Error Handling and Recovery](#error-handling-and-recovery)
- [Performance: Latency, Cost, Throughput](#performance-latency-cost-throughput)
- [Real-World Applications](#real-world-applications)
- [Security Considerations](#security-considerations)
- [Code Examples](#code-examples)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## What Are Computer-Use Agents?

A computer-use agent is an LLM that controls a graphical interface by interpreting screenshots and issuing low-level input commands (mouse moves, clicks, keystrokes). It replaces the human in the human-computer interaction loop.

```
Traditional Tool Use:           Computer Use:

User Request                    User Request
     |                               |
     v                               v
 LLM reasons                    LLM reasons
     |                               |
     v                               v
 Structured API call             Screenshot captured
 {"tool": "search",                  |
  "query": "..."}                    v
     |                          LLM sees pixels, finds button
     v                               |
 API returns JSON                    v
     |                          Mouse click at (x=340, y=220)
     v                               |
 LLM formats answer                  v
                                New screenshot captured
                                     |
                                     v
                                LLM verifies result, continues...
```

The key difference: traditional tool use requires pre-defined APIs with known schemas. Computer use works with any application that has a visual interface -- no API required.

### The Landscape (2026)

Multiple providers now offer computer-use capabilities:

| Provider | Agent | Approach | Key Strength |
|----------|-------|----------|--------------|
| Anthropic | Claude Computer Use | Vision + coordinate reasoning | Desktop + browser, mature API |
| OpenAI | ChatGPT Agent Mode | Operator-based browser agent | Deep web navigation |
| Google | Project Mariner | Gemini vision-language | Chrome integration |
| Microsoft | UFO/UFO2 | Windows UI Automation + vision | Native Windows support |
| Amazon | Nova Act | Purpose-built browser model | E-commerce workflows |

---

## The Screenshot-Reason-Act Loop

Every computer-use agent follows the same core loop, often called the "agent loop" or "action loop":

```
+------------------+
|  Capture Screen  |<-----------+
+--------+---------+            |
         |                      |
         v                      |
+------------------+            |
|  Send to LLM     |            |
|  (screenshot +   |            |
|   task context)  |            |
+--------+---------+            |
         |                      |
         v                      |
+------------------+            |
|  LLM Reasons     |            |
|  about next      |            |
|  action           |           |
+--------+---------+            |
         |                      |
    +----+----+                 |
    |         |                 |
    v         v                 |
 [Action]  [Done]               |
    |                           |
    v                           |
+------------------+            |
| Execute Action   |            |
| (click, type,    |            |
|  scroll, key)    |            |
+--------+---------+            |
         |                      |
         +----------------------+
```

Each iteration:
1. **Capture**: Take a screenshot of the current display state.
2. **Send**: Pass the screenshot (base64 image) plus the conversation history to the LLM.
3. **Reason**: The model analyzes what is on screen, determines the next step toward the goal.
4. **Act**: The model outputs a tool call (e.g., `click at (450, 320)`) which the runtime executes.
5. **Repeat**: A new screenshot is captured and the loop continues until the model signals completion.

The model maintains context across iterations via the conversation history, which accumulates screenshots and actions like a visual "memory" of what has happened.

---

## Claude Computer Use: Tools and API

Claude exposes three built-in tools for computer use. These are Anthropic-defined tools -- you do not write the implementation; Claude knows how to generate calls to them and your runtime executes them against the environment.

### The Three Tools

**1. `computer` -- Full GUI Control**

Controls mouse and keyboard on a virtual display. Capabilities:
- `screenshot` -- capture current screen state
- `left_click`, `right_click`, `double_click`, `triple_click` -- mouse clicks at coordinates
- `left_click_drag` -- drag from one point to another
- `type` -- type a string of text
- `key` -- press keyboard keys (e.g., `ctrl+c`, `Return`, `Escape`)
- `scroll` -- scroll up/down/left/right at a coordinate
- `move` -- move cursor to coordinates
- `hold_key` -- hold a modifier key while performing another action
- `wait` -- pause for a specified duration

**2. `bash` -- Shell Command Execution**

Runs shell commands in a persistent session:
- Commands share state (environment variables, working directory)
- Supports multi-line scripts
- Output is captured and returned as text

**3. `text_editor` -- File Operations**

Structured file editing with commands:
- `view` -- read file contents (with optional line range)
- `create` -- create a new file with content
- `str_replace` -- replace a specific string in a file (must be unique match)
- `insert` -- insert text at a specific line number

### API Request Structure

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=4096,
    tools=[
        {
            "type": "computer_20250124",
            "name": "computer",
            "display_width_px": 1280,
            "display_height_px": 800,
            "display_number": 1
        },
        {
            "type": "bash_20250124",
            "name": "bash"
        },
        {
            "type": "text_editor_20250124",
            "name": "str_replace_based_edit_tool"
        }
    ],
    messages=[
        {
            "role": "user",
            "content": "Open Firefox, navigate to github.com, and find repos trending today."
        }
    ],
    betas=["computer-use-2025-01-24"]
)
```

The response will contain `tool_use` blocks that your runtime must execute and feed back as `tool_result` messages.

---

## Architecture: Sandboxed Environments

Computer-use agents must run in isolated environments. The model has full control of mouse and keyboard -- you do not want that on your production workstation.

### Standard Architecture: Docker + VNC

```
+-----------------------------------------------------+
|  Docker Container                                   |
|                                                     |
|  Xvfb (Virtual X11) + Mutter (WM) + Tint2 (Panel)  |
|         |                                           |
|         v                                           |
|  +------------------+     +-------------------+     |
|  | Virtual Desktop  |---->| Screenshot Capture|     |
|  | 1280x800         |     | (scrot/maim)      |     |
|  | Firefox, apps    |     +--------+----------+     |
|  +------------------+              |                |
|                                    v                |
|                           +--------+----------+     |
|                           | Agent Runtime     |     |
|                           | - Calls Claude API|     |
|                           | - Executes actions|     |
|                           | - Manages loop    |     |
|                           +-------------------+     |
+-----------------------------------------------------+
```

### Cloud-Hosted Alternatives

Services like E2B (e2b.dev) provide pre-configured sandboxed environments:
- Ephemeral VMs with pre-installed browsers and tools
- API for screenshot capture and input injection
- Automatic cleanup after session ends
- No Docker management overhead

### Key Environment Components

| Component | Purpose | Example |
|-----------|---------|---------|
| Xvfb | Virtual X11 display server | Creates a framebuffer without physical display |
| Mutter/Xfwm | Window manager | Handles window positioning, resizing |
| Tint2 | Task panel | Shows running applications |
| xdotool | Input injection | Executes mouse/keyboard commands |
| scrot/maim | Screenshot capture | Takes display snapshots as PNG |

---

## Browser vs Desktop Automation

| Dimension | Browser-Only | Full Desktop |
|-----------|-------------|--------------|
| Scope | Web apps only | Any GUI application |
| Setup complexity | Lower (headless browser) | Higher (full desktop env) |
| Performance | Faster (smaller screenshots) | Slower (full screen captures) |
| Reliability | Higher (predictable layouts) | Lower (OS variations) |
| Use case | Web scraping, form filling | Legacy software, cross-app workflows |

Browser automation controls a web browser (navigate, fill forms, click buttons, handle SPAs). Desktop automation controls the full OS environment (launch applications, use native dialogs, interact with thick-client software, chain operations across multiple apps).

---

## Comparison with Traditional Automation

Selenium, Playwright, and Puppeteer automate browsers via direct DOM access. Computer-use agents work with pixels. Both have a place in production.

| Feature | Selenium/Playwright | Computer Use Agent |
|---------|--------------------|--------------------|
| Speed | Fast (direct DOM) | Slow (screenshot + LLM) |
| Reliability | Brittle (selector changes) | Resilient (visual recognition) |
| Maintenance | Constant selector updates | Minimal (adapts to UI changes) |
| Anti-bot detection | Frequently blocked | Harder to detect |
| Cost per action | ~$0.001 | ~$0.01-0.05 |
| Non-web support | No | Yes (any GUI) |

**Hybrid approaches** work best in production: Playwright handles high-volume, well-defined flows (login, navigation) while computer-use agents handle dynamic, unpredictable steps (visual verification, novel layouts, anti-bot sites).

---

## When Computer-Use Beats API Calls

**Use computer use when:** no API exists (legacy systems), anti-bot protections block Selenium, visual judgment is required (chart verification, PDF layout), UIs change faster than selectors can be maintained, or the workflow spans multiple desktop applications.

**Stick with APIs when:** a structured API is available (always prefer it), latency matters (sub-second), volume is high (thousands of actions/hour), or determinism is required (same input, same output).

---

## Error Handling and Recovery

Computer-use agents fail differently from API-based tools. The main failure modes:

### 1. Misclicks (Wrong Coordinates)

The model calculates coordinates from the screenshot but may miss by a few pixels:
- **Mitigation**: Use `screenshot` after every click to verify the expected state change occurred.
- **Recovery**: If the wrong element was clicked, the model can reason about the new state and correct course.

### 2. Stale Screenshots

The screen may have changed between capture and action execution (animations, popups, loading):
- **Mitigation**: Add a short wait before screenshots. Use the `wait` action when pages are loading.
- **Recovery**: Re-capture and re-assess before continuing.

### 3. Infinite Loops

The model repeats the same action without making progress:
- **Mitigation**: Set a maximum iteration count (e.g., 50 actions per task).
- **Recovery**: After N repeated identical actions, force a different approach or escalate to a human.

### 4. Unexpected Dialogs

Cookie banners, popups, permission dialogs appear unexpectedly:
- **Mitigation**: Include instructions in the system prompt about handling common dialogs.
- **Recovery**: The model's visual reasoning usually handles these naturally -- it sees the dialog and dismisses it.

### 5. Resolution and Scaling Mismatches

The model was trained at specific resolutions. Mismatches cause coordinate errors:
- **Mitigation**: Use recommended resolution (1280x800) with display scaling at 100%.
- **Recovery**: Adjust `display_width_px` and `display_height_px` to match actual display.

### Error Handling Pattern

The agent loop should track action history and detect repeats. If the same action is emitted 3+ times consecutively, inject a message telling the model to try a different approach. Always set a hard maximum iteration count (e.g., 50) and capture a verification screenshot after every action to detect state changes. See the full agent loop in the Code Examples section below.

---

## Performance: Latency, Cost, Throughput

### Latency Breakdown

Each iteration of the agent loop involves:

```
Screenshot capture:     ~100ms
Image encoding (base64): ~50ms
API call (with image):   ~2-5s  (model inference)
Action execution:        ~100ms
                        --------
Total per action:        ~2.5-5.5s
```

A typical 10-step task takes 25-55 seconds. Compare this to Playwright which completes the same 10 steps in under 2 seconds.

### Cost Per Action

Each action sends a screenshot (~800KB base64) plus conversation history:

| Model | Cost per Action (approx) | Notes |
|-------|-------------------------|-------|
| Claude Sonnet 4 | $0.01-0.03 | Recommended for most tasks |
| Claude Opus 4 | $0.05-0.15 | Use for complex visual reasoning |

A 20-step workflow costs roughly $0.20-0.60 with Sonnet, or $1.00-3.00 with Opus.

### Throughput Optimization

- **Parallel sessions**: Run multiple Docker containers for concurrent tasks.
- **Selective screenshots**: Only capture after uncertain actions; skip after typing text.
- **Resolution reduction**: Use 1024x768 instead of 1920x1080 to reduce token cost.
- **Early termination**: Teach the model to signal completion as soon as the goal is verified.

---

## Real-World Applications

| Application | How It Works | Why Computer Use |
|------------|--------------|------------------|
| Legacy system integration | Agent navigates mainframe/thick-client UI, extracts data to structured format | No API exists for legacy software |
| Form filling / data entry | Reads source documents, fills web forms field by field, handles multi-page wizards | Government portals, insurance claims with complex conditional logic |
| QA and visual testing | Navigates app as a user, verifies visual rendering, reports issues in natural language | Goes beyond pixel-diff -- understands layout and UX |
| Competitive intelligence | Navigates product pages, captures pricing data from JS-rendered widgets | Works on sites that block traditional scrapers |

---

## Security Considerations

| Risk | What Happens | Mitigation |
|------|-------------|------------|
| **Visible secrets** | Model sees passwords, sessions, notifications in screenshots | Ephemeral containers, clear credentials after use |
| **Unrestricted actions** | Agent can run shell commands, navigate anywhere, download files | Firewall rules, read-only FS, session time limits, HITL for destructive ops |
| **Data exfiltration** | Screenshots sent to LLM provider contain sensitive data | On-premise deployment for regulated industries, mask sensitive UI fields |
| **Prompt injection via UI** | Malicious site displays text to manipulate the agent | System prompt warnings against following on-screen instructions that contradict the task |

The cardinal rule: **never run computer-use agents on your production workstation or with access to real credentials unless in a fully sandboxed container**.

---

## Code Examples

### Minimal Agent Loop

```python
import anthropic, base64, subprocess

client = anthropic.Anthropic()

def capture_screenshot():
    subprocess.run(["scrot", "/tmp/screen.png", "-o"], check=True)
    with open("/tmp/screen.png", "rb") as f:
        return base64.standard_b64encode(f.read()).decode()

def execute_action(action):
    name = action["action"]
    if name == "left_click":
        x, y = action["coordinate"]
        subprocess.run(["xdotool", "mousemove", str(x), str(y), "click", "1"])
    elif name == "type":
        subprocess.run(["xdotool", "type", "--", action["text"]])
    elif name == "key":
        subprocess.run(["xdotool", "key", action["text"]])

def run_agent(task: str, max_steps: int = 30):
    messages = [{"role": "user", "content": task}]
    tools = [
        {"type": "computer_20250124", "name": "computer",
         "display_width_px": 1280, "display_height_px": 800},
        {"type": "bash_20250124", "name": "bash"},
        {"type": "text_editor_20250124", "name": "str_replace_based_edit_tool"},
    ]
    for step in range(max_steps):
        response = client.messages.create(
            model="claude-sonnet-4-20250514", max_tokens=4096,
            tools=tools, messages=messages, betas=["computer-use-2025-01-24"],
        )
        if response.stop_reason == "end_turn":
            return [b.text for b in response.content if b.type == "text"]

        tool_results = []
        for block in response.content:
            if block.type != "tool_use":
                continue
            if block.name == "computer":
                execute_action(block.input)
                tool_results.append({
                    "type": "tool_result", "tool_use_id": block.id,
                    "content": [{"type": "image", "source": {
                        "type": "base64", "media_type": "image/png",
                        "data": capture_screenshot()}}],
                })
            elif block.name == "bash":
                r = subprocess.run(block.input["command"],
                    shell=True, capture_output=True, text=True)
                tool_results.append({
                    "type": "tool_result", "tool_use_id": block.id,
                    "content": r.stdout + r.stderr,
                })
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})
    return ["Max steps reached"]
```

### Dockerfile for Sandboxed Environment

```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y \
    xvfb mutter tint2 xdotool scrot firefox-esr python3 python3-pip \
    && rm -rf /var/lib/apt/lists/*
RUN pip3 install anthropic
ENV DISPLAY=:1
COPY agent.py /agent.py
CMD Xvfb :1 -screen 0 1280x800x24 & sleep 1 && mutter & tint2 & \
    sleep 1 && python3 /agent.py
```

---

## Interview Questions

### Q: A client has 500 insurance claim PDFs per day that must be entered into a legacy web portal with no API. Design a system using computer-use agents.

**Strong answer:**
I would build a pipeline with three stages. First, a document processing stage using an LLM to extract structured data from the PDFs (claim number, claimant name, amounts, dates). Second, a computer-use agent stage where each claim is processed by a Claude Computer Use agent running in an isolated Docker container with a virtual display. The agent navigates the web portal, fills in the form fields using the extracted data, and captures a confirmation screenshot after submission. Third, a verification stage that uses a separate LLM call to compare the confirmation screenshot against the expected data to catch any entry errors.

For scale, I would run 10-20 containers in parallel, each processing claims sequentially. At roughly 2 minutes per claim with the agent, 20 containers handle 600 claims per 8-hour day. I would add a dead-letter queue for claims that fail after 3 retries, with human review.

Cost at $0.50 per claim (roughly 20 actions at $0.025 each) means $250/day for 500 claims -- likely cheaper than the manual data entry team it replaces.

### Q: Compare computer-use agents with Selenium for web automation. When would you choose each?

**Strong answer:**
Selenium interacts with the DOM directly -- it is fast, deterministic, and cheap. But it breaks when selectors change, gets blocked by anti-bot systems, and cannot handle tasks requiring visual judgment.

Computer-use agents are 100x slower and 10x more expensive per action, but they adapt to UI changes because they work with pixels rather than selectors. They handle anti-bot detection better because they generate human-like interaction patterns. And they can reason about visual layouts -- verifying a chart rendered correctly or reading content from a canvas element that Selenium cannot inspect.

I would choose Selenium for high-volume, stable workflows where the target site is under my control. I would choose computer-use agents for one-off tasks, third-party sites that change frequently, cross-application desktop workflows, and any task where the human cost of maintaining selectors exceeds the LLM inference cost.

The best production systems use both: Playwright handles the predictable steps (authentication, navigation), and the computer-use agent handles the dynamic steps (interpreting results, making judgment calls).

---

## References

- Anthropic. "Computer Use Tool" API Documentation (2025)
- Anthropic. "Bash Tool" and "Text Editor Tool" API Documentation (2025)
- E2B. "Sandboxed Cloud Environments for AI Agents" (2025)
- OSWorld Benchmark: Desktop Agent Evaluation Suite (2025)
- WebArena Benchmark: Web Agent Evaluation Suite (2024)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Computer-Use Agent** | An LLM that controls a graphical interface by interpreting screenshots and issuing mouse and keyboard commands | Enables AI to automate any application with a visual interface, even with no API |
| **Screenshot-Reason-Act Loop** | The repeating cycle of capturing a screenshot, sending it to the LLM, and executing the resulting action | The core operating mechanism all computer-use agents share |
| **Vision-Action Loop** | Same as the screenshot-reason-act loop; the name emphasizes that the model's only input is visual pixels | Highlights the key difference from API-based automation, which uses structured data instead |
| **`computer` tool** | The Anthropic-defined tool that gives Claude the ability to take screenshots and move the mouse and keyboard | The primary interface through which Claude interacts with the virtual desktop |
| **`bash` tool** | The Anthropic-defined tool that runs shell commands in a persistent session | Lets Claude complement GUI interactions with direct command-line execution |
| **`text_editor` tool** | The Anthropic-defined tool for structured file operations including view, create, str_replace, and insert | Provides a reliable, precise way to read and modify files without relying on GUI file dialogs |
| **base64 PNG** | A way of encoding a screenshot image as a text string so it can be embedded in an API request | How screenshots are transmitted to the Claude API; each one adds significant token cost |
| **Xvfb** | A virtual X11 display server that creates a framebuffer without requiring a physical monitor | The foundation of the sandboxed desktop environment for computer-use agents |
| **Mutter / Xfwm** | Linux window managers that handle window positioning and resizing inside the virtual display | Required for GUI applications to render and behave correctly in the virtual environment |
| **xdotool** | A command-line tool for injecting mouse movements, clicks, and keystrokes into an X11 display | The mechanism by which the agent runtime physically executes the LLM's action commands |
| **scrot / maim** | Command-line screenshot capture utilities for X11 | Used by the agent runtime to take the screenshots that become the LLM's visual input |
| **Tint2** | A lightweight task panel that shows running applications inside the virtual desktop | Helps the agent visually identify open windows and switch between applications |
| **Docker + VNC** | The standard architecture for sandboxed computer-use: a Docker container running a virtual desktop accessible via VNC | Provides safe isolation so the agent cannot affect the operator's real machine |
| **E2B** | A cloud service offering pre-configured sandboxed virtual environments with screenshot capture and input injection | Removes the need to build and maintain Docker-based desktop environments for agent testing |
| **OSWorld benchmark** | A benchmark suite that measures how accurately a computer-use agent completes real desktop tasks | The standard measure of computer-use agent capability; Claude improved from 14.9% to 72.5% between 2024 and 2026 |
| **WebArena benchmark** | A benchmark suite that measures how accurately a web agent completes realistic web browsing tasks | Complements OSWorld by focusing specifically on browser-based workflows |
| **Selenium / Playwright** | Browser automation frameworks that interact directly with the DOM rather than with screenshots | Faster and cheaper than computer-use agents, but brittle when selectors change or on sites with anti-bot measures |
| **DOM (Document Object Model)** | The tree structure that represents all elements of a web page in memory | What Selenium and Playwright manipulate directly; computer-use agents see only the rendered pixels, not the DOM |
| **Headless browser** | A web browser that runs without a visible window, controlled entirely via code | A lighter-weight alternative to a full virtual desktop when the agent only needs to automate web pages |
| **Anti-bot detection** | Techniques websites use to identify and block automated browser scripts | A major limitation of Selenium; computer-use agents mimic human interaction patterns and are harder to block |
| **Zoom Action** | A 2026 enhancement that captures a high-resolution crop of a small UI region before acting | Reduces misclicks on dense interfaces where standard-resolution screenshots do not show enough detail |
| **Stale screenshot** | A screenshot captured before a page animation or loading event has finished | A common failure mode; mitigated by adding a short wait before captures and re-taking screenshots when in doubt |
| **Infinite loop (agent)** | When the agent repeats the same action without making progress toward the goal | Prevented by setting a maximum iteration count and detecting repeated identical actions |
| **Prompt injection via UI** | An attack where a malicious website displays text designed to override the agent's instructions | A security risk unique to computer-use agents; mitigated by system-prompt warnings and sandboxed environments |
| **HITL (Human-in-the-Loop)** | A design pattern that requires a human to approve the agent's action before it is executed | Required before any destructive operation such as form submission, file deletion, or financial transaction |
| **Parallel sessions** | Running multiple Docker containers simultaneously, each processing a different task | The primary way to increase computer-use agent throughput since each action is inherently sequential |
| **Resolution / display scaling** | The pixel dimensions and zoom level of the virtual display | Must match the values declared in the API request; mismatches cause coordinate errors and misclicks |

*Next: [Building Tool-Use Agents](05-building-tool-agents.md)*
