# Accessibility Best Practices

> **File Purpose**: Implement ARIA, semantic HTML, focus management
> **Agent Use Case**: Reference when building accessible UIs

## Key Practices

1. **Semantic HTML**: Use `<button>`, `<nav>`, `<main>`, etc.
2. **ARIA attributes**: `aria-label`, `role`, `aria-live`
3. **Keyboard navigation**: Ensure all interactive elements are keyboard-accessible
4. **Focus management**: Programmatically manage focus for modals, drawers

```vue
<button aria-label="Close modal" @click="close">×</button>

<div role="dialog" aria-modal="true" aria-labelledby="modal-title">
  <h2 id="modal-title">Modal Title</h2>
</div>
```

---

## Navigation
- **Up**: `00-overview.md`
