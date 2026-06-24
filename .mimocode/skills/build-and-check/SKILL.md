---
description: Build and validate the email-otp-autofill project: TypeScript type-check the agent, syntax-check Chrome extension JS files, and validate manifest.json. Run this before any commit or after code changes to catch errors early.
name: build-and-check
---

# Build & Check

Run a comprehensive build validation across the agent backend and Chrome extension before committing or deploying.

## When to use

- After making code changes to either `agent/` or `chrome-extension/`
- Before committing (pre-flight check)
- When user says "check", "build", "compile", "validate", "有没有报错"

## Steps

### 1. Agent TypeScript type-check

```bash
cd agent && npx tsc -p tsconfig.json --noEmit; echo "TSC=$?"
```

All changes must compile cleanly. If TSC != 0, fix errors before proceeding.

### 2. Chrome extension JS syntax check

```bash
cd chrome-extension && for f in background.js options.js popup.js i18n.js content.js; do
  node --check "$f" && echo "  ✓ $f" || echo "  ✗ $f FAIL"
done
```

Every JS file must pass `node --check`.

### 3. Manifest JSON validation

```bash
cd chrome-extension && node -e "JSON.parse(require('fs').readFileSync('manifest.json','utf8')); console.log('✓ manifest.json valid')"
```

### 4. Admin inline JS parse check (if admin/index.html was modified)

```bash
cd agent/admin && node -e "
const s = require('fs').readFileSync('index.html','utf8');
const m = s.match(/<script>([\s\S]*)<\/script>/);
new Function(m[1]);
console.log('✓ admin inline JS parseable')
"
```

### 5. Quick residual file check

```bash
ls agent/_*.ts chrome-extension/_*.html 2>/dev/null && echo "⚠ temp files found!" || echo "✓ no temp files"
```

## Stopping condition

- All checks pass (✓) → proceed
- Any check fails → fix the error, re-run the failing step only

## Notes

- The agent uses `tsx` at runtime but `tsc --noEmit` for type-checking only (no emit).
- Chrome extension is plain JS (no build step), so `node --check` is sufficient.
- The admin panel is a single HTML file with inline `<script>` — parse check catches syntax errors.
