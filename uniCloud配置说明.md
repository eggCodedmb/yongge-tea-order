# uniCloud 配置说明

## 已完成的配置

✅ 删除了旧版 `uni-pay` 插件（解决云函数名称冲突）
✅ 创建了 uni-id 配置文件
✅ 创建了 uni-pay 配置文件
✅ 复制了必要的云函数和公共模块到 uniCloud-alipay 目录

## 目录结构

```
uniCloud-alipay/
├── cloudfunctions/
│   ├── common/
│   │   ├── uni-config-center/     # 配置中心
│   │   │   ├── uni-id/
│   │   │   │   └── config.json    # uni-id 配置
│   │   │   └── uni-pay/
│   │   │       └── config.json    # uni-pay 配置
│   │   ├── uni-id-common/         # uni-id 公共模块
│   │   └── uni-pay/               # uni-pay 公共模块
│   └── uni-pay-co/                # uni-pay 云函数
└── database/
```

## 需要在 HBuilderX 中完成的操作

### 1. 关联云服务空间

1. 右键点击 `uniCloud-alipay` 目录
2. 选择 "关联云服务空间或项目"
3. 选择你的支付宝云服务空间（如果没有，需要先创建）

### 2. 上传云函数

1. 右键点击 `uniCloud-alipay/cloudfunctions/uni-pay-co` 目录
2. 选择 "上传部署云函数"
3. 等待上传完成

### 3. 上传公共模块

1. 右键点击 `uniCloud-alipay/cloudfunctions/common/uni-config-center`
2. 选择 "上传公共模块"
3. 对以下模块重复此操作：
   - `uni-id-common`
   - `uni-pay`

### 4. 初始化数据库

1. 右键点击 `uniCloud-alipay/database` 目录
2. 选择 "初始化云数据库"
3. 这会创建 uni-pay 需要的数据表（uni-pay-orders 等）

### 5. 配置支付参数（重要！）

需要修改以下配置文件，填入真实的支付参数：

#### uni-id 配置 (`uniCloud-alipay/cloudfunctions/common/uni-config-center/uni-id/config.json`)

```json
{
  "passwordSecret": "请修改为随机字符串（至少16位）",
  "tokenSecret": "请修改为随机字符串（至少16位）",
  ...
}
```

⚠️ **安全提示**：`passwordSecret` 和 `tokenSecret` 必须修改为随机字符串，不要使用示例值！

#### uni-pay 配置 (`uniCloud-alipay/cloudfunctions/common/uni-config-center/uni-pay/config.json`)

**微信支付配置：**
```json
{
  "wxpay": {
    "enable": true,
    "mchId": "你的微信商户号",
    "key": "你的微信商户密钥",
    "appId": "wx764f09d8c31f970a",
    ...
  }
}
```

**支付宝配置：**
```json
{
  "alipay": {
    "enable": true,
    "appId": "你的支付宝应用ID",
    "privateKey": "你的支付宝应用私钥",
    "publicKey": "支付宝公钥",
    ...
  }
}
```

### 6. 重新上传配置

修改配置文件后，需要重新上传 `uni-config-center` 公共模块：

1. 右键点击 `uniCloud-alipay/cloudfunctions/common/uni-config-center`
2. 选择 "上传公共模块"

### 7. 测试支付

完成以上步骤后，重新运行项目测试支付功能。

## 常见问题

### Q: 如何获取支付宝/微信支付的配置参数？

**微信支付：**
- 登录 [微信支付商户平台](https://pay.weixin.qq.com/)
- 在"账户中心 > API安全"中获取商户号和密钥

**支付宝：**
- 登录 [支付宝开放平台](https://open.alipay.com/)
- 在"开发设置"中配置应用并获取密钥

### Q: 云函数上传失败怎么办？

1. 检查是否已关联云服务空间
2. 检查网络连接
3. 查看 HBuilderX 控制台的错误信息

### Q: 支付时仍然报错怎么办？

1. 确认云函数已成功上传
2. 确认配置文件中的参数已正确填写
3. 在 uniCloud web 控制台查看云函数日志
4. 检查支付金额是否正确（必须大于0）

## 参考文档

- [uni-pay-x 官方文档](https://doc.dcloud.net.cn/uniCloud/uni-pay/uni-app-x.html)
- [uniCloud 云函数文档](https://doc.dcloud.net.cn/uniCloud/cf-functions.html)
- [uni-id 配置文档](https://doc.dcloud.net.cn/uniCloud/uni-id-summary.html)
