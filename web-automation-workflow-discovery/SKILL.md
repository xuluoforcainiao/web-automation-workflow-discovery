---
name: web-automation-workflow-discovery
description: Learn and automate human workflows on websites through browser exploration, screenshot analysis, and iterative script refinement. Use when the user needs to automate repetitive tasks on web applications, replicate human click-type-upload sequences, or build unattended browser automation for any website.
name_zh: 学习如何自动化操作网站
---

# Web Automation Workflow Discovery

Learn how a human performs a task on a website, then build a robust automation script for it.

## Workflow Overview

```
Discovery -> Exploration -> Element Mapping -> Script Build -> Test & Refine -> Package
```

## Phase 1: Discovery

Gather the essential context before touching any code:

1. **Target website URL** — The entry point
2. **Human workflow** — Step-by-step what a human does (click, type, upload, etc.)
3. **Input data** — What files/data does the process consume?
4. **Success criteria** — How do you know the task completed successfully?
5. **Frequency** — One-time, daily, or triggered by events?
6. **Security constraints** — Should credentials be saved? Can others use this?

If the user has not provided these, ask before proceeding.

## Phase 2: Exploration

Use Playwright to explore the website automatically. Do NOT rely on the user to manually click through for you.

### 2.1 Open and screenshot

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False, slow_mo=100)
    page = browser.new_page(viewport={"width": 1600, "height": 1000})
    page.goto("https://target-site.com/path")
    page.screenshot(path="screenshot_01_entry.png", full_page=True)
    with open("page_01_entry.html", "w", encoding="utf-8") as f:
        f.write(page.content())
```

### 2.2 Auto-navigate through the workflow

Write a small exploration script that clicks likely buttons and saves a screenshot at each step:

```python
def explore_step(page, action_name):
    """Execute next action and save state."""
    page.screenshot(path=f"explore_{action_name}.png", full_page=True)
    with open(f"explore_{action_name}.html", "w", encoding="utf-8") as f:
        f.write(page.content())
```

Use keyword-based selectors to find elements (text content, placeholder, etc.) rather than fragile CSS paths.

### 2.3 Analyze screenshots and HTML

For each saved screenshot and HTML file:
- Identify the page type: login, dashboard, form, modal, grid/list, upload
- Note any loading states, overlays, or AJAX patterns
- Record the exact text of buttons/links you need to interact with

**Key page types to recognize:**

| Page Type | Characteristics | Handling Strategy |
|-----------|-----------------|-------------------|
| SPA (Single Page App) | URL doesn't change, content replaced dynamically | Use `wait_until="domcontentloaded"`, detect AJAX completion |
| Login Modal | `#modal-login`, form inside overlay | Fill inputs inside modal, wait for modal to disappear |
| Grid/Table (yiiGridView, DataTables) | `table`, pagination, filter inputs | Use Enter key for filtering, wait for `.grid-view-loading` to disappear |
| File Upload (plupload, Dropzone) | Hidden `input[type="file"]`, progress bars | `set_input_files()` on hidden input, look for `<a class="plupload_start">` not `<button>` |
| Popup Window | `window.open()` | Use `page.expect_popup()` |

## Phase 3: Element Mapping

Build a selector reference table. Test each selector with `page.locator(sel).count()` to confirm it resolves to exactly the expected number of elements.

```
Element                | Selector                                    | Count Expected
-----------------------|---------------------------------------------|---------------
Username input         | input[name="LoginForm[user]"]               | 1
Password input         | input[name="LoginForm[pwd]"]                | 1
Login button           | button:has-text("Login")                    | 1
Held Shipments menu    | a:has-text("Held Shipments")                | 1
Tracking filter        | #task_grid_view input#ImParcel_ref          | 1
View link              | a.tracking-modal-link                       | >=0
Upload Document link   | a:has-text("Upload Document")               | >=0
File input (plupload)  | input[type="file"]                          | 1
Start Upload button    | a.plupload_start:not(.plupload_disabled)    | 1
```

**Selector priority (most to least reliable):**
1. `id` or stable `data-testid`
2. `input[name="exact_name"]`
3. Text-based: `button:has-text("Exact Text")`, `a:has-text("Partial")`
4. Scoped selector: `#parent_id input#child_id` (prevents matching wrong elements)
5. CSS class (only if unique and semantic): `.tracking-modal-link`
6. XPath (last resort)

## Phase 4: Script Build

Build the automation script incrementally. Do not write the entire script at once.

### 4.1 Structure the script

```python
def main():
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=False, slow_mo=120)
        page = browser.new_page(viewport={"width": 1600, "height": 1000})

        # Step 1: Navigate to entry page
        # Step 2: Login (if needed)
        # Step 3: Navigate to target page
        # Step 4: Loop through input data
        # Step 5: For each item: search -> open -> interact -> verify
        # Step 6: Completion dialog
        browser.close()
```

### 4.2 SPA navigation — prefer direct `goto` over clicks

For Single Page Applications, clicking dashboard menu items is unreliable because AJAX timing is unpredictable. Instead:

```python
# Good: direct navigation
page.goto("https://site.com/shipment/heldList.app",
          timeout=120000, wait_until="domcontentloaded")

# Avoid: clicking SPA menu buttons
# page.click("a:has-text('Held Shipments')")  # Unreliable
```

