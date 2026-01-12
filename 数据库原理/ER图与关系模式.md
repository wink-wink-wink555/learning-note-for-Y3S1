# 高校教材管理系统 - ER图与关系模式

## 一、ER图（Mermaid）

```mermaid
erDiagram
    %% 实体定义与属性

    PUBLISHER {
        INT publisher_id PK "出版社ID"
        VARCHAR publisher_name UK "出版社名称"
        VARCHAR contact_person "联系人"
        VARCHAR contact_phone "联系电话"
        VARCHAR address "地址"
        VARCHAR email "邮箱"
        TIMESTAMP created_at "创建时间"
        TIMESTAMP updated_at "更新时间"
        TINYINT status "状态"
    }

    TEXTBOOK_TYPE {
        INT type_id PK "类型ID"
        VARCHAR type_name UK "类型名称"
        VARCHAR type_code UK "类型编码"
        TEXT description "类型描述"
        INT parent_id FK "父类型ID"
        TIMESTAMP created_at "创建时间"
        TIMESTAMP updated_at "更新时间"
        TINYINT status "状态"
    }

    TEXTBOOK {
        INT textbook_id PK "教材ID"
        VARCHAR isbn UK "ISBN"
        VARCHAR textbook_name "教材名称"
        VARCHAR author "作者"
        INT publisher_id FK "出版社ID"
        INT type_id FK "教材类型ID"
        VARCHAR edition "版次"
        DATE publication_date "出版日期"
        DECIMAL price "单价"
        TEXT description "教材描述"
        TIMESTAMP created_at "创建时间"
        TIMESTAMP updated_at "更新时间"
        TINYINT status "状态"
    }

    PURCHASE_ORDER {
        INT order_id PK "订单ID"
        VARCHAR order_no UK "订单编号"
        INT textbook_id FK "教材ID"
        INT order_quantity "订购数量"
        DATE order_date "订购日期"
        DATE expected_date "预计到货日期"
        VARCHAR order_person "订购人"
        ENUM order_status "订单状态"
        INT arrived_quantity "已到货数量"
        TEXT remarks "备注"
        TIMESTAMP created_at "创建时间"
        TIMESTAMP updated_at "更新时间"
    }

    STOCK_IN {
        INT stock_in_id PK "入库ID"
        VARCHAR stock_in_no UK "入库单号"
        INT order_id FK "订单ID"
        INT textbook_id FK "教材ID"
        INT stock_in_quantity "入库数量"
        DATE stock_in_date "入库日期"
        VARCHAR warehouse_person "仓库管理员"
        ENUM quality_status "质量状态"
        INT actual_quantity "实际入库数量"
        TEXT remarks "备注"
        TIMESTAMP created_at "创建时间"
        TIMESTAMP updated_at "更新时间"
    }

    INVENTORY {
        INT inventory_id PK "库存ID"
        INT textbook_id FK "教材ID"
        INT current_quantity "当前库存数量"
        INT total_in_quantity "累计入库数量"
        INT total_out_quantity "累计出库数量"
        INT min_quantity "最低库存预警值"
        INT max_quantity "最高库存预警值"
        DATE last_in_date "最后入库日期"
        DATE last_out_date "最后出库日期"
        TIMESTAMP created_at "创建时间"
        TIMESTAMP updated_at "更新时间"
    }

    USER {
        INT user_id PK "用户ID"
        VARCHAR username UK "用户名"
        VARCHAR password "密码"
        VARCHAR real_name "真实姓名"
        ENUM role "用户角色"
        VARCHAR department "所属部门"
        VARCHAR email "邮箱"
        VARCHAR phone "电话"
        TIMESTAMP created_at "创建时间"
        TIMESTAMP updated_at "更新时间"
        TIMESTAMP last_login "最后登录时间"
        TINYINT status "状态"
    }

    %% 关系定义 - 添加了基数标注
    PUBLISHER ||--o{ TEXTBOOK : "1:N (一家出版社可以出版多本教材)"
    TEXTBOOK_TYPE ||--o{ TEXTBOOK : "1:N (一个类型下可以有多本教材)"
    TEXTBOOK_TYPE ||--o| TEXTBOOK_TYPE : "1:0..N (一个父类型可以有零个或多个子类型)"
    TEXTBOOK ||--o{ PURCHASE_ORDER : "1:N (一本教材可以被多次订购)"
    TEXTBOOK ||--o{ STOCK_IN : "1:N (一本教材可以有多次入库记录)"
    TEXTBOOK ||--|| INVENTORY : "1:1 (一本教材对应一条库存记录)"
    PURCHASE_ORDER ||--o{ STOCK_IN : "1:N (一个订单可以对应多次入库操作)"
    USER ||--o{ PURCHASE_ORDER : "1:N (一个用户可以创建多个订单)"
    USER ||--o{ STOCK_IN : "1:N (一个用户可以管理多个入库单)"
```

