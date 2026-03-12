---
name: accessibility-patterns
description: Web accessibility patterns for WCAG 2.1 AA compliance, ARIA roles, keyboard navigation, screen reader support, focus management, and a11y testing strategies.
origin: ECC
---

# Accessibility (a11y) Patterns

Patterns for building inclusive web interfaces that meet WCAG 2.1 AA standards.

## When to Activate

- Building interactive UI components (modals, menus, tabs, forms)
- Auditing existing pages for accessibility compliance
- Implementing keyboard navigation and focus management
- Adding ARIA attributes to custom components
- Writing a11y tests (automated + manual)
- Reviewing color contrast and visual design

## Core Principles

### 1. Semantic HTML First

Use the right element before reaching for ARIA.

```html
<!-- Good: Semantic element -->
<button onclick="handleClick()">Submit</button>
<nav aria-label="Main navigation">
  <ul>
    <li><a href="/home">Home</a></li>
  </ul>
</nav>

<!-- Bad: Div with ARIA (unnecessary) -->
<div role="button" tabindex="0" onclick="handleClick()">Submit</div>
```

### 2. ARIA Only When Needed

The first rule of ARIA: don't use ARIA if native HTML works.

```html
<!-- Disclosure pattern (needs ARIA) -->
<button aria-expanded="false" aria-controls="panel-1">
  Settings
</button>
<div id="panel-1" role="region" hidden>
  Panel content
</div>

<!-- Custom combobox (needs ARIA) -->
<div role="combobox" aria-expanded="true" aria-haspopup="listbox"
     aria-controls="listbox-1" aria-activedescendant="option-2">
  <input type="text" aria-autocomplete="list" />
</div>
<ul id="listbox-1" role="listbox">
  <li id="option-1" role="option">Option 1</li>
  <li id="option-2" role="option" aria-selected="true">Option 2</li>
</ul>
```

### 3. Keyboard Navigation is Non-Negotiable

Every interactive element must be operable by keyboard.

```typescript
// Focus trap for modal dialogs
function useFocusTrap(ref: RefObject<HTMLElement>) {
  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const focusableSelector = [
      'a[href]', 'button:not([disabled])', 'input:not([disabled])',
      'select:not([disabled])', 'textarea:not([disabled])',
      '[tabindex]:not([tabindex="-1"])',
    ].join(', ');

    const focusableElements = element.querySelectorAll<HTMLElement>(focusableSelector);
    const first = focusableElements[0];
    const last = focusableElements[focusableElements.length - 1];

    function handleKeyDown(e: KeyboardEvent) {
      if (e.key !== 'Tab') return;
      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault();
        last.focus();
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault();
        first.focus();
      }
    }

    element.addEventListener('keydown', handleKeyDown);
    first?.focus();
    return () => element.removeEventListener('keydown', handleKeyDown);
  }, [ref]);
}
```

## Common Patterns

### Accessible Modal

```tsx
function Modal({ isOpen, onClose, title, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);
  useFocusTrap(modalRef);

  useEffect(() => {
    if (!isOpen) return;
    const handleEsc = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    document.addEventListener('keydown', handleEsc);
    // Prevent body scroll
    document.body.style.overflow = 'hidden';
    return () => {
      document.removeEventListener('keydown', handleEsc);
      document.body.style.overflow = '';
    };
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return (
    <div className="modal-overlay" onClick={onClose} role="presentation">
      <div
        ref={modalRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        onClick={(e) => e.stopPropagation()}
      >
        <h2 id="modal-title">{title}</h2>
        {children}
        <button onClick={onClose} aria-label="Close dialog">×</button>
      </div>
    </div>
  );
}
```

### Accessible Form

```tsx
function LoginForm() {
  const [errors, setErrors] = useState<Record<string, string>>({});

  return (
    <form aria-label="Login form" noValidate onSubmit={handleSubmit}>
      <div>
        <label htmlFor="email">Email address</label>
        <input
          id="email"
          type="email"
          required
          aria-required="true"
          aria-invalid={!!errors.email}
          aria-describedby={errors.email ? "email-error" : undefined}
          autoComplete="email"
        />
        {errors.email && (
          <p id="email-error" role="alert" className="error">
            {errors.email}
          </p>
        )}
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          required
          aria-required="true"
          aria-invalid={!!errors.password}
          aria-describedby={errors.password ? "password-error" : undefined}
          autoComplete="current-password"
        />
        {errors.password && (
          <p id="password-error" role="alert" className="error">
            {errors.password}
          </p>
        )}
      </div>

      <button type="submit">Sign in</button>
    </form>
  );
}
```

### Live Regions for Dynamic Content

```tsx
// Announce updates to screen readers
function DetectionFeed({ detections }: { detections: Detection[] }) {
  return (
    <>
      {/* Polite: announced when user is idle */}
      <div aria-live="polite" aria-atomic="true" className="sr-only">
        {detections.length} new detections
      </div>

      {/* Assertive: announced immediately (use sparingly) */}
      <div aria-live="assertive" className="sr-only">
        {/* Only for critical alerts */}
      </div>

      <ul role="feed" aria-label="Detection results">
        {detections.map((det) => (
          <li key={det.id} aria-label={`Plate ${det.plateNumber} detected`}>
            {det.plateNumber}
          </li>
        ))}
      </ul>
    </>
  );
}
```

### Skip Navigation

```html
<!-- First element in body -->
<a href="#main-content" class="skip-link">Skip to main content</a>

<header>...</header>
<main id="main-content" tabindex="-1">...</main>

<style>
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  padding: 8px;
  z-index: 100;
  background: #000;
  color: #fff;
}
.skip-link:focus {
  top: 0;
}
</style>
```

## Color Contrast Requirements

```
| Level | Normal Text | Large Text (≥18pt / 14pt bold) |
|-------|-------------|-------------------------------|
| AA    | 4.5:1       | 3:1                           |
| AAA   | 7:1         | 4.5:1                         |

// CSS: Ensure sufficient contrast
:root {
  --text-primary: #1a1a1a;    /* On white: 16.0:1 ✓ */
  --text-secondary: #595959;  /* On white: 7.0:1 ✓ */
  --text-disabled: #767676;   /* On white: 4.5:1 ✓ (minimum!) */
}
```

## Testing

```bash
# Automated tools
npx axe-cli http://localhost:3000              # axe-core scan
npx pa11y http://localhost:3000                # Pa11y
npx lighthouse http://localhost:3000 --only-categories=accessibility

# Playwright a11y test
test('page has no a11y violations', async ({ page }) => {
  await page.goto('/');
  const results = await new AxeBuilder({ page }).analyze();
  expect(results.violations).toEqual([]);
});
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Div/span as button | Use `<button>` element |
| Images without alt | Add descriptive alt (or alt="" for decorative) |
| Outline removed without replacement | Custom focus styles instead |
| Color-only information | Add text, icon, or pattern |
| Auto-playing video/audio | Respect `prefers-reduced-motion`, provide controls |
| Tabindex > 0 | Use 0 or -1 only; fix DOM order instead |
| Missing form labels | Every input needs `<label>` with `htmlFor` |
| ARIA overuse | Prefer semantic HTML elements |
