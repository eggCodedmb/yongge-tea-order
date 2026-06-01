# uni-pay-x 支付集成完成说明

## 集成概述

本项目已成功集成 **uni-pay-x** 统一支付接口，支持微信支付和支付宝支付。

## 已完成的工作

### 1. ✅ 页面集成

**文件：** `pages/checkout/checkout.uvue`

- 添加了 `<uni-pay>` 支付组件
- 实现了完整的支付流程（创建订单 → 发起支付 → 处理回调）
- 集成了支付成功、失败、取消的回调处理
- 支付成功后自动清空购物车并跳转到订单列表

**主要功能：**
```typescript
// 发起支付
function handlePay() {
  // 1. 创建业务订单
  // 2. 构建支付参数
  // 3. 调用 uni-pay 组件发起支付
}

// 支付成功回调
function onPaySuccess(res: UTSJSONObject) {
  // 1. 通知后端更新订单状态
  // 2. 清空购物车
  // 3. 跳转到订单列表
}
```

### 2. ✅ 工具函数库

**文件：** `utils/payment.uts`

提供了完整的支付工具函数：

| 函数名 | 功能 | 示例 |
|--------|------|------|
| `yuanToFen()` | 元转分 | `yuanToFen(29.9)` → `2990` |
| `fenToYuan()` | 分转元 | `fenToYuan(2990)` → `"29.90"` |
| `generateOutTradeNo()` | 生成支付订单号 | `generateOutTradeNo(123)` |
| `generateOrderNo()` | 生成业务订单号 | `generateOrderNo(123)` |
| `buildPaymentParams()` | 构建支付参数 | 返回标准支付参数对象 |
| `validatePaymentAmount()` | 验证支付金额 | 检查金额是否有效 |
| `formatPaymentError()` | 格式化错误信息 | 返回用户友好的错误提示 |
| `getSupportedProviders()` | 获取支持的支付方式 | 返回当前平台支持的支付方式数组 |
| `getDefaultProvider()` | 获取默认支付方式 | 根据平台返回默认支付方式 |

### 3. ✅ 文档完善

创建了以下文档文件：

| 文档 | 路径 | 说明 |
|------|------|------|
| 快速开始指南 | `docs/PAYMENT_QUICKSTART.md` | 5分钟快速上手教程 |
| 配置指南 | `docs/UNI_PAY_X_SETUP.md` | uniCloud 和支付参数配置详解 |
| 使用示例 | `docs/PAYMENT_USAGE_EXAMPLES.md` | 完整的代码示例和最佳实践 |
| 集成说明 | `docs/PAYMENT_INTEGRATION_README.md` | 本文档 |

### 4. ✅ 项目文档更新

**文件：** `CLAUDE.md`

更新了项目文档，添加了 uni-pay-x 的使用说明和注意事项。

## 使用方法

### 基础用法

```vue
<template>
  <view>
    <button @click="handlePay">立即支付</button>
    
    <uni-pay 
      ref="uniPayRef"
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
  const payOptions = buildPaymentParams({
    provider: "", // 留空显示支付方式选择
    totalFee: yuanToFen(29.9),
    orderNo: "ORDER_123",
    outTradeNo: generateOutTradeNo(123),
    description: "订单支付"
  })
  
  const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
  payComponent?.open(payOptions)
}

function onPaySuccess(res: UTSJSONObject) {
  uni.showToast({ title: '支付成功', icon: 'success' })
}

function onPayFail(res: UTSJSONObject) {
  uni.showToast({ title: '支付失败', icon: 'none' })
}

function onPayCancel(res: any) {
  uni.showToast({ title: '已取消支付', icon: 'none' })
}
</script>
```

## 后续配置步骤

### ⚠️ 必须完成的配置

在使用支付功能前，你需要完成以下配置：

#### 1. 开通 uniCloud 服务

