# uni-pay-x 快速开始指南

## 快速集成（5分钟上手）

### 第一步：在页面中添加支付组件

在你的结账页面（如 `pages/checkout/checkout.uvue`）中添加 `<uni-pay>` 组件：

```vue
<template>
  <view>
    <!-- 你的页面内容 -->
    <button @click="handlePay">立即支付</button>
    
    <!-- 添加 uni-pay 组件 -->
    <uni-pay 
      ref="uniPayRef"
      @success="onPaySuccess"
      @fail="onPayFail"
      @cancel="onPayCancel"
    />
  </view>
</template>
```

### 第二步：导入支付工具函数

```typescript
<script setup lang="uts">
import { ref, getCurrentInstance } from 'vue'
import { yuanToFen, generateOutTradeNo, buildPaymentParams } from '@/utils/payment.uts'

const uniPayRef = ref(null)
const instance = getCurrentInstance()
</script>
```

### 第三步：实现支付逻辑

```typescript
// 发起支付
function handlePay() {
  // 构建支付参数
  const payOptions = buildPaymentParams({
    provider: "", // 留空显示支付方式选择
    totalFee: yuanToFen(29.9), // 金额：29.9元
    orderNo: "ORDER_123456",
    outTradeNo: generateOutTradeNo(123456),
    description: "益禾堂奶茶订单",
    custom: {
      order_id: 123456
    } as UTSJSONObject
  })
  
  // 调用支付
  const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
  payComponent?.open(payOptions)
}

// 支付成功回调
function onPaySuccess(res: UTSJSONObject) {
  uni.showToast({ title: '支付成功', icon: 'success' })
  // 跳转到订单页面
  uni.navigateTo({
    url: '/pages/order/detail'
  })
}

// 支付失败回调
function onPayFail(res: UTSJSONObject) {
  uni.showToast({ 
    title: res['errMsg'] as string || '支付失败', 
    icon: 'none' 
  })
}

// 取消支付回调
function onPayCancel(res: any) {
  uni.showToast({ title: '已取消支付', icon: 'none' })
}
```

### 第四步：配置 uniCloud（必需）

uni-pay-x 依赖 uniCloud，需要完成以下配置：

1. **开通 uniCloud 服务**
   - 登录 https://unicloud.dcloud.net.cn/
   - 创建服务空间
   - 在 HBuilderX 中关联服务空间

2. **上传云函数**
   - 右键 `uniCloud/cloudfunctions/uni-pay-co`
   - 选择"上传部署云函数"

3. **配置支付参数**
   - 在 uniCloud 控制台配置微信/支付宝支付参数
   - 详见：`docs/UNI_PAY_X_SETUP.md`

## 完整示例

### 创建订单 + 支付完整流程

```typescript
async function createOrderAndPay() {
  try {
    // 1. 创建业务订单
    uni.showLoading({ title: '创建订单中...' })
    
    const orderRes = await request<any>({
      url: '/order/create',
      method: 'POST',
      data: {
        items: cartStore.items,
        total_amount: cartStore.totalAmount,
        remark: remark.value
      }
    })
    
    const orderId = orderRes.result.id
    const orderNo = orderRes.result.order_no
    
    uni.hideLoading()
    
    // 2. 发起支付
    const payOptions = buildPaymentParams({
      provider: "",
      totalFee: yuanToFen(cartStore.totalAmount),
      orderNo: orderNo,
      outTradeNo: generateOutTradeNo(orderId),
      description: "益禾堂订单",
      custom: {
        order_id: orderId
      } as UTSJSONObject
    })
    
    const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
    payComponent?.open(payOptions)
    
  } catch (err) {
    uni.hideLoading()
    uni.showToast({ title: '创建订单失败', icon: 'none' })
  }
}

// 支付成功后通知后端
async function onPaySuccess(res: UTSJSONObject) {
  try {
    // 3. 通知后端支付成功
    await request<any>({
      url: '/order/pay_success',
      method: 'POST',
      data: {
        order_id: res.getJSON('custom')?.getNumber('order_id'),
        out_trade_no: res['out_trade_no'],
        transaction_id: res['transaction_id']
      }
    })
    
    uni.showToast({ title: '支付成功', icon: 'success' })
    
    // 4. 清空购物车并跳转
    cartStore.clearCart()
    setTimeout(() => {
      uni.switchTab({
        url: '/pages/order/order'
      })
    }, 1500)
    
  } catch (err) {
    console.error('支付回调失败', err)
  }
}
```

## 常用工具函数

### 1. 金额转换

```typescript
import { yuanToFen, fenToYuan } from '@/utils/payment.uts'

// 元转分
const fen = yuanToFen(29.9) // 2990

// 分转元
const yuan = fenToYuan(2990) // "29.90"
```

### 2. 订单号生成

```typescript
import { generateOutTradeNo, generateOrderNo } from '@/utils/payment.uts'

// 生成支付订单号（唯一）
const outTradeNo = generateOutTradeNo(123) // "PAY_1234567890_123_5678"

// 生成业务订单号
const orderNo = generateOrderNo(123) // "ORDER_123_1234567890"
```

