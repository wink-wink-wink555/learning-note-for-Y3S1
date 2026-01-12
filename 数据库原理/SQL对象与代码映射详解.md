# 数据库对象与代码映射详解

本文档详细讲解项目中 SQL 文件夹内定义的**存储过程**、**触发器**和**视图**。我们将用通俗易懂的语言解释它们是干什么的，并展示它们在 Python 代码中是如何被使用的。

## 一、存储过程 (Procedures) —— "打包好的功能包"

存储过程就像是数据库里预先写好的函数。与其在 Python 代码里写一大堆复杂的 SQL 拼凑在一起，不如直接调用一个名字，数据库自己就搞定了。

### 1. 统计类存储过程

这些过程主要用于生成报表数据，由 `StatisticsService` (`app/services/statistics_service.py`) 调用。

#### 1.1 `sp_statistics_by_type` (按类型统计)
*   **功能**：统计每种教材类型下有多少种书、订了多少本、到了多少本、发了多少本。
*   **输入**：无
*   **输出**：类型名称、教材种类数、订购数、到货数等表格数据。
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

#### 1.2 `sp_statistics_by_textbook` (按教材统计)
*   **功能**：查某一本特定教材的详细流水（订了多少、发了多少）。
*   **输入**：`p_textbook_id` (教材ID)
*   **输出**：该教材的详细统计信息。
*   **代码应用**：
    ```python
    # app/services/statistics_service.py
    def get_statistics_by_textbook(self, textbook_id):
        # 传入 ID 参数
        result = db.session.execute(
            text(f"CALL sp_statistics_by_textbook({textbook_id})")
        )
    ```

#### 1.3 `sp_statistics_by_date_range` (按日期范围统计)
*   **功能**：查某段时间内的业务量（比如“上个月我们要处理多少书？”）。
*   **输入**：`p_start_date` (开始日期), `p_end_date` (结束日期)
*   **输出**：按月份分组的订购、到货、发放数量。
*   **代码应用**：
    ```python
    # app/services/statistics_service.py
    def get_statistics_by_date_range(self, start_date, end_date):
        result = db.session.execute(
            text(f"CALL sp_statistics_by_date_range('{start_date}', '{end_date}')")
        )
    ```

#### 1.4 `sp_statistics_by_publisher` (按出版社统计)
*   **功能**：看看哪个出版社的书买得最多。
*   **输入**：无
*   **输出**：出版社列表及对应的统计数据。
*   **代码应用**：
    ```python
    # app/services/statistics_service.py
    def get_statistics_by_publisher(self):
        result = db.session.execute(text("CALL sp_statistics_by_publisher()"))
    ```

### 2. 编号生成类存储过程

这些过程用于生成格式统一的单据编号，确保不会重复且格式规范（如 `PO202401010001`）。

#### 2.1 `sp_generate_order_no` (生成订单编号)
*   **功能**：生成一个新的采购单号。格式：`PO` + `年月日` + `4位流水号`。
*   **输入**：无
*   **输出**：`p_order_no` (生成的编号)
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

#### 2.2 `sp_generate_stock_in_no` (生成入库单号)
*   **功能**：生成一个新的入库单号。格式：`SI` + `年月日` + `4位流水号`。
*   **代码应用**：
    ```python
    # app/services/stock_in_service.py
    def generate_stock_in_no(self):
        db.session.execute(text("CALL sp_generate_stock_in_no(@stock_in_no)"))
        # ... 读取 @stock_in_no ...
    ```

---

## 二、触发器 (Triggers) —— "自动连锁反应"

触发器是数据库的“自动机关”。你不需要在 Python 代码里显式调用它们，只要你动了表里的数据，它们就会自动执行。

#### 2.1 `trg_stock_in_after_insert` (入库自动更新库存)
*   **什么时候触发**：当你向 `stock_in` 表插入一条数据（即：创建入库单）之后。
*   **自动做了什么**：
    1.  找到对应的 `inventory` (库存) 表记录，把数量加上去。
    2.  找到对应的 `purchase_order` (订单) 表记录，把“已到货数量”更新，如果到齐了，把状态改为“已到货”。
*   **代码场景**：
    在 `StockInService.create_stock_in` 方法中：
    ```python
    # app/services/stock_in_service.py
    # Python 代码只管插入入库记录
    stock_in = self.stock_in_dao.create(data)
    # 此时，数据库在后台悄悄执行了触发器，把库存和订单状态都修好了！
    # 我们不需要手动去 update inventory 或 update purchase_order
    ```

#### 2.2 `trg_stock_in_before_delete` (删除入库回退库存)
*   **什么时候触发**：当你删除一条入库记录之前。
*   **自动做了什么**：
    1.  检查库存够不够扣（防止库存变成负数）。
    2.  如果够扣，就把库存数量减回去。
    3.  把订单的“已到货数量”也减回去。
*   **代码场景**：
    在 `StockInService.delete_stock_in` 方法中：
    ```python
    # app/services/stock_in_service.py
    self.stock_in_dao.delete(stock_in_id, soft=False)
    # 如果库存不足，触发器会报错，Python 这里会捕获到异常
    ```

#### 2.3 `trg_textbook_after_insert` (新建教材初始化库存)
*   **什么时候触发**：当你添加一本新书 (`textbook` 表插入) 之后。
*   **自动做了什么**：在 `inventory` 表里自动插一行数据，数量设为 0。
*   **代码场景**：
    在 `TextbookService.create_textbook` 中，我们只插入了教材表，没管库存表，因为触发器帮我们把库存记录建好了。

#### 2.4 `trg_purchase_order_before_update` (订单状态保护)
*   **什么时候触发**：更新订单信息之前。
*   **自动做了什么**：检查订单是不是已经“取消”或“发放”了。如果是，禁止修改，直接报错。这是一种数据保护机制。

---

## 三、视图 (Views) —— "保存好的虚拟报表"

视图就像是一个保存好的 SQL 查询结果。在 Python 看来，它就像一张普通的表，但其实它里面存的是经过计算、关联后的数据。

#### 3.1 `v_inventory_warning` (库存预警视图)
*   **功能**：它自动筛选出那些 **库存不足**（低于最低线）或者 **库存积压**（高于最高线）的教材。它还算好了缺口数量和预警级别。
*   **代码应用**：
    在 `StatisticsService` 中，我们像查表一样查它：
    ```python
    # app/services/statistics_service.py
    def get_inventory_warnings(self):
        result = db.session.execute(text("""
            SELECT 
                textbook_id, isbn, textbook_name, ...
            FROM v_inventory_warning  -- 直接查视图
            ORDER BY warning_level ASC
        """))
    ```

#### 3.2 `v_textbook_detail` (教材详情视图)
*   **功能**：把教材表、出版社表、类型表、库存表拼在了一起，形成一张“宽表”。
*   **现状**：这是一个扩展功能，目前的 Python 代码主要使用 ORM 的关联查询（Relationship），暂时没有大量使用这个视图，但如果未来要做纯报表查询，这个视图会非常方便。
