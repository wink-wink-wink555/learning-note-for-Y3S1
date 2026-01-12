# 核心数据表详解：stock_in (入库表)

本文档详细解析 `stock_in` 表的结构、字段含义，以及它在前后端业务流程中的具体应用。

## 一、 数据表结构解析

`stock_in` 表记录了每一次教材入库的详细流水。它是连接“采购订单”和“库存”的桥梁。

| 字段名 | 类型 | 含义 | 备注 |
| :--- | :--- | :--- | :--- |
| **stock_in_id** | INT | 入库ID | 主键，自增 |
| **stock_in_no** | VARCHAR(50) | 入库单号 | 唯一，格式如 `SI202401010001`，由存储过程生成 |
| **order_id** | INT | 订单ID | 外键，关联 `purchase_order` 表，表明这批货属于哪个订单 |
| **textbook_id** | INT | 教材ID | 外键，关联 `textbook` 表 |
| **stock_in_quantity** | INT | 入库数量 | 填写的单据数量 |
| **actual_quantity** | INT | **实际数量** | **核心字段**。触发器根据此字段更新库存。通常与入库数量一致 |
| **stock_in_date** | DATE | 入库日期 | 业务发生日期 |
| **warehouse_person**| VARCHAR(50) | 经办人 | 记录是谁操作入库的（通常是仓库管理员账号） |
| **quality_status** | ENUM | 质量状态 | '合格', '部分合格', '不合格' |
| **remarks** | TEXT | 备注 | 选填 |

---

## 二、 前端页面应用

### 1. 页面位置
目前主要在 **订单管理页面 (`orders.html`)** 中使用。

### 2. 用户操作流程
1.  **入口**：管理员在订单列表中，找到状态为“已审核”或“部分到货”的订单。
2.  **点击**：点击右侧绿色的 **“入库”** 按钮。
3.  **弹窗**：系统弹出 `stockInModal` (入库模态框)。
    *   系统自动回显：订单号、教材名称、剩余可入库数量。
    *   用户填写：**本次入库数量**、入库日期、质量状态。
4.  **提交**：点击“确认入库”。

### 3. 关键 JS 代码 (`orders.html`)
```javascript
// 提交入库
async function submitStockIn() {
    // 1. 获取用户填写的表单数据
    const data = {
        order_id: ...,
        textbook_id: ...,
        stock_in_quantity: ...,  // 用户填的数量
        actual_quantity: ...,    // 目前前端逻辑将实际数量设为与入库数量一致
        warehouse_person: user.username, // 当前登录用户
        // ...
    };
    
    // 2. 调用 API
    const result = await api.createStockIn(data);
}
```

---

## 三、 后端调用流程

### 1. API 层 (`app/api/v1/stock_in.py`)
接收前端请求，进行权限验证（`@warehouse_required`，确保只有仓库管理员能操作）。

```python
@api_v1.route('/stock-ins', methods=['POST'])
@warehouse_required
def create_stock_in():
    # ... 校验数据 ...
    result = stock_in_service.create_stock_in(data)
    return success_response(message='入库成功，库存已自动更新')
```

### 2. Service 层 (`app/services/stock_in_service.py`)
负责业务逻辑校验和单号生成。

```python
def create_stock_in(self, data):
    # 1. 检查订单状态（已取消的不能入库）
    # 2. 检查数量（不能超过订单剩余未到货数量）
    
    # 3. 生成入库单号 (调用存储过程 sp_generate_stock_in_no)
    data['stock_in_no'] = self.generate_stock_in_no()
    
    # 4. 保存到数据库
    stock_in = self.stock_in_dao.create(data)
    return stock_in
```

### 3. 数据库触发器 (自动化核心)
当 Service 层执行 `create` (INSERT) 操作后，数据库触发器 `trg_stock_in_after_insert` 立即启动：
1.  读取 `actual_quantity`。
2.  **更新库存**：`inventory.current_quantity += actual_quantity`。
3.  **更新订单**：`purchase_order.arrived_quantity += actual_quantity`。

---

## 四、 总结

`stock_in` 表不仅仅是一个简单的记录表，它是业务流转的关键节点。
*   **向上**：它终结了“采购订单”的生命周期（当入库量=订购量时，订单完成）。
*   **向下**：它开启了“库存管理”的生命周期（增加了库存，教材才能被发放）。
*   **自动化**：通过数据库触发器，它实现了入库操作与库存更新的**强一致性**，避免了代码层面的漏改或错改。
