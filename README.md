# Web Automation Workflow Discovery

Learn and automate human workflows on websites through browser exploration, screenshot analysis, and iterative script refinement.

## Use Cases

- Automate repetitive tasks on web applications
- Replicate human click-type-upload sequences
- Build unattended browser automation for any website
- Transform manual operations into repeatable scripts

## Methodology: Six-Phase Workflow

```
Discovery -> Exploration -> Element Mapping
  -> Script Build -> Test & Refine -> Package
```

### Phase 1: Discovery
Clarify target website, human steps, input data, success criteria, frequency, security constraints.

### Phase 2: Exploration
Use Playwright to open the site, take screenshots, explore page structure. Identify page types: SPA, login modal, data grid, upload widget, etc.

### Phase 3: Element Mapping
Build a selector reference table. Priority (most to least reliable): id > name > text-based > scoped selector > CSS class > XPath.

### Phase 4: Script Build
- SPA: prefer direct `goto` over clicking menu buttons
- Use `wait_until="domcontentloaded"` to avoid load timeouts
- Multi-signal login detection (form gone / welcome text appears)
- Use `continue` not `break` in wait loops

### Phase 5: Test & Refine
Diagnose from screenshots, HTML, and stack traces. Fix selector mismatches, timeouts, unexpected page states.

### Phase 6: Package
- PyInstaller resource path compatibility (sys._MEIPASS)
- Bundle Chromium for standalone distribution
- Desktop log files for user diagnostics

## Security Checklist

- [ ] Credentials not hardcoded
- [ ] Credentials not saved to disk
- [ ] Log files do not contain passwords
