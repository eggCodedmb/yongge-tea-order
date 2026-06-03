# 创建订单前库存校验方案

## 问题分析

### 当前流程问题

1. **后端错误信息丢失**：`orderService.createOrder` (line 45) 捕获了 `productInventory` 抛出的 "库存不足" 错误，但重新抛出为通用的 "创建订单失败"，前端无法得知具体原因
2. **前端忽略错误**：`checkout.uvue` 的 `onMockPaySuccess` 方法在 catch 块中仅打印日志，仍然显示 "支付成功" toast
3. **无预检查机制**：用户点击支付后才在事务中检查库存，用户体验差

## 修改方案

### 1. 后端：`orderService.js` - 保留原始错误信息

**文件**: `E:\Code\store-server-node\src\service\orderService.js`

**修改内容**: 在 `createOrder` 方法的 catch 块中，保留原始错误信息而非覆盖

```javascript
// 当前 (line 42-46):
catch (error) {
  await transaction.rollback();
  console.error("创建订单失败:", error);
  throw new Error("创建订单失败");
}

// 修改为:
catch (error) {
  await transaction.rollback();
  console.error("创建订单失败:", error);
  throw error; // 保留原始错误信息（如 "库存不足"、"商品不存在"）
}
```

### 2. 后端：新增库存预检查接口

**文件**: `E:\Code\store-server-node\src\controller\orderController.js`

**新增方法**: `checkStock` - 在创建订单前检查库存

```javascript
async checkStock(ctx) {
  try {
    const { items } = ctx.request.body;
    if (!items || items.length === 0) {
      throw new Error("商品列表不能为空");
    }
    
    const Goods = require("../model/product/goods");
    const result = [];
    
    for (const item of items) {
      const goods = await Goods.findByPk(item.goods_id || item.id);
      if (!goods) {
        result.push({
          goods_id: item.goods_id || item.id,
          name: item.name || '未知商品',
          available: false,
          reason: '商品不存在',
          stock: 0
        });
      } else if (goods.goods_num < item.quantity) {
        result.push({
          goods_id: goods.id,
          name: goods.goods_name,
          available: false,
          reason: '库存不足',
          stock: goods.goods_num
        });
      } else {
        result.push({
          goods_id: goods.id,
          name: goods.goods_name,
          available: true,
          stock: goods.goods_num
        });
      }
    }
    
    const allAvailable = result.every(r => r.available);
    
    ctx.body = {
      code: 0,
      message: allAvailable ? "库存充足" : "部分商品库存不足",
      result: {
        all_available: allAvailable,
        items: result
      }
    };
  } catch (error) {
    console.error(error);
    ctx.body = { code: 500, message: error.message };
  }
}
```

**文件**: `E:\Code\store-server-node\src\router\orderRouter.js`

**新增路由**: 在路由文件中添加 `check_stock` 路由

```javascript
router.post("/check_stock", auth, checkStock);
```

### 3. 前端：`checkout.uvue` - 添加库存校验逻辑

**文件**: `E:\Code\yongge-tea-order\pages\checkout\checkout.uvue`

**修改内容**:

1. **新增 `checkStock` 方法** - 在支付前调用后端接口检查库存

```typescript
async function checkStock(): Promise<boolean> {
  const selectedItems = cartStore.items.filter(i => i.selected)
  if (selectedItems.length === 0) return false
  
  try {
    const res = await request<any>({
      url: '/order/check_stock',
      method: 'POST',
      data: {
        items: selectedItems.map(item => ({
          goods_id: item.id,
          quantity: item.quantity,
          name: item.name
        }))
      }
    })
    
    if (!res.result.all_available) {
      // 找出库存不足的商品
      const unavailableItems = res.result.items.filter((i: any) => !i.available)
      const itemNames = unavailableItems.map((i: any) => `${i.name}(库存:${i.stock})`).join('、')
      
      uni.showModal({
        title: '库存不足',
        content: `以下商品库存不足：${itemNames}，请修改数量后再试`,
        showCancel: false,
        confirmText: '知道了'
      })
      return false
    }
    
    return true
  } catch (e) {
    console.error('库存检查失败:', e)
    return true // 检查接口失败时不阻塞，让创建订单接口来判断
  }
}
```

2. **修改 `handlePay` 方法** - 在支付前调用库存检查

```typescript
async function handlePay() {
  // ... 现有的前置检查 ...
  
  // 新增：库存检查
  const stockOk = await checkStock()
  if (!stockOk) return
  
  // ... 继续现有的模拟支付流程 ...
}
```

3. **修改 `onMockPaySuccess` 方法** - 正确处理库存错误

```typescript
async function onMockPaySuccess() {
  uni.showLoading({ title: '支付处理中...' })
  
  try {
    const createRes = await request<any>({
      url: '/order/create_new',
      method: 'POST',
      data: {
        items: cartStore.items.filter(i => i.selected),
        order_type: appStore.orderMode === 'self-pickup' ? 1 : 2,
        address_id: appStore.orderMode === 'delivery' ? addressStore.selectedAddress?.id : 0,
        remark: remark.value
      }
    })
    
    // ... 成功逻辑 ...
    
  } catch (e: any) {
    uni.hideLoading()
    console.error('订单创建失败:', e)
    
    // 判断是否是库存不足错误
    const errorMsg = e.message || '订单创建失败'
    if (errorMsg.includes('库存不足')) {
      uni.showModal({
        title: '库存不足',
        content: '商品库存不足，请返回购物车修改数量',
        confirmText: '返回购物车',
        success: (res) => {
          if (res.confirm) {
            uni.navigateBack()
          }
        }
      })
    } else {
      uni.showToast({ title: errorMsg, icon: 'none' })
    }
  }
}
```

## 涉及文件清单

| 文件 | 修改类型 | 说明 |
|------|---------|------|
| `E:\Code\store-server-node\src\service\orderService.js` | 修改 | 保留原始错误信息 |
| `E:\Code\store-server-node\src\controller\orderController.js` | 新增 | 添加 `checkStock` 方法 |
| `E:\Code\store-server-node\src\router\orderRouter.js` | 修改 | 添加 `/check_stock` 路由 |
| `E:\Code\yongge-tea-order\pages\checkout\checkout.uvue` | 修改 | 添加库存检查和错误处理 |

## 测试要点

1. 后端：库存不足时，`/order/create_new` 返回的错误信息应包含 "库存不足"
2. 后端：`/order/check_stock` 接口正确返回各商品的库存状态
3. 前端：点击支付时，库存不足应弹窗提示具体哪些商品库存不足
4. 前端：即使绕过预检查，创建订单失败时也能正确提示库存不足
