# uni-pay-x 集成验证清单

## 集成验证清单

使用此清单验证 uni-pay-x 是否正确集成到项目中。

### ✅ 代码集成检查

- [x] **checkout.uvue 页面已更新**
  - [x] 添加了 `<uni-pay>` 组件
  - [x] 导入了支付工具函数
  - [x] 实现了 `handlePay()` 支付逻辑
  - [x] 实现了 `onPaySuccess()` 成功回调
  - [x] 实现了 `onPayFail()` 失败回调
  - [x] 实现了 `onPayCancel()` 取消回调

- [x] **支付工具函数库已创建**
  - [x] `utils/payment.uts` 文件存在
  - [x] 包含金额转换函数
  - [x] 包含订单号生成函数
  - [x] 包含参数构建函数
  - [x] 包含验证和错误处理函数

- [x] **文档已完善**
  - [x] 快速开始指南
  - [x] 配置指南
  - [x] 使用示例文档
  - [x] 集成说明文档
  - [x] CLAUDE.md 已更新

### ⚠️ 待完成配置

- [ ] **uniCloud 服务配置**
  - [ ] 已开通 uniCloud 服务空间
  - [ ] 已在 HBuilderX 中关联服务空间
  - [ ] 已上传 `uni-pay-co` 云函数
  - [ ] 已初始化数据库表

- [ ] **支付参数配置**
  - [ ] 微信支付参数已配置（如需要）
    - [ ] appId
    - [ ] mchId
    - [ ] key
    - [ ] appSecret
  - [ ] 支付宝支付参数已配置（如需要）
    - [ ] appId
    - [ ] privateKey
    - [ ] alipayPublicKey

- [ ] **后端接口调整**
  - [ ] `/order/pay_success` 接口已实现
  - [ ] 支付回调逻辑已实现
  - [ ] 订单状态更新逻辑已实现

- [ ] **测试验证**
  - [ ] 已完成支付流程测试
  - [ ] 已测试支付成功场景
  - [ ] 已测试支付失败场景
  - [ ] 已测试取消支付场景
  - [ ] 已测试订单状态更新

## 快速验证步骤

### 1. 验证代码集成

```bash
# 检查文件是否存在
ls pages/checkout/checkout.uvue
ls utils/payment.uts
ls docs/PAYMENT_*.md
```

### 2. 验证组件引用

在 `pages/checkout/checkout.uvue` 中搜索：
- `<uni-pay` - 组件是否存在
- `import { yuanToFen` - 工具函数是否导入
- `function handlePay()` - 支付函数是否实现

### 3. 验证 uni-pay-x 插件

```bash
# 检查插件是否存在
ls uni_modules/uni-pay-x/
```

### 4. 测试支付流程（需要完成配置后）

1. 启动项目
2. 添加商品到购物车
3. 进入结账页面
4. 点击"立即支付"按钮
5. 选择支付方式
6. 完成支付
7. 验证订单状态是否更新

## 常见问题排查

### 问题 1: 点击支付按钮无反应

**排查步骤：**
1. 检查控制台是否有错误
2. 确认 `uniPayRef` 是否正确引用
3. 确认 `getCurrentInstance()` 是否正常工作
4. 检查 `payComponent?.open` 是否被调用

**解决方案：**
```typescript
// 添加调试日志
function handlePay() {
  console.log('开始支付')
  const payComponent = instance?.proxy?.$refs['uniPayRef'] as any
  console.log('支付组件:', payComponent)
  
  if (!payComponent) {
    console.error('支付组件未找到')
    return
  }
  
  payComponent.open(payOptions)
}
```

### 问题 2: 支付金额显示错误

**原因：** 未使用 `yuanToFen()` 转换

**解决方案：**
```typescript
// ❌ 错误
total_fee: 29.9

// ✅ 正确
total_fee: yuanToFen(29.9)
```

### 问题 3: 支付成功但订单未更新

**排查步骤：**
1. 检查 `onPaySuccess` 是否被调用
2. 检查后端接口是否正常
3. 检查 `custom` 字段中的数据是否正确

**解决方案：**
```typescript
function onPaySuccess(res: UTSJSONObject) {
  console.log('支付成功回调:', res)
  console.log('订单ID:', res.getJSON('custom')?.getNumber('order_id'))
  
  // 确保订单ID正确传递
  const orderId = res.getJSON('custom')?.getNumber('order_id')
  if (!orderId) {
    console.error('订单ID缺失')
    return
  }
  
  // 调用后端接口
}
```

