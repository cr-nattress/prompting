# Multi-Layered Testing Strategy

> **File Purpose**: Overview of comprehensive testing approach for Vue 3 applications
> **Prerequisites**: Project setup complete
> **Agent Use Case**: Reference when planning test coverage
> **Related Files**: `unit-testing.md`, `component-testing.md`, `e2e-testing.md`, `quality-gates.md`

## In One Sentence

Implement a balanced testing pyramid: unit tests for business logic, component tests for UI contracts, and E2E tests for critical user flows.

## The Testing Pyramid

```
      /\
     /E2E\          Few (slow, expensive, high confidence)
    /------\
   /Component\      Some (medium speed, good confidence)
  /----------\
 /   Unit     \     Many (fast, cheap, focused)
/--------------\
```

## Test Type Comparison

| Test Type | What | Tools | Speed | When |
|-----------|------|-------|-------|------|
| **Unit** | Pure functions, composables, store actions | Vitest | Very Fast | Always |
| **Component** | Component rendering, props, events | Vitest + Vue Test Utils | Fast | UI components |
| **E2E** | Full user flows across app | Playwright | Slow | Critical paths only |

## Coverage Targets

Recommended minimum coverage:
- **Unit tests**: 80% of business logic
- **Component tests**: All public component APIs (props, emits, slots)
- **E2E tests**: 5-10 critical user journeys

## When to Write Which Test

### Unit Tests

**Test**:
- Composables (useFetch, useAuth, etc.)
- Pinia store actions/getters
- Utility functions
- Validators/transformers
- Services/repositories

**Example scenarios**:
- "Does `useCounter` increment correctly?"
- "Does `formatCurrency` handle edge cases?"
- "Does the auth store update token correctly?"

### Component Tests

**Test**:
- Props render correctly
- Events emit with correct payloads
- Slots render content
- Conditional rendering (v-if, v-show)
- User interactions (click, input)

**Example scenarios**:
- "Does UserCard display the user's name?"
- "Does clicking the button emit 'submit'?"
- "Does the form show errors when invalid?"

### E2E Tests

**Test**:
- Complete user journeys
- Multi-page flows
- Integration with backend
- Authentication flows

**Example scenarios**:
- "Can a user register, log in, and place an order?"
- "Does the checkout flow work end-to-end?"
- "Can an admin create and publish a post?"

## Test Naming Conventions

```typescript
// Unit test
describe('useFetch', () => {
  it('should fetch data successfully', () => {})
  it('should handle errors', () => {})
  it('should set loading state', () => {})
})

// Component test
describe('UserCard', () => {
  it('should render user name from props', () => {})
  it('should emit delete event when button clicked', () => {})
  it('should not show email if private prop is true', () => {})
})

// E2E test
test.describe('Authentication Flow', () => {
  test('should allow user to register and log in', async () => {})
  test('should redirect to login if accessing protected route', async () => {})
})
```

## Test File Organization

```
src/
├── components/
│   └── UserCard.vue
│   └── __tests__/
│       └── UserCard.spec.ts        # Component test
├── composables/
│   └── useFetch.ts
│   └── __tests__/
│       └── useFetch.spec.ts        # Unit test
├── stores/
│   └── auth.ts
│   └── __tests__/
│       └── auth.spec.ts            # Unit test
└── services/
    └── api/
        └── users.ts
        └── __tests__/
            └── users.spec.ts       # Unit test

tests/
└── e2e/
    ├── auth.spec.ts                # E2E tests
    ├── checkout.spec.ts
    └── fixtures/
        └── users.json
```

## What NOT to Test

### Don't Test Implementation Details

```typescript
// ❌ Bad: Testing internal state
it('should set internalCounter to 1', () => {
  const wrapper = mount(Counter)
  expect(wrapper.vm.internalCounter).toBe(1)
})

// ✅ Good: Testing public behavior
it('should display count of 1', () => {
  const wrapper = mount(Counter)
  expect(wrapper.text()).toContain('Count: 1')
})
```

### Don't Test Framework Functionality

```typescript
// ❌ Bad: Testing Vue's reactivity
it('should update when ref changes', () => {
  const count = ref(0)
  count.value = 1
  expect(count.value).toBe(1)
})

// ✅ Good: Test your logic that uses reactivity
it('should increment count when button clicked', () => {
  const { count, increment } = useCounter()
  increment()
  expect(count.value).toBe(1)
})
```

### Don't Test Third-Party Libraries

```typescript
// ❌ Bad: Testing axios
it('should make HTTP requests', async () => {
  const response = await axios.get('/api/users')
  expect(response.status).toBe(200)
})

// ✅ Good: Test your service layer (with mocked axios)
it('should fetch users', async () => {
  vi.mocked(axios.get).mockResolvedValue({ data: [mockUser] })
  const users = await fetchUsers()
  expect(users).toEqual([mockUser])
})
```

## Continuous Testing in Development

```bash
# Run tests in watch mode during development
npm run test:unit -- --watch

# Run specific test file
npm run test:unit -- UserCard.spec.ts

# Run tests matching pattern
npm run test:unit -- --grep="UserCard"
```

## Test-Driven Development (TDD) Workflow

1. **Write failing test** (Red)
2. **Write minimal code to pass** (Green)
3. **Refactor** (Refactor)
4. Repeat

```typescript
// 1. Red: Write test first
it('should double the count', () => {
  const { doubled } = useCounter()
  expect(doubled.value).toBe(0)
})

// 2. Green: Implement
export function useCounter() {
  const count = ref(0)
  const doubled = computed(() => count.value * 2)
  return { count, doubled }
}

// 3. Refactor: Improve code while tests pass
```

---

## Navigation

- **Next**: `08-testing/unit-testing.md`
- **See Also**:
  - `08-testing/component-testing.md`
  - `08-testing/e2e-testing.md`
  - `08-testing/quality-gates.md`
- **Up**: `00-overview.md`
