<script lang="ts">
</script>
<script setup lang="ts">
import { z } from 'zod'
import { toTypedSchema } from '@vee-validate/zod'
import { useField } from 'vee-validate'

interface Props {
  length?: number
}

const props = withDefaults(defineProps<Props>(), {
  length: 6
})

const emit = defineEmits<{
  complete: [otp: string]
}>()

const digits = ref<string[]>(Array(props.length).fill(''))
const inputRefs = ref<HTMLInputElement[]>([])
const loading = ref(false)
const apiError = ref<string | null>(null)

const schema = computed(() =>
  toTypedSchema(
    z.string()
      .length(props.length, `請輸入 ${props.length} 位驗證碼`)
      .regex(/^\d+$/, '只允許輸入數字 0-9')
  )
)

const { errorMessage, setValue, validate } = useField<string>('otp', schema)

const hasError = computed(() => !!(apiError.value || errorMessage.value))
const displayError = computed(() => apiError.value || errorMessage.value || '')

const syncValue = () => {
  setValue(digits.value.join(''))
}

const focusAt = (index: number) => {
  nextTick(() => {
    inputRefs.value[index]?.focus()
  })
}

const handleInput = (index: number, event: Event) => {
  const input = event.target as HTMLInputElement
  const val = input.value.replace(/\D/g, '').slice(-1)
  digits.value[index] = val
  console.log(input.value);
  input.value = val
  console.log(input.value);
  console.log(index);
  console.log(val);
  
  
  
  syncValue()
  apiError.value = null

  if (val && index < props.length - 1) {
    focusAt(index + 1)
  } else if (val && index === props.length - 1) {
    onComplete()
  }
}

const handleKeydown = (index: number, event: KeyboardEvent) => {
  if (event.key === 'Backspace') {
    event.preventDefault()
    if (digits.value[index]) {
      digits.value[index] = ''
      syncValue()
    } else if (index > 0) {
      digits.value[index - 1] = ''
      syncValue()
      focusAt(index - 1)
    }
    apiError.value = null
  }
}

const handlePaste = (event: ClipboardEvent) => {
  event.preventDefault()
  const pasted = (event.clipboardData?.getData('text') ?? '').replace(/\D/g, '').slice(0, props.length)
  if (!pasted) return

  pasted.split('').forEach((char, i) => {
    digits.value[i] = char
  })
  // Clear remaining digits if paste is shorter
  for (let i = pasted.length; i < props.length; i++) {
    digits.value[i] = ''
  }
  syncValue()
  apiError.value = null

  const nextEmpty = digits.value.findIndex(d => !d)
  focusAt(nextEmpty === -1 ? props.length - 1 : nextEmpty)

  if (pasted.length === props.length) {
    onComplete()
  }
}

const onComplete = async () => {
  const result = await validate()
  if (!result.valid) return

  const otp = digits.value.join('')
  emit('complete', otp)

  loading.value = true
  apiError.value = null

  try {
    const response = await useApi().example.verifyOtpSimple(otp)
    if (!response.success) {
      apiError.value = '驗證碼錯誤，請重試'
    }
  } catch {
    apiError.value = '驗證失敗，請稍後再試'
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <div class="flex flex-col items-center gap-3">
    <div class="relative flex items-center gap-2">
      <input
        v-for="(_, index) in digits"
        :key="index"
        :ref="(el) => { if (el) inputRefs[index] = el as HTMLInputElement }"
        :value="digits[index]"
        type="text"
        inputmode="numeric"
        maxlength="1"
        :disabled="loading"
        class="h-12 w-10 rounded-lg border-2 text-center text-lg font-semibold outline-0 transition-colors"
        :class="[
          hasError ? 'border-red-500 bg-red-50' : 'border-black bg-white',
          loading ? 'cursor-not-allowed opacity-50' : ''
        ]"
        autocomplete="off"
        @input="handleInput(index, $event)"
        @keydown="handleKeydown(index, $event)"
        @paste="handlePaste($event)"
      >

      <!-- Loading spinner overlay -->
      <div
        v-if="loading"
        class="absolute inset-0 flex items-center justify-center"
      >
        <svg
          class="h-6 w-6 animate-spin text-black"
          xmlns="http://www.w3.org/2000/svg"
          fill="none"
          viewBox="0 0 24 24"
        >
          <circle
            class="opacity-25"
            cx="12"
            cy="12"
            r="10"
            stroke="currentColor"
            stroke-width="4"
          />
          <path
            class="opacity-75"
            fill="currentColor"
            d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"
          />
        </svg>
      </div>
    </div>

    <!-- Error message -->
    <p
      v-if="displayError"
      class="text-sm text-red-500"
    >
      {{ displayError }}
    </p>
  </div>
</template>