### 问题 4: uniCloud 未配置

**错误提示：** "请先初始化 uniCloud"

**解决方案：**
1. 登录 uniCloud 控制台
2. 创建服务空间
3. 在 HBuilderX 中关联服务空间
4. 上传云函数

### 问题 5: 支付参数未配置

**错误提示：** "支付配置不存在"

**解决方案：**
1. 进入 uniCloud 控制台
2. 找到 `uni-pay-co` 云对象
3. 添加配置文件 `uni-pay.config.json`
4. 填写支付参数

## 性能优化建议

### 1. 支付组件懒加载

```typescript
// 仅在需要时加载支付组件
const showPayComponent = ref(false)

function handlePay() {
  showPayComponent.value = true
  nextTick(() => {
    // 调用支付
  })
}
```

### 2. 支付参数缓存

```typescript
// 缓存支付参数，避免重复构建
const cachedPayOptions = ref<UTSJSONObject | null>(null)

function buildPayOptions() {
  if (cachedPayOptions.value) {
    return cachedPayOptions.value
  }
  
  cachedPayOptions.value = buildPaymentParams({
    // ...
  })
  
  return cachedPayOptions.value
}
```

### 3. 错误重试机制

```typescript
let retryCount = 0
const MAX_RETRY = 3

async function handlePayWithRetry() {
  try {
    await handlePay()
  } catch (err) {
    if (retryCount < MAX_RETRY) {
      retryCount++
      console.log(`支付失败，重试第 ${retryCount} 次`)
      setTimeout(() => handlePayWithRetry(), 1000)
    } else {
      uni.showToast({ title: '支付失败，请稍后重试', icon: 'none' })
    }
  }
}
```

## 安全检查清单

- [ ] **密钥安全**
  - [ ] 支付密钥未提交到代码仓库
  - [ ] 使用环境变量或云端配置存储密钥
  - [ ] 定期更换支付密钥

- [ ] **金额验证**
  - [ ] 前端验证支付金额范围
  - [ ] 后端验证支付金额一致性
  - [ ] 防止金额篡改

- [ ] **订单验证**
  - [ ] 验证订单状态，防止重复支付
  - [ ] 验证订单归属，防止越权支付
  - [ ] 验证订单金额，防止金额不一致

- [ ] **回调验证**
  - [ ] 验证回调签名
  - [ ] 验证回调来源
  - [ ] 防止伪造回调

- [ ] **HTTPS**
  - [ ] 生产环境使用 HTTPS
  - [ ] 支付回调地址使用 HTTPS

## 上线前检查

### 必须完成项

- [ ] 所有支付参数已配置为生产环境参数
- [ ] 已完成完整的支付流程测试
- [ ] 已测试支付成功、失败、取消等场景
- [ ] 已测试订单状态更新逻辑
- [ ] 已测试支付回调逻辑
- [ ] 已移除所有测试代码和调试日志
- [ ] 已配置生产环境的回调地址
- [ ] 已完成安全检查

### 建议完成项

- [ ] 添加支付日志记录
- [ ] 添加支付异常监控
- [ ] 添加支付数据统计
- [ ] 优化支付用户体验
- [ ] 添加支付帮助文档

## 监控指标

上线后建议监控以下指标：

1. **支付成功率**
   - 目标：> 95%
   - 计算：支付成功次数 / 发起支付次数

2. **支付失败率**
   - 目标：< 5%
   - 按错误类型分类统计

3. **支付取消率**
   - 目标：< 20%
   - 分析用户取消原因

4. **支付响应时间**
   - 目标：< 3秒
   - 从点击支付到调起支付的时间

5. **订单状态更新成功率**
   - 目标：100%
   - 支付成功后订单状态是否正确更新

## 联系支持

如果遇到问题，可以通过以下方式获取帮助：

1. **查看文档**
   - `docs/PAYMENT_QUICKSTART.md` - 快速开始
   - `docs/UNI_PAY_X_SETUP.md` - 配置指南
   - `docs/PAYMENT_USAGE_EXAMPLES.md` - 使用示例

2. **官方文档**
   - uni-pay-x: https://doc.dcloud.net.cn/uniCloud/uni-pay/uni-app-x.html
   - DCloud 社区: https://ask.dcloud.net.cn/

3. **技术支持**
   - 微信支付: https://pay.weixin.qq.com/
   - 支付宝: https://opendocs.alipay.com/

---

**验证完成后，请在对应项前打勾 ✓**
