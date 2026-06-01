# uni-pay-x 使用示例

## 一、基础使用示例

### 1. 在页面中集成支付组件

```vue
<template>
  <view>
    <button @click="handlePay">立即支付</button>
    
    <!-- uni-pay-x 组件 -->
    <uni-pay 
      ref="uniPayRef"
      return-url="/pages/order/detail"
      @success="onPaySuccess"
      @fail="onPayFail"
      @cancel="onPayCancel"
    />
  </view>
</template>

<script setup lang="uts">
import { ref, getCurrentInstance } from 'vue'
import { yuanToFen, generateOutTradeNo, buildPaymentParams } from '@/utils/payment.uts'

const uniPayRef = ref(null)
const instance = getCurrentInstance()

function handlePay() {
  // 构建支付参数
  const payOptions = buildPaymentParams({
    provider: "", // 留空显示支付方式选择
    totalFee: yuanToFen(29.9), // 29.9元转为2990分
    orderNo: "ORDER_123456",
    outTradeNo: generateOutTradeNo(123456),
    description: "益禾堂奶茶订单",
    custom: {
      order_id: 123456,
      user_id: 1001
    } as UTSJSONObject
  })
  
  // 调用支付
  const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
  payComponent?.open(payOptions)
}

function onPaySuccess(res: UTSJSONObject) {
  console.log('支付成功', res)
  uni.showToast({ title: '支付成功', icon: 'success' })
}

function onPayFail(res: UTSJSONObject) {
  console.log('支付失败', res)
  uni.showToast({ title: '支付失败', icon: 'none' })
}

function onPayCancel(res: any) {
  console.log('取消支付', res)
}
</script>
```

### 2. 直接指定支付方式

```typescript
// 直接使用支付宝支付（适用于支付宝小程序）
const payOptions = {
  provider: "alipay",
  total_fee: 2990, // 29.9元 = 2990分
  order_no: "ORDER_123456",
  out_trade_no: "PAY_123456_1234567890",
  description: "商品订单"
} as UTSJSONObject

payComponent?.open(payOptions)
```

### 3. 直接使用微信支付（适用于微信小程序）

```typescript
const payOptions = {
  provider: "wxpay",
  total_fee: 2990,
  order_no: "ORDER_123456",
  out_trade_no: "PAY_123456_1234567890",
  description: "商品订单"
} as UTSJSONObject

payComponent?.open(payOptions)
```

## 二、完整支付流程示例

### 1. 创建订单 + 支付

```typescript
async function createOrderAndPay() {
  try {
    // 步骤1: 创建业务订单
    uni.showLoading({ title: '创建订单中...' })
    
    const orderRes = await request<any>({
      url: '/order/create',
      method: 'POST',
      data: {
        items: cartItems,
        total_amount: 29.9,
        remark: '少糖少冰'
      }
    })
    
    const orderId = orderRes.result.id
    const orderNo = orderRes.result.order_no
    
    uni.hideLoading()
    
    // 步骤2: 发起支付
    const payOptions = {
      provider: "",
      total_fee: yuanToFen(29.9),
      order_no: orderNo,
      out_trade_no: generateOutTradeNo(orderId),
      description: "益禾堂订单",
      custom: {
        order_id: orderId
      } as UTSJSONObject
    } as UTSJSONObject
    
    const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
    payComponent?.open(payOptions)
    
  } catch (err) {
    uni.hideLoading()
    uni.showToast({ title: '创建订单失败', icon: 'none' })
  }
}

// 支付成功回调
async function onPaySuccess(res: UTSJSONObject) {
  try {
    // 步骤3: 通知后端支付成功
    await request<any>({
      url: '/order/pay_callback',
      method: 'POST',
      data: {
        order_id: res.getJSON('custom')?.getNumber('order_id'),
        out_trade_no: res['out_trade_no'],
        transaction_id: res['transaction_id']
      }
    })
    
    uni.showToast({ title: '支付成功', icon: 'success' })
    
    // 步骤4: 跳转到订单详情
    setTimeout(() => {
      uni.navigateTo({
        url: `/pages/order/detail?id=${res.getJSON('custom')?.getNumber('order_id')}`
      })
    }, 1500)
    
  } catch (err) {
    console.error('支付回调失败', err)
  }
}
```

## 三、高级用法

### 1. 查询订单支付状态