### 4.3 Wait strategies

```python
# For AJAX grid updates
def wait_for_grid_update(page, timeout=30):
    start = time.time()
    while time.time() - start < timeout:
        loading = page.locator("#loading-container").is_visible()
        grid_loading = page.locator(".grid-view-loading").count() > 0
        if not loading and not grid_loading:
            time.sleep(0.8)
            return True
        time.sleep(0.3)
    return False

# For modal content loading
modal.wait_for(state="visible", timeout=15000)
```

### 4.4 Login detection with dual criteria

SPA logins often replace content without redirecting. Use multiple signals:

```python
logged_in = False
while time.time() - start < 600:
    try:
        login_form_gone = page.locator('input[name="LoginForm[user]"]').count() == 0
        has_welcome = page.locator("text=Welcome").count() > 0
        if login_form_gone or has_welcome:
            logged_in = True
            break
    except Exception:
        time.sleep(0.5)
        continue
    time.sleep(1)
```

### 4.5 Exception handling in wait loops

**Critical**: In login wait loops, use `continue` on exception, never `break`. A page refresh during the wait is normal.

```python
# WRONG — exits early on any error
except:
    break

# CORRECT — retries on transient errors
except Exception as e:
    log(f"Retry after error: {e}")
    time.sleep(0.5)
    continue
```

### 4.6 Duplicate detection

Before uploading or creating records, check if they already exist:

```python
uploaded_rows = upload_page.locator("#grid-id table tbody tr").all()
for row in uploaded_rows:
    if filename in row.inner_text():
        log(f"Already exists: {filename}, skipping")
        return
```

## Phase 5: Test & Refine

### 5.1 Run and observe

Run the script. If it fails:
1. Check the last saved screenshot — what did the browser actually see?
2. Check the last saved HTML — what elements were present?
3. Read the error traceback — is it a selector mismatch, timeout, or unexpected page state?

### 5.2 Fix common failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Timeout 60000ms exceeded` on page load | Slow SPA, large assets | `wait_until="domcontentloaded"`, `timeout=120000` |
| Selector resolves to N elements (N>1) | Ambiguous selector | Add parent scope: `#parent input#id` |
| `Executable doesn't exist` (exe only) | Chromium not bundled | Bundle with `--add-data "path/to/chromium;chromium"` or detect system install |
| Button not found | Wrong tag type | plupload uses `<a>`, not `<button>` |
| No popup detected | Popup blocked or same-tab | Fallback: `upload_page = page` |
| File not actually uploaded | Hidden input not wired | Use `set_input_files()` on `input[type="file"]` inside the plupload container |

### 5.3 Add visual feedback for the user

When running as an exe, the user may not see console output. Inject a completion dialog into the browser:

```javascript
// Injected via page.evaluate()
const div = document.createElement('div');
div.innerHTML = `
  <div style="position:fixed;top:0;left:0;width:100%;height:100%;
      background:rgba(0,0,0,0.75);z-index:999999;
      display:flex;align-items:center;justify-content:center;">
      <div style="background:#fff;padding:40px;border-radius:14px;text-align:center;">
          <h2>All tasks completed!</h2>
          <p>Closing in <span id="countdown">30</span>s</p>
          <button id="close-btn">Close Now</button>
      </div>
  </div>
`;
document.body.appendChild(div);
```

Then wait for the dialog to disappear:
```python
page.wait_for_selector("#dialog-id", state="detached", timeout=30000)
```

## Phase 6: Package

### 6.1 PyInstaller resource path

Make the script work both as `.py` and as PyInstaller `.exe`:

```python
def resource_path(relative_path):
    if hasattr(sys, '_MEIPASS'):
        return os.path.join(sys._MEIPASS, relative_path)
    return os.path.join(os.path.abspath("."), relative_path)
```

### 6.2 Bundle Chromium for standalone distribution

```bash
py -m PyInstaller --onefile \
  --add-data "config.html;." \
  --add-data "C:\Users\...\ms-playwright\chromium-XXXX;chromium" \
  --name "MyAutomation" script.py
```

In the script, check bundled Chromium first:
```python
if hasattr(sys, '_MEIPASS'):
    bundled = os.path.join(sys._MEIPASS, "chromium", "chrome-win", "chrome.exe")
    if os.path.exists(bundled):
        return bundled
```

### 6.3 Desktop logging

Always write logs to a file on the Desktop so users can diagnose issues even if the console closes:

```python
LOG_FILE = os.path.join(os.path.expanduser("~"), "Desktop", "automation_log.txt")

def log(msg):
    line = f"[{time.strftime('%H:%M:%S')}] {msg}"
    print(line)
    try:
        with open(LOG_FILE, "a", encoding="utf-8") as f:
            f.write(line + "\n")
    except Exception:
        pass
```

## Security Checklist

Before delivering to others:
- [ ] Credentials are NOT hardcoded
- [ ] Credentials are NOT saved to disk (only configuration like paths/URLs)
- [ ] If credentials must be entered each run, provide a local HTML config page
- [ ] Log files do NOT contain passwords

## Additional Resources

- For detailed Playwright patterns and selector reference, see [reference.md](reference.md)
- For a complete worked example (customs upload portal), see [examples.md](examples.md)