---

## 二、ER图简化版（便于查看关系）

```mermaid
erDiagram
    PUBLISHER ||--o{ TEXTBOOK : "出版"
    TEXTBOOK_TYPE ||--o{ TEXTBOOK : "分类"
    TEXTBOOK_TYPE ||--o| TEXTBOOK_TYPE : "父子类型"
    TEXTBOOK ||--o{ PURCHASE_ORDER : "被订购"
    TEXTBOOK ||--o{ STOCK_IN : "入库"
    TEXTBOOK ||--|| INVENTORY : "库存"
    PURCHASE_ORDER ||--o{ STOCK_IN : "入库来源"
    USER }o--o| USER : "独立用户管理"
```

---

## 三、关系模式（标注主码与外键）

### 1. 出版社 (Publisher)

```
Publisher (
    publisher_id,       -- 主码 (PK)
    publisher_name,     -- 候选码 (UNIQUE)
    contact_person,
    contact_phone,
    address,
    email,
    created_at,
    updated_at,
    status
)
```

| 属性 | 说明 |
|------|------|
| **publisher_id** | **主码 (Primary Key)** |
| publisher_name | 候选码 (UNIQUE) |

---

### 2. 教材类型 (Textbook_Type)

```
Textbook_Type (
    type_id,            -- 主码 (PK)
    type_name,          -- 候选码 (UNIQUE)
    type_code,          -- 候选码 (UNIQUE)
    description,
    parent_id,          -- 外键 → Textbook_Type(type_id)
    created_at,
    updated_at,
    status
)
```

| 属性 | 说明 |
|------|------|
| **type_id** | **主码 (Primary Key)** |
| type_name | 候选码 (UNIQUE) |
| type_code | 候选码 (UNIQUE) |
| *parent_id* | *外键 (Foreign Key) → Textbook_Type(type_id)* |

---

### 3. 教材 (Textbook)

```
Textbook (
    textbook_id,        -- 主码 (PK)
    isbn,               -- 候选码 (UNIQUE)
    textbook_name,
    author,
    publisher_id,       -- 外键 → Publisher(publisher_id)
    type_id,            -- 外键 → Textbook_Type(type_id)
    edition,
    publication_date,
    price,
    description,
    created_at,
    updated_at,
    status
)
```

| 属性 | 说明 |
|------|------|
| **textbook_id** | **主码 (Primary Key)** |
| isbn | 候选码 (UNIQUE) |
| *publisher_id* | *外键 (Foreign Key) → Publisher(publisher_id)* |
| *type_id* | *外键 (Foreign Key) → Textbook_Type(type_id)* |

---

### 4. 订购表 (Purchase_Order)

```
Purchase_Order (
    order_id,           -- 主码 (PK)
    order_no,           -- 候选码 (UNIQUE)
    textbook_id,        -- 外键 → Textbook(textbook_id)
    order_quantity,
    order_date,
    expected_date,
    order_person,
    order_status,
    arrived_quantity,
    remarks,
    created_at,
    updated_at
)
```

| 属性 | 说明 |
|------|------|
| **order_id** | **主码 (Primary Key)** |
| order_no | 候选码 (UNIQUE) |
| *textbook_id* | *外键 (Foreign Key) → Textbook(textbook_id)* |

---

### 5. 入库表 (Stock_In)