1. 登录 [uniCloud 控制台](https://unicloud.dcloud.net.cn/)
2. 创建服务空间（阿里云或腾讯云）
3. 在 HBuilderX 中关联服务空间

#### 2. 上传云函数

1. 右键 `uniCloud/cloudfunctions/uni-pay-co`
2. 选择"上传部署云函数"
3. 等待上传完成

#### 3. 配置支付参数

在 uniCloud 控制台配置支付参数：

**微信支付配置：**
```json
{
  "wxpay": {
    "enable": true,
    "appId": "你的小程序AppID",
    "mchId": "你的商户号",
    "key": "你的API密钥",
    "appSecret": "你的小程序密钥"
  }
}
```

**支付宝支付配置：**
```json
{
  "alipay": {
    "enable": true,
    "appId": "你的支付宝AppID",
    "privateKey": "你的应用私钥",
    "alipayPublicKey": "支付宝公钥"
  }
}
```

详细配置步骤请查看：`docs/UNI_PAY_X_SETUP.md`

#### 4. 后端接口调整

你需要在后端添加或修改以下接口：

**支付成功回调接口：**
```
POST /order/pay_success
参数：
- order_id: 订单ID
- out_trade_no: 支付订单号
- transaction_id: 第三方交易号
```

这个接口用于在支付成功后更新订单状态。

## 关键注意事项

### 1. 金额单位

⚠️ **重要：** uni-pay-x 使用 **分（fen）** 作为金额单位，不是元！

```typescript
// ❌ 错误
total_fee: 29.9

// ✅ 正确
total_fee: yuanToFen(29.9) // 2990
```

### 2. 订单号唯一性

`out_trade_no` 必须全局唯一，建议使用提供的工具函数生成：

```typescript
const outTradeNo = generateOutTradeNo(orderId)
// 生成格式：PAY_时间戳_订单ID_随机数
```

### 3. 平台差异

不同平台支持的支付方式不同：

| 平台 | 支持的支付方式 | provider 参数 |
|------|---------------|--------------|
| 微信小程序 | 仅微信支付 | `"wxpay"` |
| 支付宝小程序 | 仅支付宝支付 | `"alipay"` |
| APP / H5 | 微信、支付宝 | `""` (显示选择) |

使用 `getDefaultProvider()` 可以自动适配当前平台。

### 4. 错误处理

始终实现完整的错误处理：

```typescript
function onPayFail(res: UTSJSONObject) {
  const errCode = res.getNumber('errCode') || -1
  const errMsg = res.getString('errMsg') || ''
  
  // 使用工具函数格式化错误信息
  const friendlyMsg = formatPaymentError(errCode, errMsg)
  
  uni.showToast({ title: friendlyMsg, icon: 'none' })
  
  // 记录错误日志
  console.error('支付失败:', { errCode, errMsg })
}
```

## 测试建议

### 开发环境测试

1. **使用小额测试：** 设置支付金额为 0.01 元
2. **添加测试标识：** 在订单描述中添加 `[测试]` 前缀
3. **查看日志：** 开启 debug 模式查看详细日志

```typescript
const isDev = true // 根据环境判断

const payOptions = {
  total_fee: isDev ? 1 : yuanToFen(actualAmount), // 0.01元测试
  description: isDev ? "[测试]订单支付" : "订单支付",
  // ...
}
```

### 支付宝沙箱测试

支付宝提供沙箱环境：
1. 登录支付宝开放平台
2. 进入"沙箱环境"获取沙箱配置
3. 下载"支付宝沙箱版"APP 进行测试

### 微信支付测试

微信不提供公开沙箱，需要：
1. 使用真实商户号
2. 使用 0.01 元小额测试
3. 测试完成后可以申请退款

## 项目结构

```
E:\Code\点物\
├── pages/
│   └── checkout/
│       └── checkout.uvue          # ✅ 已集成支付功能
├── utils/
│   └── payment.uts                # ✅ 支付工具函数库
├── docs/
│   ├── PAYMENT_QUICKSTART.md      # ✅ 快速开始指南
│   ├── UNI_PAY_X_SETUP.md         # ✅ 配置指南
│   ├── PAYMENT_USAGE_EXAMPLES.md  # ✅ 使用示例
│   └── PAYMENT_INTEGRATION_README.md  # ✅ 本文档
├── uni_modules/
│   └── uni-pay-x/                 # ✅ uni-pay-x 插件
└── CLAUDE.md                      # ✅ 已更新项目文档
```

## 常见问题

### Q: 支付时提示"未配置支付"？

**A:** 检查以下几点：
1. 是否已上传 `uni-pay-co` 云函数
2. 是否在 uniCloud 控制台配置了支付参数
3. 小程序是否已关联商户号

### Q: 支付成功但订单状态未更新？

**A:** 检查：
1. 后端 `/order/pay_success` 接口是否正常
2. 查看云函数日志，确认回调是否执行
3. 检查 `custom` 字段中的 `order_id` 是否正确传递

### Q: 如何查看支付日志？

**A:** 
1. 在组件上添加 `:debug="true"` 开启调试模式
2. 查看浏览器控制台或 HBuilderX 控制台
3. 在 uniCloud 控制台查看云函数日志

### Q: 支付金额显示不正确？

**A:** 检查是否正确使用了 `yuanToFen()` 转换：
```typescript
// ✅ 正确
total_fee: yuanToFen(29.9) // 2990分

// ❌ 错误
total_fee: 29.9 // 会被当作 0.299 元
```

## 技术支持

- **uni-pay-x 官方文档：** https://doc.dcloud.net.cn/uniCloud/uni-pay/uni-app-x.html
- **DCloud 社区：** https://ask.dcloud.net.cn/
- **微信支付文档：** https://pay.weixin.qq.com/wiki/doc/api/index.html
- **支付宝开放平台：** https://opendocs.alipay.com/

## 更新日志

### 2026-05-31
- ✅ 完成 uni-pay-x 集成
- ✅ 创建支付工具函数库
- ✅ 更新 checkout 页面支付逻辑
- ✅ 编写完整的配置和使用文档
- ✅ 更新项目 CLAUDE.md 文档

## 下一步建议

1. **完成 uniCloud 配置：** 按照 `docs/UNI_PAY_X_SETUP.md` 完成配置
2. **测试支付流程：** 使用 0.01 元进行完整流程测试
3. **实现退款功能：** 参考 `docs/PAYMENT_USAGE_EXAMPLES.md` 中的退款示例
4. **添加支付记录：** 在订单详情页显示支付信息
5. **优化用户体验：** 添加支付状态轮询、超时处理等

---

**集成完成！** 🎉

如有问题，请查看相关文档或联系技术支持。
