# AGENTS.md

This file provides guidance to Codex (Codex.ai/code) when working with code in this repository.

## Project Overview

This is a uni-app x project for "Yi He Tang" (益禾堂) mobile ordering application, built with UTS (Uni-app TypeScript) and Vue 3. It targets multiple platforms including WeChat mini-program, Alipay, and native mobile apps.

## Development Environment

**Primary IDE:** HBuilderX is the recommended development environment for uni-app x projects with UTS. The project includes `.hbuilderx/launch.json` configuration.

**Install dependencies:**
```bash
npm install
```

**Running the project:** Use HBuilderX's "Run" menu to deploy to browser, emulator, or device. CLI-based builds are not fully documented in this project.

**Debugging:** Use Alipay mini-program developer tools (支付宝小程序开发者工具) for debugging. The project is configured for Alipay mini-program in `manifest.json` with `mp-alipay` settings.

## Architecture

### State Management (Pinia)

All stores are in `store/` directory:
- `app.uts` - Application-level state (current store selection, etc.)
- `cart.uts` - Shopping cart logic and items
- `user.uts` - User authentication and token management
- `address.uts` - User address management

Stores are registered in `main.uts` via `createPinia()`.

### API Request Pattern

**All API calls must use `request()` from `utils/request.uts`:**

```typescript
import { request } from '@/utils/request.uts'

const response = await request<YourResponseType>({
  url: '/api/endpoint',
  method: 'POST',
  data: { ... }
})
```

Key behaviors:
- Base URL: `http://192.168.2.240:8800` (configured in `request.uts`)
- Automatically injects `Authorization: Bearer ${token}` header from user store
- Handles token expiration (code 10101) by auto-logout
- Shows toast notifications for errors
- Returns typed `ApiResponse<T>` with `code`, `message`, `result` structure

### UTS Type System

This project uses UTS (not standard TypeScript). Key differences:
- File extension: `.uts` instead of `.ts`
- Strong typing is enforced - define interfaces for all data structures
- Use `UTSJSONObject` for dynamic object types
- API response types should extend `ApiResponse<T>` interface

### Routing and Pages

Pages are configured in `pages.json`:
- Tab bar pages: index (首页), menu (点单), order (订单), profile (我的)
- Detail pages: checkout, menu/detail, order/detail, store/list, profile/address, profile/address-edit
- Navigation bar titles and styles are defined per-page in `pages.json`

### UI Components

Uses `@dcloudio/uni-ui` with easycom auto-import:
- Components are auto-imported with `uni-*` prefix (e.g., `<uni-card>`)
- Theme color: `#00CC99` (primary green)
- Global styles in `pages.json` globalStyle section

## File Structure

```
pages/          # UI pages organized by feature
  index/        # Home page
  menu/         # Menu browsing and product detail
  order/        # Order list and detail
  profile/      # User profile and address management
  checkout/     # Order checkout flow
  store/        # Store selection
store/          # Pinia state management
utils/          # Utility functions
  request.uts   # Centralized API client
  format.uts    # Data formatting utilities
  common.uts    # Common helper functions
static/         # Static assets (images, icons)
main.uts        # App entry point
App.uvue        # Root component
pages.json      # Page routing and tab bar config
manifest.json   # Platform-specific app configuration
```

## Key Conventions

1. **API Integration:** Always use the centralized `request()` function - never call `uni.request()` directly
2. **State Access:** Import stores with `useXxxStore()` pattern, access state reactively
3. **Error Handling:** The request utility handles common errors; code 10101 triggers automatic logout
4. **Type Safety:** Define interfaces for all API responses and data structures
5. **Component Naming:** Use `.uvue` extension for Vue components in uni-app x projects

## Payment Integration

### uni-pay-x Unified Payment

The project uses **uni-pay-x** for unified payment integration, supporting WeChat Pay and Alipay across multiple platforms.

**Payment utility functions:** `utils/payment.uts`
- `yuanToFen()` / `fenToYuan()` - Currency conversion
- `generateOutTradeNo()` / `generateOrderNo()` - Order number generation
- `buildPaymentParams()` - Build payment parameters
- `validatePaymentAmount()` - Validate payment amount
- `formatPaymentError()` - Format error messages
- `getSupportedProviders()` / `getDefaultProvider()` - Platform detection

**Payment component usage:**

```vue
<template>
  <uni-pay 
    ref="uniPayRef"
    return-url="/pages/order/detail"
    @success="onPaySuccess"
    @fail="onPayFail"
    @cancel="onPayCancel"
  />
</template>

<script setup lang="uts">
import { ref, getCurrentInstance } from 'vue'
import { yuanToFen, generateOutTradeNo, buildPaymentParams } from '@/utils/payment.uts'

const uniPayRef = ref(null)
const instance = getCurrentInstance()

function handlePay() {
  const payOptions = buildPaymentParams({
    provider: "", // Leave empty to show payment method selection
    totalFee: yuanToFen(29.9), // Convert yuan to fen
    orderNo: "ORDER_123",
    outTradeNo: generateOutTradeNo(123),
    description: "Order payment",
    custom: { order_id: 123 } as UTSJSONObject
  })
  
  const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
  payComponent?.open(payOptions)
}

function onPaySuccess(res: UTSJSONObject) {
  // Handle payment success
}
</script>
```

**Key points:**
- Payment amounts are in **fen (分)**, not yuan - use `yuanToFen()` to convert
- `out_trade_no` must be unique for each payment
- `order_no` is your business order number
- `custom` field can store additional data returned in callbacks
- Requires uniCloud configuration - see `docs/UNI_PAY_X_SETUP.md`

**Platform-specific behavior:**
- WeChat mini-program: Only supports `provider: "wxpay"`
- Alipay mini-program: Only supports `provider: "alipay"`
- APP/H5: Supports both, leave `provider: ""` for user selection

**Documentation:**
- Setup guide: `docs/UNI_PAY_X_SETUP.md`
- Usage examples: `docs/PAYMENT_USAGE_EXAMPLES.md`
- Official docs: https://doc.dcloud.net.cn/uniCloud/uni-pay/uni-app-x.html
