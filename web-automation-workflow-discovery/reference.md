# Technical Reference

## Filename Parsing Strategies

When automating file uploads, the input filenames often contain metadata suffixes (e.g., `tracking_merged.pdf`). Extracting the actual identifier requires choosing the right strategy based on accuracy needs, budget, and offline constraints.

**Three deployment-ready strategies** (see `toplogistics-customs-upload/reference.md` for full implementation details):

| Strategy | Cost | Network | Accuracy | Best For |
|----------|------|---------|----------|----------|
| **A. Heuristic Rules** | Free | Offline | 80-95% | Common naming patterns, zero budget |
| **B. Configurable Suffix List** | Free | Offline | 100% (known) | User-maintained, evolving suffixes |
| **C. LLM API** | Paid | Online | 99%+ | Complex/unpredictable naming, budget available |

**Hybrid approach**: Strategy A+B as default baseline; Strategy C as optional enhancement with automatic fallback to A+B on API failure.

Trigger keywords: "文件名提取", "尾程单号识别", "tracking number extraction", "文件名后缀"

---

## Playwright Patterns by Component Type

### Login Forms

**Standard form (page redirect):**
```python
page.goto("https://site.com/login")
page.locator('input[name="username"]').fill(USER)
page.locator('input[name="password"]').fill(PASS)
page.locator('button[type="submit"]').click()
page.wait_for_url("**/dashboard**")
```

**Modal login (SPA, no redirect):**
```python
page.locator('input[name="LoginForm[user]"]').fill(USER)
page.locator('input[name="LoginForm[pwd]"]').fill(PASS)
page.locator('button:has-text("Login")').click()
# Wait for modal to disappear or welcome text to appear
page.wait_for_selector("#modal-login", state="hidden", timeout=30000)
```

### Tables and Grids

**yiiGridView filtering:**
```python
ref_input = page.locator("#task_grid_view input#ImParcel_ref")
ref_input.fill(tracking_number)
page.keyboard.press("Enter")
# Wait for AJAX
page.wait_for_selector(".grid-view-loading", state="hidden", timeout=30000)
```

**DataTables search:**
```python
search = page.locator("input[type='search']")
search.fill(query)
search.press("Enter")
page.wait_for_timeout(1000)  # DataTables debounce
```

### File Upload Components

**plupload:**
```python
# plupload uses a hidden moxie-shim input
file_input = page.locator("input[type='file']")
file_input.set_input_files("/path/to/file.pdf")
# Start button is an <a> tag, not <button>
start_btn = page.locator("a.plupload_start:not(.plupload_disabled)")
start_btn.click()
```

**Native HTML upload:**
```python
page.locator("input[type='file']").set_input_files("/path/to/file.pdf")
page.locator("button:has-text('Upload')").click()
```

**Dropzone.js:**
```python
# Dropzone creates a hidden input with class dz-hidden-input
page.locator("input.dz-hidden-input").set_input_files("/path/to/file.pdf")
```

### Popups and New Tabs

```python
# Detect popup
with page.expect_popup(timeout=15000) as popup_info:
    page.locator("a[target='_blank']").click()
new_page = popup_info.value

# Fallback if no popup
except Exception:
    new_page = page  # Stay on current page
```

### Modals

```python
# Open modal
page.locator("a.tracking-modal-link").first.click()
modal = page.locator("#modal-tracking")
modal.wait_for(state="visible", timeout=15000)

# Close modal
page.locator("#modal_close").click()
# or
page.keyboard.press("Escape")
```

## Selector Reference

### Playwright Selector Engine

| Syntax | Example | Notes |
|--------|---------|-------|
| CSS | `input#username` | Standard CSS selectors |
| Text | `text=Submit` | Exact match, case-sensitive |
| Has-text | `button:has-text("Log in")` | Substring match |
| Role | `role=button[name="Submit"]` | ARIA role + name |
| Test ID | `data-testid=submit-btn` | Best practice for testability |
| XPath | `xpath=//input[@id='x']` | Last resort |

### Scoped Selection

When multiple elements share the same ID or class (bad practice but common in legacy sites), scope the selector:

```python
# Bad — may match elements outside the target grid
page.locator("input#ImParcel_ref")

# Good — scoped to the specific grid container
page.locator("#task_grid_view input#ImParcel_ref")
```

## Wait Strategies

| Strategy | Use When | Code |
|----------|----------|------|
| URL change | Page redirect after action | `page.wait_for_url("**/target**")` |
| Element visible | Modal or dynamic content appears | `page.wait_for_selector("#modal", state="visible")` |
| Element hidden | Loading overlay disappears | `page.wait_for_selector("#loading", state="hidden")` |
| Network idle | All AJAX requests complete | `page.wait_for_load_state("networkidle")` |
| DOM ready | DOM parsed, images may still load | `page.goto(url, wait_until="domcontentloaded")` |
| Custom polling | Complex conditions | Write a `while` loop with `time.sleep()` |

## PyInstaller Bundling

### Single-file exe with bundled browser

```bash
py -m PyInstaller --onefile \
  --add-data "config.html;." \
  --add-data "C:\Users\...\ms-playwright\chromium-XXXX;chromium" \
  --name "MyTool" script.py
```

### Resource path for PyInstaller

```python
def resource_path(relative_path):
    if hasattr(sys, '_MEIPASS'):
        return os.path.join(sys._MEIPASS, relative_path)
    return os.path.join(os.path.abspath("."), relative_path)
```

### Finding bundled vs system Chromium

```python
def find_chromium():
    # 1. Bundled (PyInstaller)
    if hasattr(sys, '_MEIPASS'):
        bundled = os.path.join(sys._MEIPASS, "chromium", "chrome-win", "chrome.exe")
        if os.path.exists(bundled):
            return bundled
    # 2. System playwright install
    candidates = [
        os.path.expandvars(r"%LOCALAPPDATA%\ms-playwright\chromium-1067\chrome-win\chrome.exe"),
    ]
    for path in candidates:
        if os.path.exists(path):
            return path
    return None
```

## Common Page Frameworks

### Yii (PHP framework)
- Grid views: `yiiGridView`, AJAX filtering on Enter key
- Modals: `#modal-*`, content loaded via `.modal-body.load(href)`
- Login: Modal-based, `#modal-login`

### React/Vue/Angular SPAs
- Navigation: prefer `page.goto(direct_url)` over clicking menu items
- State changes: watch for DOM mutations, not URL changes
- Loading: custom spinners (no standard class)

### Bootstrap
- Modals: `.modal`, `.modal-dialog`, `.modal-content`
- Tables: `.table`, `.table-striped`
- Forms: `.form-control`, `.btn-primary`
