# uni-pay-x 统一支付配置指南

## 一、前置准备

### 1. 开通 uniCloud 服务

1. 登录 [uniCloud 控制台](https://unicloud.dcloud.net.cn/)
2. 创建服务空间（阿里云或腾讯云）
3. 在 HBuilderX 中关联服务空间：
   - 右键项目根目录
   - 选择"关联云服务空间"
   - 选择你创建的服务空间

### 2. 上传云函数

uni-pay-x 需要上传以下云函数到 uniCloud：

1. 在 HBuilderX 中，右键 `uniCloud/cloudfunctions/uni-pay-co`
2. 选择"上传部署云函数"
3. 等待上传完成

### 3. 初始化数据库

1. 在 HBuilderX 中，右键 `uniCloud/database`
2. 选择"初始化云数据库"
3. 勾选以下表：
   - `uni-pay-orders` - 支付订单表
   - `uni-id-users` - 用户表（如果使用 uni-id）

## 二、支付配置

### 1. 微信支付配置

#### 微信小程序支付

1. 登录 [微信支付商户平台](https://pay.weixin.qq.com/)
2. 获取以下信息：
   - `appId`: 小程序 AppID
   - `mchId`: 商户号
   - `key`: API密钥（32位）
   - `appSecret`: 小程序密钥

3. 在 uniCloud 控制台配置：
   - 进入"云函数/云对象" → "uni-pay-co" → "配置"
   - 添加配置文件 `uni-pay.config.json`：

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

#### 微信 APP 支付

需要额外配置：
- 在微信开放平台注册应用
- 获取 APP 的 AppID 和 AppSecret
- 配置支付回调地址

### 2. 支付宝支付配置

#### 支付宝小程序支付

1. 登录 [支付宝开放平台](https://open.alipay.com/)
2. 创建小程序应用
3. 开通"手机网站支付"或"小程序支付"产品
4. 获取以下信息：
   - `appId`: 应用 AppID
   - `privateKey`: 应用私钥
   - `alipayPublicKey`: 支付宝公钥

5. 在 uniCloud 控制台配置：

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

### 3. 完整配置示例

在 uniCloud 控制台，`uni-pay-co` 云对象的配置文件：

```json
{
  "wxpay": {
    "enable": true,
    "appId": "wx1234567890abcdef",
    "mchId": "1234567890",
    "key": "your32characterAPIkeyhere12345",
    "appSecret": "your_app_secret_here",
    "notifyUrl": "https://your-domain.com/notify/wxpay"
  },
  "alipay": {
    "enable": true,
    "appId": "2021001234567890",
    "privateKey": "MIIEvQIBADANBgkqhkiG9w0BAQE...",
    "alipayPublicKey": "MIIBIjANBgkqhkiG9w0BAQEFAA...",
    "notifyUrl": "https://your-domain.com/notify/alipay"
  }
}
```

## 三、支付回调配置

### 1. 配置支付回调地址

在 `uniCloud/cloudfunctions/uni-pay-co/notify/` 目录下：

- `wxpay.js` - 微信支付回调
- `alipay.js` - 支付宝支付回调

### 2. 回调处理逻辑

支付成功后，uni-pay-x 会自动调用回调函数，你需要在回调中：

1. 验证支付结果
2. 更新业务订单状态
3. 发放商品或服务
4. 返回处理结果

示例回调代码（在云函数中）：

```javascript
// uniCloud/cloudfunctions/uni-pay-co/notify/wxpay.js
module.exports = async (obj) => {
  const { data } = obj
  
  // 1. 验证签名（uni-pay-x 已自动验证）
  
  // 2. 获取订单信息
  const outTradeNo = data.out_trade_no
  const transactionId = data.transaction_id
  
  // 3. 更新业务订单状态
  const db = uniCloud.database()
  await db.collection('orders').where({
    out_trade_no: outTradeNo
  }).update({
    status: 2, // 已支付
    pay_time: Date.now(),
    transaction_id: transactionId
  })
  
  // 4. 返回成功
  return {
    errCode: 0,
    errMsg: 'success'
  }
}
```

## 四、测试支付

### 1. 沙箱环境测试

**微信支付沙箱：**
- 微信支付不提供公开沙箱，需要使用真实商户号
- 可以使用 0.01 元小额测试

**支付宝沙箱：**
1. 登录支付宝开放平台
2. 进入"开发者中心" → "研发服务" → "沙箱环境"
3. 获取沙箱 AppID 和密钥
4. 下载"支付宝沙箱版"APP 进行测试

### 2. 测试流程

1. 创建测试订单
2. 发起支付（金额设置为 0.01 元）
3. 使用测试账号完成支付
4. 验证订单状态更新
5. 验证回调是否正常执行

## 五、常见问题

### 1. 支付失败：签名错误

**原因：** API密钥配置错误

**解决：**
- 检查 `uni-pay.config.json` 中的 `key` 是否正确
- 微信支付密钥是32位字符串
- 支付宝需要使用正确的私钥格式

### 2. 支付成功但订单未更新

**原因：** 回调地址配置错误或回调处理失败

**解决：**
- 检查 `notifyUrl` 配置是否正确
- 查看云函数日志，确认回调是否执行
- 确保回调函数返回正确的格式

### 3. 小程序支付提示"未配置支付"

**原因：** 小程序未关联商户号

**解决：**
- 微信小程序：在小程序后台 → 微信支付 → 关联商户号
- 支付宝小程序：在应用详情 → 开发设置 → 添加功能

### 4. APP 支付无法调起支付

**原因：** 未配置 APP 支付参数或签名错误

**解决：**
- 检查 manifest.json 中的支付配置
- 确保 APP 已在开放平台注册
- 检查包名/Bundle ID 是否一致

## 六、安全建议

1. **密钥安全：** 不要将支付密钥提交到代码仓库
2. **金额验证：** 在服务端验证支付金额，防止篡改
3. **订单验证：** 支付前验证订单状态，防止重复支付
4. **回调验证：** 验证回调签名，防止伪造回调
5. **HTTPS：** 生产环境必须使用 HTTPS

## 七、相关文档

- [uni-pay-x 官方文档](https://doc.dcloud.net.cn/uniCloud/uni-pay/uni-app-x.html)
- [微信支付开发文档](https://pay.weixin.qq.com/wiki/doc/api/index.html)
- [支付宝开放平台文档](https://opendocs.alipay.com/mini/introduce)
- [uniCloud 云函数文档](https://doc.dcloud.net.cn/uniCloud/cf-functions.html)