```typescript
import { ref } from 'vue'

const uniPayRef = ref(null)

async function checkPaymentStatus(outTradeNo: string) {
  try {
    const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
    
    // 调用 uni-pay 的 getOrder 方法查询订单
    const result = await payComponent?.getOrder({
      out_trade_no: outTradeNo
    })
    
    if (result.errCode === 0) {
      const hasPaid = result.has_paid
      if (hasPaid) {
        console.log('订单已支付')
      } else {
        console.log('订单未支付')
      }
    }
  } catch (err) {
    console.error('查询失败', err)
  }
}
```

### 2. 订单退款

```typescript
async function refundOrder(outTradeNo: string, refundFee: number) {
  try {
    const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
    
    const result = await payComponent?.refund({
      out_trade_no: outTradeNo,
      refund_fee: refundFee, // 退款金额（分）
      refund_desc: '用户申请退款'
    })
    
    if (result.errCode === 0) {
      uni.showToast({ title: '退款成功', icon: 'success' })
    } else {
      uni.showToast({ title: result.errMsg || '退款失败', icon: 'none' })
    }
  } catch (err) {
    console.error('退款失败', err)
  }
}
```

### 3. 关闭订单

```typescript
async function closeOrder(outTradeNo: string) {
  try {
    const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
    
    const result = await payComponent?.closeOrder({
      out_trade_no: outTradeNo
    })
    
    if (result.errCode === 0) {
      uni.showToast({ title: '订单已关闭', icon: 'success' })
    }
  } catch (err) {
    console.error('关闭订单失败', err)
  }
}
```

## 四、错误处理

### 1. 完整的错误处理示例

```typescript
import { formatPaymentError } from '@/utils/payment.uts'

function onPayFail(res: UTSJSONObject) {
  const errCode = res.getNumber('errCode') || -1
  const errMsg = res.getString('errMsg') || ''
  
  // 格式化错误信息
  const friendlyMsg = formatPaymentError(errCode, errMsg)
  
  uni.showToast({
    title: friendlyMsg,
    icon: 'none',
    duration: 2000
  })
  
  // 记录错误日志
  console.error('支付失败:', {
    errCode,
    errMsg,
    timestamp: Date.now()
  })
}

function onPayCancel(res: any) {
  // 用户主动取消支付
  uni.showToast({
    title: '已取消支付',
    icon: 'none'
  })
  
  // 可选：跳转到订单列表
  setTimeout(() => {
    uni.navigateTo({
      url: '/pages/order/order'
    })
  }, 1500)
}
```

### 2. 支付前验证

```typescript
import { validatePaymentAmount } from '@/utils/payment.uts'

function handlePay() {
  // 验证金额
  if (!validatePaymentAmount(totalAmount)) {
    uni.showToast({
      title: '支付金额无效',
      icon: 'none'
    })
    return
  }
  
  // 验证用户登录状态
  if (!userStore.token) {
    uni.showToast({
      title: '请先登录',
      icon: 'none'
    })
    uni.navigateTo({
      url: '/pages/login/login'
    })
    return
  }
  
  // 验证订单状态
  if (orderStatus !== 'pending') {
    uni.showToast({
      title: '订单状态异常',
      icon: 'none'
    })
    return
  }
  
  // 发起支付
  startPayment()
}
```

## 五、平台差异处理

### 1. 根据平台选择支付方式

```typescript
import { getDefaultProvider, getSupportedProviders } from '@/utils/payment.uts'

function handlePay() {
  const provider = getDefaultProvider()
  const supportedProviders = getSupportedProviders()
  
  console.log('当前平台支持的支付方式:', supportedProviders)
  console.log('默认支付方式:', provider)
  
  const payOptions = {
    provider: provider, // 自动选择平台默认支付方式
    total_fee: 2990,
    order_no: "ORDER_123",
    out_trade_no: "PAY_123",
    description: "订单支付"
  } as UTSJSONObject
  
  const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
  payComponent?.open(payOptions)
}
```

### 2. 条件编译处理