```
Stock_In (
    stock_in_id,        -- 主码 (PK)
    stock_in_no,        -- 候选码 (UNIQUE)
    order_id,           -- 外键 → Purchase_Order(order_id)
    textbook_id,        -- 外键 → Textbook(textbook_id)
    stock_in_quantity,
    stock_in_date,
    warehouse_person,
    quality_status,
    actual_quantity,
    remarks,
    created_at,
    updated_at
)
```

| 属性 | 说明 |
|------|------|
| **stock_in_id** | **主码 (Primary Key)** |
| stock_in_no | 候选码 (UNIQUE) |
| *order_id* | *外键 (Foreign Key) → Purchase_Order(order_id)* |
| *textbook_id* | *外键 (Foreign Key) → Textbook(textbook_id)* |

---

### 6. 库存表 (Inventory)

```
Inventory (
    inventory_id,       -- 主码 (PK)
    textbook_id,        -- 外键 → Textbook(textbook_id)，且 UNIQUE
    current_quantity,
    total_in_quantity,
    total_out_quantity,
    min_quantity,
    max_quantity,
    last_in_date,
    last_out_date,
    created_at,
    updated_at
)
```

| 属性 | 说明 |
|------|------|
| **inventory_id** | **主码 (Primary Key)** |
| *textbook_id* | *外键 (Foreign Key) → Textbook(textbook_id)*，候选码 (UNIQUE) |

> 📝 说明：`textbook_id` 同时是外键和候选码（UNIQUE），确保每本教材只有一条库存记录（1:1关系）。

---

### 7. 用户表 (User)

```
User (
    user_id,            -- 主码 (PK)
    username,           -- 候选码 (UNIQUE)
    password,
    real_name,
    role,
    department,
    email,
    phone,
    created_at,
    updated_at,
    last_login,
    status
)
```

| 属性 | 说明 |
|------|------|
| **user_id** | **主码 (Primary Key)** |
| username | 候选码 (UNIQUE) |

> 📝 说明：用户表是独立表，用于系统登录和权限管理，不与其他业务表有直接外键关联。

---

## 四、实体关系汇总表

| 关系 | 类型 | 外键位置 | 说明 |
|------|------|----------|------|
| Publisher → Textbook | 1:N | Textbook.publisher_id | 一个出版社出版多本教材 |
| Textbook_Type → Textbook | 1:N | Textbook.type_id | 一个类型包含多本教材 |
| Textbook_Type → Textbook_Type | 1:N | Textbook_Type.parent_id | 自关联，支持分级分类 |
| Textbook → Purchase_Order | 1:N | Purchase_Order.textbook_id | 一本教材可有多个订单 |
| Textbook → Stock_In | 1:N | Stock_In.textbook_id | 一本教材可有多次入库 |
| Textbook → Inventory | 1:1 | Inventory.textbook_id (UNIQUE) | 一本教材对应一条库存 |
| Purchase_Order → Stock_In | 1:N | Stock_In.order_id | 一个订单可有多次入库 |

---

## 五、外键约束行为

| 外键 | ON DELETE | ON UPDATE | 说明 |
|------|-----------|-----------|------|
| Textbook_Type.parent_id | SET NULL | CASCADE | 父类型删除时，子类型的parent_id设为NULL |
| Textbook.publisher_id | RESTRICT | CASCADE | 禁止删除有教材的出版社 |
| Textbook.type_id | RESTRICT | CASCADE | 禁止删除有教材的类型 |
| Purchase_Order.textbook_id | RESTRICT | CASCADE | 禁止删除有订单的教材 |
| Stock_In.order_id | RESTRICT | CASCADE | 禁止删除有入库记录的订单 |
| Stock_In.textbook_id | RESTRICT | CASCADE | 禁止删除有入库记录的教材 |
| Inventory.textbook_id | RESTRICT | CASCADE | 禁止删除有库存记录的教材 |

---

## 六、数据库范式分析

本系统数据库设计满足 **第三范式（3NF）**：

1. **1NF**: 所有属性都是原子的，不可再分
2. **2NF**: 每个非主属性完全函数依赖于主码
3. **3NF**: 不存在传递函数依赖（非主属性不依赖于其他非主属性）

---

> 📅 文档生成时间：2026年1月12日  
> 📁 对应SQL文件：`sql/02_create_tables.sql`



