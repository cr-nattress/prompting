# Component Testing

> **File Purpose**: Test component rendering, props, emits
> **Agent Use Case**: Reference when testing Vue components

## Example

```typescript
import { mount } from '@vue/test-utils'
import UserCard from './UserCard.vue'

it('should render user name', () => {
  const wrapper = mount(UserCard, {
    props: { user: { id: 1, name: 'John' } }
  })
  expect(wrapper.text()).toContain('John')
})

it('should emit delete on button click', async () => {
  const wrapper = mount(UserCard, { props: { user: mockUser } })
  await wrapper.find('button').trigger('click')
  expect(wrapper.emitted('delete')).toBeTruthy()
})
```

---

## Navigation
- **Up**: `00-overview.md`