```typescript
function handlePay() {
  let provider = ""
  
  // #ifdef MP-WEIXIN
  provider = "wxpay" // 微信小程序只能用微信支付
  // #endif
  
  // #ifdef MP-ALIPAY
  provider = "alipay" // 支付宝小程序只能用支付宝支付
  // #endif
  
  // #ifdef APP || H5
  provider = "" // APP和H5可以选择支付方式
  // #endif
  
  const payOptions = {
    provider: provider,
    total_fee: 2990,
    order_no: "ORDER_123",
    out_trade_no: "PAY_123",
    description: "订单支付"
  } as UTSJSONObject
  
  const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
  payComponent?.open(payOptions)
}
```

## 六、最佳实践

### 1. 支付状态管理

```typescript
import { ref } from 'vue'

const paymentStatus = ref<'idle' | 'creating' | 'paying' | 'success' | 'failed'>('idle')
const currentOrderId = ref(0)

async function handlePay() {
  if (paymentStatus.value === 'paying') {
    uni.showToast({ title: '支付进行中...', icon: 'none' })
    return
  }
  
  try {
    paymentStatus.value = 'creating'
    
    // 创建订单
    const orderRes = await createOrder()
    currentOrderId.value = orderRes.id
    
    paymentStatus.value = 'paying'
    
    // 发起支付
    const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
    payComponent?.open(buildPaymentParams({
      totalFee: yuanToFen(orderRes.amount),
      orderNo: orderRes.order_no,
      outTradeNo: generateOutTradeNo(orderRes.id),
      description: "订单支付"
    }))
    
  } catch (err) {
    paymentStatus.value = 'failed'
    uni.showToast({ title: '操作失败', icon: 'none' })
  }
}

function onPaySuccess(res: UTSJSONObject) {
  paymentStatus.value = 'success'
  // 处理支付成功逻辑
}

function onPayFail(res: UTSJSONObject) {
  paymentStatus.value = 'failed'
  // 处理支付失败逻辑
}
```

### 2. 支付超时处理

```typescript
const paymentTimeout = ref<number | null>(null)

function startPayment() {
  // 设置支付超时（5分钟）
  paymentTimeout.value = setTimeout(() => {
    uni.showModal({
      title: '支付超时',
      content: '支付时间过长，是否继续等待？',
      success: (res) => {
        if (res.confirm) {
          // 继续等待
          startPayment()
        } else {
          // 取消支付
          cancelPayment()
        }
      }
    })
  }, 5 * 60 * 1000) as unknown as number
  
  // 发起支付
  const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
  payComponent?.open(payOptions)
}

function onPaySuccess(res: UTSJSONObject) {
  // 清除超时定时器
  if (paymentTimeout.value) {
    clearTimeout(paymentTimeout.value)
    paymentTimeout.value = null
  }
  
  // 处理支付成功
}
```

### 3. 支付日志记录

```typescript
function logPaymentEvent(event: string, data: any) {
  const log = {
    event: event,
    data: data,
    timestamp: Date.now(),
    user_id: userStore.userId,
    platform: uni.getSystemInfoSync().platform
  }
  
  console.log('[Payment Log]', log)
  
  // 可选：上报到服务器
  // reportToServer(log)
}

function onPayCreate(res: UTSJSONObject) {
  logPaymentEvent('pay_create', res)
}

function onPaySuccess(res: UTSJSONObject) {
  logPaymentEvent('pay_success', res)
}

function onPayFail(res: UTSJSONObject) {
  logPaymentEvent('pay_fail', res)
}
```

## 七、调试技巧

### 1. 开启调试模式

```vue
<uni-pay 
  ref="uniPayRef"
  :debug="true"
  @success="onPaySuccess"
  @fail="onPayFail"
/>
```

### 2. 查看支付参数

```typescript
function handlePay() {
  const payOptions = buildPaymentParams({
    totalFee: yuanToFen(29.9),
    orderNo: "ORDER_123",
    outTradeNo: generateOutTradeNo(123),
    description: "测试订单"
  })
  
  // 打印支付参数
  console.log('支付参数:', JSON.stringify(payOptions))
  
  const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
  payComponent?.open(payOptions)
}
```

### 3. 模拟支付测试

```typescript
// 开发环境使用小额测试
const isDev = process.env.NODE_ENV === 'development'

const payOptions = {
  provider: "",
  total_fee: isDev ? 1 : yuanToFen(actualAmount), // 开发环境使用0.01元
  order_no: orderNo,
  out_trade_no: outTradeNo,
  description: isDev ? "[测试]订单支付" : "订单支付"
} as UTSJSONObject
```