### 3. 金额验证

```typescript
import { validatePaymentAmount } from '@/utils/payment.uts'

if (!validatePaymentAmount(amount)) {
  uni.showToast({ title: '支付金额无效', icon: 'none' })
  return
}
```

### 4. 错误处理

```typescript
import { formatPaymentError } from '@/utils/payment.uts'

function onPayFail(res: UTSJSONObject) {
  const errCode = res.getNumber('errCode') || -1
  const errMsg = res.getString('errMsg') || ''
  
  const friendlyMsg = formatPaymentError(errCode, errMsg)
  uni.showToast({ title: friendlyMsg, icon: 'none' })
}
```

## 平台差异

### 微信小程序

```typescript
// 微信小程序只能使用微信支付
const payOptions = {
  provider: "wxpay", // 必须指定为 wxpay
  total_fee: yuanToFen(29.9),
  order_no: orderNo,
  out_trade_no: outTradeNo,
  description: "订单支付"
} as UTSJSONObject
```

### 支付宝小程序

```typescript
// 支付宝小程序只能使用支付宝支付
const payOptions = {
  provider: "alipay", // 必须指定为 alipay
  total_fee: yuanToFen(29.9),
  order_no: orderNo,
  out_trade_no: outTradeNo,
  description: "订单支付"
} as UTSJSONObject
```

### APP / H5

```typescript
// APP 和 H5 可以选择支付方式
const payOptions = {
  provider: "", // 留空显示支付方式选择弹窗
  total_fee: yuanToFen(29.9),
  order_no: orderNo,
  out_trade_no: outTradeNo,
  description: "订单支付"
} as UTSJSONObject
```

### 自动适配平台

```typescript
import { getDefaultProvider } from '@/utils/payment.uts'

const payOptions = {
  provider: getDefaultProvider(), // 自动选择当前平台的默认支付方式
  total_fee: yuanToFen(29.9),
  order_no: orderNo,
  out_trade_no: outTradeNo,
  description: "订单支付"
} as UTSJSONObject
```

## 常见问题

### Q1: 支付金额单位是什么？

**A:** uni-pay-x 使用 **分（fen）** 作为金额单位，不是元。

```typescript
// ❌ 错误：直接使用元
total_fee: 29.9

// ✅ 正确：转换为分
total_fee: yuanToFen(29.9) // 2990
```

### Q2: out_trade_no 和 order_no 有什么区别？

**A:** 
- `out_trade_no`: 支付订单号，必须全局唯一，用于支付系统
- `order_no`: 业务订单号，你的系统中的订单编号

```typescript
const payOptions = {
  order_no: "ORDER_123456", // 你的业务订单号
  out_trade_no: "PAY_1234567890_123456_5678", // 支付订单号（必须唯一）
  // ...
}
```

### Q3: 如何测试支付？

**A:** 
1. 使用真实的商户号配置
2. 支付金额设置为 0.01 元进行测试
3. 支付宝提供沙箱环境，微信需要使用真实环境

```typescript
// 开发环境使用小额测试
const isDev = true // 根据实际情况判断

const payOptions = {
  total_fee: isDev ? 1 : yuanToFen(actualAmount), // 开发环境 0.01 元
  description: isDev ? "[测试]订单支付" : "订单支付"
  // ...
}
```

### Q4: 支付失败如何处理？

**A:** 在 `onPayFail` 回调中处理：

```typescript
function onPayFail(res: UTSJSONObject) {
  const errCode = res.getNumber('errCode') || -1
  const errMsg = res.getString('errMsg') || ''
  
  // 格式化错误信息
  const friendlyMsg = formatPaymentError(errCode, errMsg)
  
  // 显示错误提示
  uni.showToast({ title: friendlyMsg, icon: 'none' })
  
  // 记录日志
  console.error('支付失败:', { errCode, errMsg })
}
```

### Q5: 如何查询订单支付状态？

**A:** 使用 `getOrder` 方法：

```typescript
const payComponent = instance?.proxy?.$refs['uniPayRef'] as any

const result = await payComponent?.getOrder({
  out_trade_no: outTradeNo
})

if (result.errCode === 0 && result.has_paid) {
  console.log('订单已支付')
}
```

## 下一步

- 📖 查看完整配置指南：`docs/UNI_PAY_X_SETUP.md`
- 💡 查看更多使用示例：`docs/PAYMENT_USAGE_EXAMPLES.md`
- 🔗 官方文档：https://doc.dcloud.net.cn/uniCloud/uni-pay/uni-app-x.html

## 技术支持

- uni-pay-x 官方文档：https://doc.dcloud.net.cn/uniCloud/uni-pay/uni-app-x.html
- DCloud 社区：https://ask.dcloud.net.cn/
- 微信支付文档：https://pay.weixin.qq.com/wiki/doc/api/index.html
- 支付宝开放平台：https://opendocs.alipay.com/
