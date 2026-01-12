# 数据库对象与代码映射详解

本文档详细讲解项目中 SQL 文件夹内定义的**存储过程**、**触发器**和**视图**。我们将用通俗易懂的语言解释它们是干什么的，前端页面哪里用到了它，以及数据是如何一步步传过去的。

## 一、存储过程 (Procedures) —— "打包好的功能包"

存储过程就像是数据库里预先写好的函数。与其在 Python 代码里写一大堆复杂的 SQL 拼凑在一起，不如直接调用一个名字，数据库自己就搞定了。

### 1. 统计类存储过程

这些过程主要用于生成报表数据，由 `StatisticsService` 调用。

#### 1.1 `sp_statistics_by_type` (按类型统计)

*   **功能**：统计每种教材类型下有多少种书、订了多少本、到了多少本、发了多少本。
*   **前端页面功能**：
    *   **位置**：`dashboard.html` (仪表盘) 的 "各类教材统计" 卡片，以及 `statistics.html` (统计报表) 的 "按教材类型统计" 表格。
    *   **用户操作**：用户打开页面时自动加载。
*   **详细调用链路**：
    1.  **前端 (JS)**: `api.getStatisticsByType()` 发起 GET 请求到 `/api/v1/statistics/by-type`。
    2.  **API 层**: `StatisticsAPI.get_statistics_by_type` 接收请求。
    3.  **Service 层**: `StatisticsService.get_statistics_by_type` 执行 `db.session.execute("CALL sp_statistics_by_type()")`。
    4.  **数据库**: 执行存储过程，聚合计算 `textbook`、`purchase_order`、`inventory` 表的数据并返回结果集。
*   **代码应用**：
    ```python
    # app/services/statistics_service.py
    def get_statistics_by_type(self):
        """按类型统计"""
        # 直接呼叫存储过程，不需要在Python里写复杂的 Join 查询
        result = db.session.execute(text("CALL sp_statistics_by_type()"))
        rows = result.fetchall()
        # ... 处理返回结果 ...
    ```

#### 1.2 `sp_statistics_by_publisher` (按出版社统计)

*   **功能**：看看哪个出版社的书买得最多。
*   **前端页面功能**：
    *   **位置**：`statistics.html` (统计报表) 的 "按出版社统计" 表格。
    *   **用户操作**：用户打开统计页面时自动加载。
*   **详细调用链路**：
    1.  **前端 (JS)**: `loadPublisherStats()` 调用 `api.getStatisticsByPublisher()`。
    2.  **API 层**: 路由 `/api/v1/statistics/by-publisher` 触发。
    3.  **Service 层**: `StatisticsService` 调用 `CALL sp_statistics_by_publisher()`。
    4.  **数据库**: 关联 `publisher` 和 `textbook` 表进行统计。
*   **代码应用**：
    ```python
    # app/services/statistics_service.py
    def get_statistics_by_publisher(self):
        result = db.session.execute(text("CALL sp_statistics_by_publisher()"))
    ```

#### 1.3 `sp_statistics_by_textbook` (按教材统计) & `sp_statistics_by_date_range`

*   **功能**：查某一本特定教材的详细流水，或查某段时间内的业务量。
*   **前端页面功能**：这些是为更细粒度的报表查询准备的接口，目前在前端主要通过 API 调用展示详情，或者作为未来扩展功能的后端支持。
*   **代码应用**：
    ```python
    # app/services/statistics_service.py
    def get_statistics_by_textbook(self, textbook_id):
        result = db.session.execute(
            text(f"CALL sp_statistics_by_textbook({textbook_id})")
        )
    ```

### 2. 编号生成类存储过程

这些过程用于生成格式统一的单据编号，确保不会重复且格式规范。

#### 2.1 `sp_generate_order_no` (生成订单编号)

*   **功能**：生成一个新的采购单号。格式：`PO` + `年月日` + `4位流水号` (例如 `PO202401010001`)。
*   **前端页面功能**：
    *   **位置**：订单管理页面 (`orders.html`)。
    *   **用户操作**：用户点击 "新建订单" 按钮并提交表单时。
*   **详细调用链路**：
    1.  **前端**: 用户提交表单，调用 `api.createPurchaseOrder(data)`。
    2.  **API 层**: `PurchaseOrderAPI` 接收数据。
    3.  **Service 层**: `PurchaseService.create_order` 方法第一步就是调用 `self.generate_order_no()`。
    4.  **数据库**: 存储过程计算当天的最大流水号，加 1 后返回新编号。
    5.  **Service 层**: 拿到编号后，将其作为 `order_no` 字段，写入 `purchase_order` 表。
