# InputOtp v-model Refactor Design

**Date:** 2026-03-12  
**Status:** Approved  

## Problem

`InputOtp.vue` currently:

- Uses `:value + @input` on native `<input>` elements with hard-coded numeric filtering
- Contains internal validation logic (Zod schema + vee-validate `useField`)
- Makes the API call (`verifyOtpSimple`) directly inside the component
- Only exposes `@complete` event to the parent — no v-model interface

This makes the component hard to reuse in contexts where the OTP is not purely numeric, or where validation and API logic differ between systems.

## Goal

Refactor `InputOtp.vue` into a clean, reusable UI component by:

1. Exposing the current OTP value via `v-model` (`defineModel`)
2. Removing hard-coded filtering — callers control their own validation
3. Moving API call and error handling to the caller
4. Replacing internal `loading` ref with a `loading` prop
5. Converting internal native `<input>` elements from `:value + @input` to `v-model`

## New Component API

### Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `length` | `number` | `6` | Number of OTP digits |
| `loading` | `boolean` | `false` | Disables inputs and shows spinner when true |

### v-model

```ts
defineModel<string>({ default: '' })
```

Exposes the current (partial or complete) OTP string to the parent. Updated on every keystroke.

### Emits

| Event | Payload | Description |
|-------|---------|-------------|
| `complete` | `otp: string` | Fired when all `length` digits are filled |

### Usage Example

```vue
<script setup lang="ts">
const otpValue = ref('')
const isVerifying = ref(false)

const handleVerify = async (otp: string) => {
  isVerifying.value = true
  try {
    const response = await useApi().example.verifyOtpSimple(otp)
    if (!response.success) {
      // handle error in parent
    }
  } finally {
    isVerifying.value = false
  }
}
</script>

<template>
  <BaseInputOtp
    v-model="otpValue"
    :length="6"
    :loading="isVerifying"
    @complete="handleVerify"
  />
</template>
```

## Architecture

### Internal State

- `digits: Ref<string[]>` — array of individual digit strings, one per input box
- Model value = `digits.join('')`

### Internal `<input>` elements

Convert from:

```html
:value="digits[index]"
@input="handleInput(index, $event)"
```

To:

```html
v-model="digits[index]"
@input="handleInput(index)"
```

This is valid because filtering is no longer needed — each native `<input>` has `maxlength="1"`, which limits the character count at the HTML level. The `@input` handler now only manages focus movement and model sync.

### Event Handlers

| Handler | Responsibility |
|---------|----------------|
| `handleInput(index)` | Update `model.value`, move focus forward, call `onComplete` if last digit |
| `handleKeydown(index, event)` | Backspace handling: clear current or previous digit, move focus back |
| `handlePaste(event)` | Distribute pasted characters across digit boxes, trigger `onComplete` if full |

### `onComplete`

Simplified to only emit the `complete` event:

```ts
const onComplete = () => {
  const otp = digits.value.join('')
  emit('complete', otp)
}
```

No more `validate()`, no more API call, no more `apiError`.

## What Is Removed

| Item | Reason |
|------|--------|
| `import { z } from 'zod'` | Validation moved to caller |
| `import { toTypedSchema }` | Validation moved to caller |
| `import { useField }` | Validation moved to caller |
| Zod `schema` computed | Validation moved to caller |
| `apiError` ref | Error display moved to caller |
| `loading` ref | Replaced by `loading` prop |
| API call in `onComplete` | Moved to caller's `@complete` handler |
| Error message template block | Caller renders errors outside the component |
| `input.value = val` DOM override | No longer needed — no filtering means v-model handles DOM sync cleanly |
| `syncValue()` vee-validate sync | Removed with vee-validate |

## What Is Retained

- `digits` internal array for per-box state
- Focus movement logic (`focusAt`, `handleKeydown`)
- Paste handling (`handlePaste`)
- Loading spinner overlay (now driven by `props.loading`)
- `@complete` emit

## Trade-offs

**Pros:**
- Component is now a pure UI element — no business logic
- v-model interface is idiomatic Vue 3 and works with vee-validate `useField` from the caller
- Filtering is flexible per use case
- Caller handles API calls, errors, retries

**Cons:**
- Callers must implement their own error display and API handling (more responsibility at call site)
- `maxlength="1"` provides basic length enforcement at HTML level, but not type enforcement — callers must validate character type if needed
