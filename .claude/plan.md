# 切换门店时购物车商品转移方案

## 问题分析

### 当前行为
- 用户在门店 A 添加商品到购物车（本地 localStorage）
- 切换到门店 B 时，购物车商品保持不变
- 问题：门店 B 可能没有该商品，或者该商品已售罄

### 后端已有能力
- 购物车服务 `oneUserCarts` 已有**跨门店商品名称匹配逻辑**
- 通过 `goods_name` 匹配不同门店的同名商品
- 返回 `matching_product` 字段标识目标门店的对应商品
- 返回 `is_available` 字段标识商品是否可用

### 前端现状
- 购物车数据存储在本地 localStorage（`cart.uts`）
- 切换门店时仅更新 `appStore.currentStore`，未处理购物车
- 需要新增：切换门店时的购物车校验和转移逻辑

## 实现方案

### 1. 后端：新增批量商品校验接口

**文件**: `E:\Code\store-server-node\src\controller\goodsController.js`

新增 `batchCheckAvailability` 方法，接收商品列表和目标门店 ID，返回每个商品在目标门店的可用状态。

```javascript
async batchCheckAvailability(ctx) {
  try {
    const { items, store_id } = ctx.request.body;
    // items: [{ goods_id, name }]
    
    const Goods = require("../model/product/goods");
    const result = [];
    
    for (const item of items) {
      // 按名称在目标门店查找同名商品
      const match = await Goods.findOne({
        where: {
          goods_name: item.name,
          store_id: store_id,
          status: 1
        }
      });
      
      if (match) {
        result.push({
          original_id: item.goods_id,
          original_name: item.name,
          matched_id: match.id,
          matched_name: match.goods_name,
          available: match.goods_num > 0,
          stock: match.goods_num,
          price: match.goods_price
        });
      } else {
        result.push({
          original_id: item.goods_id,
          original_name: item.name,
          matched_id: null,
          available: false,
          stock: 0,
          reason: '该门店无此商品'
        });
      }
    }
    
    ctx.body = {
      code: 0,
      message: "校验成功",
      result: result
    };
  } catch (error) {
    console.error(error);
    ctx.body = { code: 500, message: error.message };
  }
}
```

**文件**: `E:\Code\store-server-node\src\router\goodsRouter.js`

新增路由：
```javascript
router.post("/batch_check", auth, batchCheckAvailability);
```

### 2. 前端：购物车 Store 新增转移方法

**文件**: `E:\Code\yongge-tea-order\store\cart.uts`

新增 `transferCartToStore` 方法：

```typescript
async transferCartToStore(targetStoreId: number): Promise<TransferResult> {
  if (this.items.length === 0) {
    return { transferred: [], removed: [], updated: [] }
  }
  
  try {
    const res = await request<any>({
      url: '/goods/batch_check',
      method: 'POST',
      data: {
        store_id: targetStoreId,
        items: this.items.map(item => ({
          goods_id: item.id,
          name: item.name
        }))
      }
    })
    
    const checkResult = res.result as CheckResult[]
    const transferred: CartItem[] = []
    const removed: CartItem[] = []
    const updated: CartItem[] = []
    
    for (const item of this.items) {
      const check = checkResult.find(c => c.original_id === item.id)
      
      if (check && check.available && check.matched_id) {
        // 商品在目标门店可用，更新 ID 和价格
        const updatedItem = { ...item }
        if (updatedItem.id !== check.matched_id) {
          updatedItem.id = check.matched_id
          updatedItem.price = check.price
          updatedItem.totalPrice = check.price * updatedItem.quantity
          updated.push(updatedItem)
        }
        transferred.push(updatedItem)
      } else {
        // 商品不可用，移除
        removed.push(item)
      }
    }
    
    // 更新购物车
    this.items = transferred
    this.saveToStorage()
    
    return { transferred, removed, updated }
  } catch (e) {
    console.error('购物车转移失败:', e)
    // 失败时清空购物车，避免脏数据
    this.items = []
    this.saveToStorage()
    return { transferred: [], removed: this.items, updated: [] }
  }
}
```

### 3. 前端：门店切换页面增加确认流程

**文件**: `E:\Code\yongge-tea-order\pages\store\list.uvue`

修改 `selectStore` 方法，切换门店时检查购物车：

```typescript
async function selectStore(store: StoreInfo) {
  // 如果购物车为空，直接切换
  if (cartStore.items.length === 0) {
    appStore.setStore(store)
    uni.navigateBack()
    return
  }
  
  // 购物车不为空，提示用户
  uni.showModal({
    title: '切换门店',
    content: `切换到"${store.name}"后，购物车中部分商品可能不可用，是否继续？`,
    confirmText: '切换',
    cancelText: '取消',
    success: async (res) => {
      if (res.confirm) {
        uni.showLoading({ title: '正在检查商品...' })
        
        const result = await cartStore.transferCartToStore(store.id)
        
        uni.hideLoading()
        
        // 显示转移结果
        if (result.removed.length > 0) {
          const removedNames = result.removed.map(i => i.name).join('、')
          uni.showModal({
            title: '商品变动',
            content: `以下商品在"${store.name}"不可用，已从购物车移除：\n${removedNames}`,
            showCancel: false,
            confirmText: '知道了'
          })
        }
        
        appStore.setStore(store)
        uni.navigateBack()
      }
    }
  })
}
```

## 涉及文件清单

| 文件 | 修改类型 | 说明 |
|------|---------|------|
| `E:\Code\store-server-node\src\controller\goodsController.js` | 新增 | 添加 `batchCheckAvailability` 方法 |
| `E:\Code\store-server-node\src\router\goodsRouter.js` | 修改 | 添加 `/goods/batch_check` 路由 |
| `E:\Code\yongge-tea-order\store\cart.uts` | 修改 | 添加 `transferCartToStore` 方法和类型定义 |
| `E:\Code\yongge-tea-order\pages\store\list.uvue` | 修改 | 切换门店时调用购物车转移逻辑 |

## 业务规则

1. **商品匹配**：按 `goods_name` 精确匹配目标门店的同名商品
2. **库存检查**：目标商品 `goods_num > 0` 才算可用
3. **价格更新**：如果匹配到新商品，使用目标门店的价格
4. **规格保留**：保留原有规格选择（specs, spec_ids）
5. **失败处理**：校验接口失败时清空购物车，避免脏数据

## 用户体验流程

```
用户点击切换门店
    ↓
购物车为空？ → 直接切换
    ↓ 否
弹窗确认："切换到XX门店后，部分商品可能不可用"
    ↓ 用户确认
显示 Loading："正在检查商品..."
    ↓
调用 batchCheck 接口
    ↓
有不可用商品？ → 显示变动提示："以下商品已移除：XXX"
    ↓
切换门店，更新购物车
```