*   **代码应用**：
    ```python
    # app/services/purchase_service.py
    def generate_order_no(self):
        # 调用存储过程生成编号，并读取输出变量 @order_no
        result = db.session.execute(
            text("CALL sp_generate_order_no(@order_no)")
        )
        order_no = db.session.execute(text("SELECT @order_no")).scalar()
        return order_no
    ```

---

## 二、触发器 (Triggers) —— "自动连锁反应"

触发器是数据库的“自动机关”。你不需要在 Python 代码里显式调用它们，只要你动了表里的数据，它们就会自动执行。

#### 2.1 `trg_stock_in_after_insert` (入库自动更新库存)

*   **功能**：这是系统最核心的自动化逻辑。
    1.  找到对应的 `inventory` (库存) 表记录，把数量加上去。
    2.  找到对应的 `purchase_order` (订单) 表记录，把“已到货数量”更新，如果到齐了，把状态自动改为“已到货”。
*   **前端页面功能**：
    *   **位置**：入库管理页面 (或者订单页面的 "入库" 按钮)。
    *   **用户操作**：仓库管理员填写实际入库数量，点击 "确认入库"。
*   **详细调用链路**：
    1.  **前端**: `api.createStockIn(data)` 发送入库请求。
    2.  **Service 层**: `StockInService.create_stock_in` 验证数据合法性。
    3.  **DAO 层**: `stock_in_dao.create(data)` 执行 SQL `INSERT INTO stock_in ...`。
    4.  **数据库 (关键)**: 
        *   INSERT 语句刚执行完，**触发器立马启动**。
        *   触发器内部执行 `UPDATE inventory SET current_quantity = ...`。
        *   触发器内部执行 `UPDATE purchase_order SET arrived_quantity = ...`。
    5.  **结果**: Python 代码完全不用写更新库存的逻辑，数据就已经一致了。
*   **代码场景**：
    ```python
    # app/services/stock_in_service.py
    # Python 代码只管插入入库记录
    stock_in = self.stock_in_dao.create(data)
    # 此时，数据库在后台悄悄执行了触发器，把库存和订单状态都修好了！
    ```

#### 2.2 `trg_stock_in_before_delete` (删除入库回退库存)

*   **功能**：如果不小心录错了入库单，删除它时，库存会自动扣减回去，订单状态也会回退。
*   **前端页面功能**：入库记录列表页面的 "删除" 按钮。
*   **详细调用链路**：
    1.  **Service 层**: `StockInService.delete_stock_in` 调用 `delete` 方法。
    2.  **数据库**: 在执行 `DELETE FROM stock_in` 之前，触发器先检查库存够不够扣。
    3.  **逻辑**: 如果库存够，触发器自动减少库存；如果不够（比如入库后已经被发走了），触发器会报错，阻止删除操作，保护数据安全。

---

## 三、视图 (Views) —— "保存好的虚拟报表"

视图就像是一个保存好的 SQL 查询结果。在 Python 看来，它就像一张普通的表。

#### 3.1 `v_inventory_warning` (库存预警视图)

*   **功能**：它自动筛选出那些 **库存不足**（低于最低线）或者 **库存积压**（高于最高线）的教材。
*   **前端页面功能**：
    *   **位置**：`dashboard.html` (仪表盘) 右下角的 "库存预警" 列表，以及 `statistics.html` 页面。
    *   **用户操作**：页面加载时，管理员看到的红色高亮列表。
*   **详细调用链路**：
    1.  **前端**: `loadWarnings()` 调用 `api.getInventoryWarnings()`。
    2.  **Service 层**: `StatisticsService.get_inventory_warnings`。
    3.  **数据库**: 执行 `SELECT * FROM v_inventory_warning ORDER BY warning_level`。
    4.  **优势**: Python 代码不需要去遍历所有书判断 `if current < min`，数据库视图已经预先筛选好了，查询速度极快，代码也简洁。
*   **代码应用**：
    ```python
    # app/services/statistics_service.py
    def get_inventory_warnings(self):
        result = db.session.execute(text("""
            SELECT 
                textbook_id, isbn, textbook_name, ...
            FROM v_inventory_warning  -- 直接查视图，像查普通表一样
            ORDER BY warning_level ASC
        """))
    ```
