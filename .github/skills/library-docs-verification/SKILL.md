---
name: library-docs-verification
description: Patterns for verifying library documentation, checking version compatibility, detecting deprecated APIs, and keeping dependencies up to date across Python, Node.js, Go, and Rust projects.
---

# Library Documentation Verification

Patterns for staying current with library APIs, detecting version drift, and avoiding deprecated syntax.

## When to Activate

- Using an external library you haven't used recently
- Upgrading dependencies (major or minor version bumps)
- Encountering unexpected behavior from a library call
- Reviewing code that uses third-party APIs
- Setting up a new project with dependency choices
- Debugging errors that might be version-related

## Core Principles

### 1. Trust Docs Over Memory

Library APIs change. Always verify against current documentation before assuming syntax.

```python
# Example: ultralytics API changed over versions
# v8.0: model.predict(source="img.jpg", save=True)
# v8.1+: model.predict(source="img.jpg", save=True, stream=True)  # new param
# v8.3+: model.predict(source="img.jpg", save=True, vid_stride=1)  # video stride added

# ALWAYS check: https://docs.ultralytics.com/modes/predict/
```

### 2. Pin Versions, Document Why

```
# requirements.txt — pin exact versions with reason
ultralytics==8.4.21    # LPR model trained on this version
opencv-python==4.13.0.92  # Tested with this; 4.14 changes VideoCapture API
torch==2.10.0          # CUDA 12.x support required
```

### 3. Check Before Upgrade

Never blindly upgrade. Check the changelog.

```bash
# Python: Check what would change
pip list --outdated
pip install --dry-run --upgrade ultralytics

# Node.js: Check outdated packages
npm outdated
npx npm-check-updates

# Check changelogs
# https://github.com/{org}/{repo}/blob/main/CHANGELOG.md
# https://github.com/{org}/{repo}/releases
```

## Version Verification Workflow

### Step 1: Identify Installed Versions

```python
# Python
import importlib.metadata

def get_installed_versions(packages: list[str]) -> dict[str, str]:
    versions = {}
    for pkg in packages:
        try:
            versions[pkg] = importlib.metadata.version(pkg)
        except importlib.metadata.PackageNotFoundError:
            versions[pkg] = "NOT INSTALLED"
    return versions

# Quick check
packages = ["ultralytics", "opencv-python", "torch", "torchvision", "polars"]
for pkg, ver in get_installed_versions(packages).items():
    print(f"{pkg}: {ver}")
```

### Step 2: Compare Against Latest

```bash
# Python: Check PyPI for latest
pip index versions ultralytics 2>/dev/null || pip install --dry-run ultralytics

# Node.js: Check npm registry
npm view react version
npm view next versions --json | python -c "import sys,json; print(json.load(sys.stdin)[-5:])"
```

### Step 3: Check for Breaking Changes

Common places to find breaking changes:
1. **CHANGELOG.md** / **HISTORY.md** in repo root
2. **GitHub Releases** page (look for "Breaking Changes" section)
3. **Migration guides** in official docs
4. **Deprecation warnings** in previous versions

### Step 4: Verify Code Against Docs

```python
# Pattern: Check if method exists and signature matches
import inspect

def verify_api(obj, method_name: str, expected_params: list[str]) -> dict:
    """Verify a library method exists and has expected parameters."""
    if not hasattr(obj, method_name):
        return {"status": "MISSING", "method": method_name}
    
    method = getattr(obj, method_name)
    sig = inspect.signature(method)
    actual_params = list(sig.parameters.keys())
    
    missing = [p for p in expected_params if p not in actual_params]
    extra = [p for p in actual_params if p not in expected_params and p != "self"]
    
    return {
        "status": "OK" if not missing else "MISMATCH",
        "missing_params": missing,
        "new_params": extra,
        "actual_signature": str(sig),
    }
```

## Common Library Changes to Watch

### Ultralytics YOLO (your project)

```python
# v8.0 → v8.1: Added streaming mode
results = model.predict(source, stream=True)  # Returns generator

# v8.1 → v8.2: Results API changes
for r in results:
    boxes = r.boxes.xyxy     # Tensor of [x1, y1, x2, y2]
    confs = r.boxes.conf     # Confidence scores
    classes = r.boxes.cls    # Class indices

# v8.3+: New export formats, new task types
model.export(format="ncnn")   # New in 8.3
model.export(format="paddle") # New in 8.3
```

### OpenCV

```python
# v4.x: Common changes to watch
# cv2.findContours returns (contours, hierarchy) — NOT (image, contours, hierarchy) like v3

# v4.8+: DNN module changes for ONNX models
net = cv2.dnn.readNetFromONNX("model.onnx")  # Verify supported ops

# v4.10+: New VideoCapture backends
cap = cv2.VideoCapture(0, cv2.CAP_V4L2)  # Explicit backend
```

### PyTorch

```python
# v2.0: torch.compile() introduced
model = torch.compile(model)  # New in 2.0

# v2.1: torch.inference_mode preferred over torch.no_grad
with torch.inference_mode():  # Faster than no_grad (2.0+)
    output = model(input)

# v2.4+: New AMP API
with torch.autocast("cuda"):  # Replaces torch.cuda.amp.autocast
    output = model(input)
```

### React

```typescript
// v18: Concurrent features
// createRoot instead of ReactDOM.render
import { createRoot } from 'react-dom/client';
const root = createRoot(document.getElementById('root')!);
root.render(<App />);

// v19: use() hook, Actions, server components
import { use } from 'react';
const data = use(fetchPromise);  // New in React 19
```

### Next.js

```typescript
// v13→14: App Router changes
// v14: Server Actions stable, Partial Prerendering
// v15: React 19, Turbopack stable
// Check: https://nextjs.org/blog
```

## Security Vulnerability Checking

```bash
# Python
pip-audit                          # Audit installed packages
safety check                       # Check for known vulnerabilities

# Node.js
npm audit                          # Built-in vulnerability check
npx auditjs ossi                   # OSS Index check

# General
# Check https://nvd.nist.gov/ for CVEs
# Check https://snyk.io/vuln/ for vulnerability database
```

## Dependency Health Assessment

```markdown
## Dependency Health Report

| Package | Installed | Latest | Status | Action |
|---------|-----------|--------|--------|--------|
| ultralytics | 8.4.21 | 8.4.21 | ✅ Current | None |
| opencv-python | 4.13.0 | 4.14.0 | ⚠️ Minor | Review changelog |
| torch | 2.10.0 | 2.10.0 | ✅ Current | None |
| numpy | 2.4.3 | 2.5.0 | ⚠️ Minor | Check deprecations |

### Deprecated APIs Found
1. `np.float` → Use `float` or `np.float64` (removed in NumPy 1.24)
2. `torch.cuda.amp.autocast` → Use `torch.autocast("cuda")` (PyTorch 2.4+)

### Breaking Changes in Scope
- None for current pinned versions
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Using library from memory without checking docs | Fetch current docs first |
| Unpinned dependencies | Pin exact versions in requirements |
| Ignoring deprecation warnings | Fix before they become errors |
| Upgrading all deps at once | Upgrade one at a time, test each |
| No changelog review before upgrade | Always read CHANGELOG.md |
| Trusting Stack Overflow answers from 2020 | Verify against current docs |
| No vulnerability scanning | Run pip-audit / npm audit in CI |
| Using alpha/beta in production | Pin stable releases only |
