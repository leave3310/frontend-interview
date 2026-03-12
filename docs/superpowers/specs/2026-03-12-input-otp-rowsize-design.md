# InputOtp — `rowSize` Prop 設計文件

**日期：** 2026-03-12  
**元件：** `components/base/InputOtp.vue`  
**狀態：** 審核通過

---

## 問題背景

目前 `InputOtp` 元件透過 `length` prop 決定驗證碼位數，所有輸入框固定排成一橫排。  
當 `length` 較大時（例如 8 位以上），單行排列在行動裝置上可能過長，有必要提供多行排列的能力。

---

## 目標

新增 `rowSize` prop，讓呼叫方可以指定每行顯示幾個輸入框，將所有格子從左到右依序填滿每行。

---

## 需求

| 項目 | 規格 |
|---|---|
| 新 prop 名稱 | `rowSize?: number` |
| 預設值 | `props.length`（維持原本單行行為，向下相容）|
| 不整除行為 | 最後一行顯示剩餘格數，不補空格 |
| 焦點移動 | 線性（index + 1），填完一行最後一格自動跳到下一行第一格 |
| Backspace | 同現有邏輯，跨行自然往回移動 |
| Paste | 同現有邏輯，依 index 順序填入，跨行正常 |

---

## 設計方案

### Props 介面

```ts
interface Props {
  length?: number
  rowSize?: number
}

const props = withDefaults(defineProps<Props>(), {
  length: 6
  // rowSize 無靜態預設值，由 effectiveRowSize computed 處理
})
```

### 新增 Computed（`<script setup>` 內）

```ts
// 每行幾格，預設等於 length → 維持單行向下相容
const effectiveRowSize = computed(() => props.rowSize ?? props.length)

// 將 0..length-1 的索引切分成 2D 陣列（每組 rowSize 個）
const rows = computed(() => {
  const result: number[][] = []
  for (let i = 0; i < props.length; i += effectiveRowSize.value) {
    result.push(
      Array.from(
        { length: Math.min(effectiveRowSize.value, props.length - i) },
        (_, j) => i + j
      )
    )
  }
  return result
})
```

### Template 結構

```html
<!-- 外層容器改為 flex-col，讓各行垂直堆疊 -->
<div class="relative flex flex-col items-center gap-2">
  <!-- 外層 v-for 迭代每行 -->
  <div
    v-for="(row, rowIndex) in rows"
    :key="rowIndex"
    class="flex gap-2"
  >
    <!-- 內層 v-for 迭代該行的 digit 索引 -->
    <input
      v-for="index in row"
      :key="index"
      :ref="(el) => { if (el) inputRefs[index] = el as HTMLInputElement }"
      :value="digits[index]"
      ...（其餘 attributes 不變）
    >
  </div>

  <!-- Loading spinner overlay 不變 -->
</div>
```

---

## 不需要修改的部分

以下邏輯全部基於全域 `index`，**無需任何變動**：

- `handleInput(index, event)` — 填入值並線性 focus 下一個
- `handleKeydown(index, event)` — Backspace 往前清除並 focus
- `handlePaste(event)` — 依序填入 digits，focus 最近空格
- `onComplete()` — 收集所有 digits.join('')
- `schema` / `errorMessage` / `validate` — 整體 OTP 字串驗證

---

## 使用範例

```html
<!-- 預設行為：6 格，單行（向下相容）-->
<InputOtp :length="6" @complete="handleOtp" />

<!-- 6 格，每行 3 個 → 2 行 -->
<InputOtp :length="6" :row-size="3" @complete="handleOtp" />

<!-- 8 格，每行 4 個 → 2 行 -->
<InputOtp :length="8" :row-size="4" @complete="handleOtp" />

<!-- 7 格，每行 3 個 → 3 行（最後一行 1 格）-->
<InputOtp :length="7" :row-size="3" @complete="handleOtp" />
```

---

## 邊界情境

| 情況 | 行為 |
|---|---|
| `rowSize >= length` | 單行顯示（等同不傳 rowSize）|
| `rowSize = 1` | 每格獨行 |
| `length % rowSize !== 0` | 最後一行顯示餘數格，不補空 |
| `rowSize` 未傳入 | 等同 `rowSize = length`，單行 |
