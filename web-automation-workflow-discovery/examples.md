# Worked Example: Customs Document Upload Portal

This example walks through the complete workflow discovery process for automating PDF uploads to a logistics portal.

## Discovery Phase

**User request**: "Automate uploading customs inspection PDFs to TLA IMS portal"

**Gathered context**:
- Website: `https://ims.toplogistics.com.au`
- Human workflow: Login with captcha -> click Held Shipments -> search by tracking number -> click View -> click Upload Document -> select PDF -> start upload
- Input data: Folder of PDF files named by tracking number (e.g., `6013026525458.pdf`)
- Security: Credentials must not be saved to disk
- Target users: Multiple people should be able to use the tool

## Exploration Phase

**Step 1: Entry page analysis**

Opened `https://ims.toplogistics.com.au/shipment/heldList.app`. Screenshot revealed:
- A login modal (`#modal-login`) overlays the page
- Form inputs: `input[name="LoginForm[user]"]` and `input[name="LoginForm[pwd]"]`
- Captcha image present — requires human interaction

**Step 2: Post-login navigation**

After login, the SPA shows a dashboard. Instead of clicking "Held Shipments" (unreliable AJAX), directly navigated to the held list URL:
```python
page.goto("https://ims.toplogistics.com.au/shipment/heldList.app",
          timeout=120000, wait_until="domcontentloaded")
```

**Step 3: Grid analysis**

HTML dump revealed:
- yiiGridView table with ID `#task_grid_view`
- Filter input: `input#ImParcel_ref` inside the grid container
- View button: `a.tracking-modal-link` in each row
- AJAX loading indicator: `#loading-container`

**Step 4: Modal and upload analysis**

Clicking View opens `#modal-tracking` with dynamic content. Inside the modal:
- Upload link text is singular: `"Upload Document"` (not plural)
- Clicking it opens a popup to `https://os.toplogistics.com.au/parcelStatus?hbn=...`
- Upload page uses **pluploadQueue** component
- File input is hidden: `input[type="file"]` inside moxie-shim
- Start button: `<a class="plupload_start">` (NOT a `<button>`)

## Element Mapping

```
Element                        | Selector
-------------------------------|---------------------------------------------
Username input                 | input[name="LoginForm[user]"]
Password input                 | input[name="LoginForm[pwd]"]
Login button                   | button:has-text("Login")
Held list direct URL           | https://ims.toplogistics.com.au/shipment/heldList.app
Grid container                 | #task_grid_view
Tracking filter (scoped)       | #task_grid_view input#ImParcel_ref
View link                      | #task_grid_view table tbody tr:first-child a.tracking-modal-link
Modal                          | #modal-tracking
Upload link                    | a:has-text("Upload Document")
File input (plupload)          | input[type="file"]
Start upload button            | a.plupload_start:not(.plupload_disabled)
Uploaded files grid            | #_import_custom-grid table tbody tr
```

## Script Build

### Key implementation decisions

1. **Direct URL navigation** instead of clicking SPA menu items
2. **Dual login detection**: form gone OR welcome text appears (SPA may refresh content)
3. **Exception resilience in wait loops**: `continue` on error, never `break`
4. **Scoped selectors**: `#task_grid_view input#ImParcel_ref` prevents matching other forms
5. **plupload awareness**: `set_input_files()` on hidden input, `<a>` tag for start button
6. **Duplicate detection**: Scan uploaded files grid before uploading

### Security implementation

- Credentials entered via local `config.html` page loaded in the browser
- Only `pdf_dir` saved to `upload_config.json`
- Username/password fields in HTML are always empty by default

## Test & Refine

**Failure 1**: Page load timeout on `heldList.app`
- Fix: `wait_until="domcontentloaded"`, `timeout=120000`

**Failure 2**: Login loop exits after 24 seconds
- Root cause: `except: break` in wait loop — page refresh caused exception
- Fix: `except Exception: time.sleep(0.5); continue`

**Failure 3**: `input#ImParcel_ref` resolves to 2 elements
- Root cause: A second form on the page also used that ID
- Fix: Scoped to `#task_grid_view input#ImParcel_ref`

**Failure 4**: Start Upload button not found
- Root cause: Assumed `<button>`, actual element is `<a class="plupload_start">`
- Fix: `a.plupload_start:not(.plupload_disabled)`

**Failure 5**: Exe can't find Chromium on other machines
- Root cause: Playwright looks for browser in temp extraction dir
- Fix: Bundle Chromium with `--add-data`, detect bundled path via `_MEIPASS`

## Package

### Final script structure

```
auto_upload_final.py
config.html
upload_config.json  (runtime generated, only stores pdf_dir)
```

### PyInstaller command

```bash
py -m PyInstaller --onefile \
  --add-data "config.html;." \
  --add-data "C:\Users\...\ms-playwright\chromium-1067;chromium" \
  --name "TLA海关查验上传" auto_upload_final.py
```

### UX additions

- **Desktop log file**: `TLA_upload_log.txt` for debugging
- **Completion dialog**: Injected HTML overlay with 30-second countdown and "Close Now" button
- **Config pre-fill**: `pdf_dir` from previous run passed via query string to `config.html`

## Result

A 148MB single-file exe that:
1. Opens a config page for the user to enter pdf folder, username, password
2. Navigates to the portal, logs in (human enters captcha)
3. Processes all PDFs: search -> view -> upload -> move to "uploaded" subfolder
4. Shows completion dialog and closes automatically
5. Requires zero pre-installed software on the target machine
