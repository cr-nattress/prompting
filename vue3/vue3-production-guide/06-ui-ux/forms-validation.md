# Forms & Validation

> **File Purpose**: Implement forms with Vee-Validate + Zod
> **Agent Use Case**: Reference when building forms

## Example

```vue
<script setup lang="ts">
import { useForm } from 'vee-validate'
import { toTypedSchema } from '@vee-validate/zod'
import { LoginFormSchema } from '@/types/forms'

const { handleSubmit, errors } = useForm({
  validationSchema: toTypedSchema(LoginFormSchema)
})

const onSubmit = handleSubmit((values) => {
  // values are typed and validated
  login(values)
})
</script>

<template>
  <form @submit="onSubmit">
    <Field name="email" type="email" />
    <span>{{ errors.email }}</span>
    <button type="submit">Submit</button>
  </form>
</template>
```

---

## Navigation
- **Up**: `00-overview.md`
