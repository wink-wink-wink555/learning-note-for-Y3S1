# 高校教材管理系统 - 权限管理规划与ER图

## 一、数据库E-R图（实体关系图）

### 1.1 核心实体

```
┌─────────────────┐
│   用户 (User)   │
├─────────────────┤
│ PK: user_id     │
│ username        │
│ password        │
│ real_name       │
│ role ⭐         │
│ department      │
│ email           │
│ phone           │
│ last_login      │
│ status          │
└─────────────────┘
        │
        │ 操作/审批
        ▼
┌─────────────────────────────────────────────────────────┐
│                                                           │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐│
│  │出版社         │   │教材类型      │   │  教材        ││
│  │(Publisher)   │   │(TextbookType)│   │(Textbook)    ││
│  ├──────────────┤   ├──────────────┤   ├──────────────┤│
│  │PK:publisher_id│  │PK:type_id    │   │PK:textbook_id││
│  │publisher_name│   │type_name     │   │isbn          ││
│  │contact_person│   │type_code     │   │textbook_name ││
│  │contact_phone │   │description   │   │author        ││
│  │address       │   │parent_id FK  │   │publisher_id FK│
│  │email         │   │status        │   │type_id FK    ││
│  │status        │   └──────────────┘   │edition       ││
│  └──────────────┘          │           │price         ││
│         │                  │           │description   ││
│         │                  │           │status        ││
│         │                  └───────────┤              ││
│         └──────────────────────────────┘              ││
└───────────────────────────┬───────────────────────────┘│
                            │                             
                            ▼                             
        ┌────────────────────────────────────┐           
        │     库存 (Inventory)               │           
        ├────────────────────────────────────┤           
        │ PK: inventory_id                   │           
        │ FK: textbook_id (UNIQUE)           │           
        │ current_quantity                   │           
        │ total_in_quantity                  │           
        │ total_out_quantity                 │           
        │ min_quantity                       │           
        │ max_quantity                       │           
        │ last_in_date                       │           
        │ last_out_date                      │           
        └────────────────────────────────────┘           
                    ▲           ▲                         
                    │           │                         
        ┌───────────┤           └─────────────┐          
        │                                      │          
┌───────┴──────────┐                  ┌───────┴──────────┐
│   入库            │                  │   领用            │
│  (StockIn)       │                  │ (Requisition)    │
├──────────────────┤                  ├──────────────────┤
│PK: stock_in_id   │                  │PK:requisition_id │
│stock_in_no       │                  │requisition_no    │
│order_id FK       │                  │textbook_id FK    │
│textbook_id FK    │                  │requisition_qty   │
│stock_in_quantity │                  │requisition_date  │
│stock_in_date     │                  │department        │
│warehouse_person  │                  │requisition_person│
│quality_status    │                  │requisition_purpose│
│actual_quantity   │                  │approver          │
│remarks           │                  │approval_status⭐ │
└──────────────────┘                  │actual_quantity   │
        ▲                             │remarks           │
        │                             └──────────────────┘
        │                                                 
┌───────┴──────────┐                                     
│   订购            │                                     
│(PurchaseOrder)   │                                     
├──────────────────┤                                     
│PK: order_id      │                                     
│order_no          │                                     
│textbook_id FK    │                                     
│order_quantity    │                                     
│order_date        │                                     
│expected_date     │                                     
│order_person      │                                     
│order_status ⭐   │                                     
│arrived_quantity  │                                     
│remarks           │                                     
└──────────────────┘                                     
```

### 1.2 实体关系说明

1. **用户 (User)** 是系统的核心，所有操作都由用户发起
2. **出版社 (Publisher)** 1:N **教材 (Textbook)**
3. **教材类型 (TextbookType)** 1:N **教材 (Textbook)**
4. **教材 (Textbook)** 1:1 **库存 (Inventory)** （通过触发器自动创建）
5. **教材 (Textbook)** 1:N **订购单 (PurchaseOrder)**
6. **订购单 (PurchaseOrder)** 1:N **入库单 (StockIn)**
7. **教材 (Textbook)** 1:N **领用单 (Requisition)**
8. **入库单 (StockIn)** → 触发器自动更新 **库存 (Inventory)**
9. **领用单 (Requisition)** → 触发器自动更新 **库存 (Inventory)**

### 1.3 关键约束

- 教材删除时，必须保护订购单、入库单、领用单的数据完整性 (RESTRICT)
- 库存数量不能为负数
- 入库时自动更新订单状态（已到货/部分到货）
- 领用发放时检查库存是否充足

---

## 二、角色权限矩阵

### 2.1 角色定义

系统定义4种角色：
1. **管理员** - 系统最高权限，管理所有数据和用户
2. **仓库管理员** - 负责入库、出库、库存管理
3. **教师** - 可以提交领用申请、查看教材信息
4. **普通用户** - 只读权限，查看公开信息

### 2.2 权限矩阵表

| 功能模块 | 操作 | 管理员 | 仓库管理员 | 教师 | 普通用户 |
|---------|------|--------|-----------|------|----------|
| **用户管理** |
| | 查看用户列表 | ✅ | ❌ | ❌ | ❌ |
| | 创建用户 | ✅ | ❌ | ❌ | ❌ |
| | 编辑用户 | ✅ | ❌ | ❌ | ❌ |
| | 删除/停用用户 | ✅ | ❌ | ❌ | ❌ |
| | 查看个人信息 | ✅ | ✅ | ✅ | ✅ |
| **出版社管理** |
| | 查看出版社列表 | ✅ | ✅ | ✅ | ✅ |
| | 查看出版社详情 | ✅ | ✅ | ✅ | ✅ |
| | 创建出版社 | ✅ | ❌ | ❌ | ❌ |
| | 编辑出版社 | ✅ | ❌ | ❌ | ❌ |
| | 删除出版社 | ✅ | ❌ | ❌ | ❌ |
| **教材类型管理** |
| | 查看类型列表 | ✅ | ✅ | ✅ | ✅ |
| | 创建类型 | ✅ | ❌ | ❌ | ❌ |
| | 编辑类型 | ✅ | ❌ | ❌ | ❌ |
| | 删除类型 | ✅ | ❌ | ❌ | ❌ |
| **教材管理** |
| | 查看教材列表 | ✅ | ✅ | ✅ | ✅ |
| | 查看教材详情 | ✅ | ✅ | ✅ | ✅ |
| | 创建教材 | ✅ | ❌ | ❌ | ❌ |
| | 编辑教材 | ✅ | ❌ | ❌ | ❌ |
| | 删除教材 | ✅ | ❌ | ❌ | ❌ |
| **订购管理** |
| | 查看订单列表 | ✅ | ✅ | ✅ (仅自己的) | ❌ |
| | 查看订单详情 | ✅ | ✅ | ✅ (仅自己的) | ❌ |
| | 创建订单 | ✅ | ✅ | ✅ | ❌ |
| | 编辑订单 | ✅ | ✅ | ✅ (待审核状态) | ❌ |
| | 审核订单 | ✅ | ✅ | ❌ | ❌ |
| | 取消订单 | ✅ | ✅ | ✅ (待审核状态) | ❌ |
| **入库管理** |
| | 查看入库列表 | ✅ | ✅ | ❌ | ❌ |
| | 查看入库详情 | ✅ | ✅ | ❌ | ❌ |
| | 创建入库单 | ✅ | ✅ | ❌ | ❌ |
| | 编辑入库单 | ✅ | ✅ | ❌ | ❌ |
| | 删除入库单 | ✅ | ❌ | ❌ | ❌ |
| **领用管理** |
| | 查看领用列表 | ✅ | ✅ | ✅ (仅自己的) | ✅ (仅自己的) |
| | 查看领用详情 | ✅ | ✅ | ✅ (仅自己的) | ✅ (仅自己的) |
| | 创建领用申请 | ✅ | ✅ | ✅ | ✅ |
| | 编辑领用申请 | ✅ | ✅ | ✅ (待审批状态) | ✅ (待审批状态) |
| | 取消领用申请 | ✅ | ✅ | ✅ (待审批状态) | ✅ (待审批状态) |
| | 审批领用申请 | ✅ | ✅ | ❌ | ❌ |
| | 发放教材 | ✅ | ✅ | ❌ | ❌ |
| **库存管理** |
| | 查看库存列表 | ✅ | ✅ | ✅ (只读) | ✅ (只读) |
| | 查看库存详情 | ✅ | ✅ | ✅ | ✅ |
| | 库存预警查询 | ✅ | ✅ | ❌ | ❌ |
| | 调整库存阈值 | ✅ | ✅ | ❌ | ❌ |
| **统计报表** |
| | 按类型统计 | ✅ | ✅ | ✅ | ❌ |
| | 按出版社统计 | ✅ | ✅ | ✅ | ❌ |
| | 按教材统计 | ✅ | ✅ | ✅ | ❌ |
| | 按日期范围统计 | ✅ | ✅ | ✅ | ❌ |
| | 库存预警报表 | ✅ | ✅ | ❌ | ❌ |

### 2.3 权限说明

#### 管理员权限
- 拥有系统所有功能的完全访问权限
- 可以管理用户、出版社、教材类型、教材等基础数据
- 可以执行所有业务操作（订购、入库、领用）
- 可以查看所有统计报表

#### 仓库管理员权限
- 可以查看所有公共信息（出版社、教材类型、教材）
- 可以创建和管理订购单
- **核心职责**：入库管理（创建、编辑入库单）
- 可以审批领用申请并发放教材
- 可以查看库存信息和预警
- 可以查看统计报表

#### 教师权限
- 可以查看所有教材、出版社、类型信息
- 可以提交订购申请
- 可以查看自己提交的订单（订购人=当前用户）
- **核心职责**：领用教材（提交领用申请）
- 可以查看自己的领用记录
- 可以查看库存信息（只读）
- 可以查看基本统计报表

#### 普通用户权限
- 只能查看基本的教材、出版社、类型信息
- 可以提交领用申请（如学生领取教材）
- 只能查看自己的领用记录
- 可以查看库存信息（只读）
- 权限最小化

---

## 三、触发器和存储过程使用情况

### 3.1 触发器（已自动生效）

| 触发器名称 | 触发时机 | 作用 | 代码中的体现 |
|-----------|---------|------|-------------|
| `trg_textbook_after_insert` | 插入教材后 | 自动创建库存记录 | 创建教材时自动初始化库存为0 |
| `trg_stock_in_after_insert` | 插入入库记录后 | 自动更新库存和订单状态 | 入库操作会自动增加库存 |
| `trg_requisition_after_update` | 更新领用记录后 | 领用发放时自动减少库存 | 审批通过并发放时自动扣减库存 |
| `trg_stock_in_before_delete` | 删除入库记录前 | 回退库存和订单数量 | 删除入库单时自动回退 |
| `trg_purchase_order_before_update` | 更新订单前 | 防止已取消订单被修改 | 保护数据完整性 |

**触发器是数据库层面自动执行的，Python代码无需显式调用。**

### 3.2 存储过程（需要在Python中调用）

| 存储过程名称 | 功能 | 当前使用情况 | 位置 |
|-------------|------|-------------|------|
| `sp_statistics_by_type()` | 按教材类型统计 | ✅ 已使用 | `app/services/statistics_service.py:17` |
| `sp_statistics_by_publisher()` | 按出版社统计 | ✅ 已使用 | `app/services/statistics_service.py:37` |
| `sp_statistics_by_textbook(id)` | 按教材统计 | ✅ 已使用 | `app/services/statistics_service.py:57` |
| `sp_statistics_by_date_range(start, end)` | 按日期范围统计 | ✅ 已使用 | `app/services/statistics_service.py:80` |
| `sp_inventory_warning()` | 库存预警查询 | ✅ 已使用 | `app/services/statistics_service.py:97` 和 `app/dao/inventory_dao.py:54` |
| `sp_generate_order_no()` | 生成订单编号 | ✅ 已使用 | `app/services/purchase_service.py:20` |
| `sp_generate_stock_in_no()` | 生成入库单号 | ⚠️ 需要使用 | 待实现入库API时调用 |
| `sp_generate_requisition_no()` | 生成领用单号 | ⚠️ 需要使用 | 待实现领用API时调用 |

### 3.3 待补充功能

需要新增以下API来使用存储过程：

1. **入库管理API** (`app/api/v1/stock_in.py`)
   - 使用 `sp_generate_stock_in_no()` 生成入库单号
   - 需要权限控制：仅管理员和仓库管理员可操作

2. **领用管理API** (`app/api/v1/requisition.py`)
   - 使用 `sp_generate_requisition_no()` 生成领用单号
   - 需要权限控制：所有用户可申请，仅管理员和仓库管理员可审批

---

## 四、实施计划

### 4.1 权限装饰器增强
- [x] 已有基础权限装饰器 (`permission_required`, `admin_required`, `warehouse_required`)
- [ ] 新增 `teacher_or_above` 装饰器（教师及以上）
- [ ] 新增数据权限检查（如：只能查看/编辑自己的记录）

### 4.2 新增API模块
- [ ] 入库管理API (`stock_in.py`)
  - GET /api/v1/stock-ins - 查询入库列表
  - POST /api/v1/stock-ins - 创建入库单
  - GET /api/v1/stock-ins/<id> - 查询入库详情
  - PUT /api/v1/stock-ins/<id> - 编辑入库单
  - DELETE /api/v1/stock-ins/<id> - 删除入库单（仅管理员）

- [ ] 领用管理API (`requisition.py`)
  - GET /api/v1/requisitions - 查询领用列表（过滤当前用户）
  - POST /api/v1/requisitions - 创建领用申请
  - GET /api/v1/requisitions/<id> - 查询领用详情
  - PUT /api/v1/requisitions/<id> - 编辑领用申请
  - POST /api/v1/requisitions/<id>/approve - 审批领用申请
  - POST /api/v1/requisitions/<id>/issue - 发放教材
  - POST /api/v1/requisitions/<id>/reject - 拒绝领用申请

### 4.3 现有API添加权限控制
- [ ] 教材API (`textbook.py`)
  - 查看：所有登录用户
  - 创建/编辑/删除：仅管理员

- [ ] 出版社API (`publisher.py`)
  - 查看：所有登录用户
  - 创建/编辑/删除：仅管理员

- [ ] 教材类型API (`textbook_type.py`)
  - 查看：所有登录用户
  - 创建/编辑/删除：仅管理员

- [ ] 订购API (`purchase_order.py`)
  - 查看：教师及以上（过滤自己的订单）
  - 创建/编辑：教师及以上
  - 审核：管理员和仓库管理员

- [ ] 统计API (`statistics.py`)
  - 基本统计：教师及以上
  - 库存预警：管理员和仓库管理员

### 4.4 前端页面适配
- [ ] 根据用户角色显示/隐藏功能按钮
- [ ] 添加入库管理页面
- [ ] 添加领用管理页面

---

## 五、技术要点

### 5.1 权限检查方式

#### 方式1：装饰器检查角色
```python
@api_v1.route('/textbooks', methods=['POST'])
@jwt_required()
@admin_required  # 仅管理员
def create_textbook():
    pass
```

#### 方式2：装饰器检查多个角色
```python
@api_v1.route('/stock-ins', methods=['POST'])
@jwt_required()
@permission_required(['管理员', '仓库管理员'])
def create_stock_in():
    pass
```

#### 方式3：代码内部检查数据权限
```python
from flask_jwt_extended import get_jwt_identity

@api_v1.route('/requisitions', methods=['GET'])
@jwt_required()
def get_requisitions():
    current_user = get_jwt_identity()
    role = current_user.get('role')
    
    # 管理员和仓库管理员可以查看所有记录
    if role in ['管理员', '仓库管理员']:
        items, total = requisition_dao.paginate(...)
    else:
        # 教师和普通用户只能查看自己的记录
        filters = {'requisition_person': current_user.get('real_name')}
        items, total = requisition_dao.paginate(..., filters=filters)
    
    return paginated_response(items, total, page, per_page)
```

### 5.2 存储过程调用方式

```python
from sqlalchemy import text
from app.extensions import db

# 无参数存储过程
result = db.session.execute(text("CALL sp_inventory_warning()"))
rows = result.fetchall()

# 有参数存储过程
result = db.session.execute(
    text("CALL sp_statistics_by_textbook(:textbook_id)"),
    {"textbook_id": 1}
)

# 输出参数存储过程
db.session.execute(text("CALL sp_generate_order_no(@order_no)"))
result = db.session.execute(text("SELECT @order_no"))
order_no = result.scalar()
```

---

## 六、测试检查清单

实施完成后需要验证：

- [ ] 管理员可以访问所有功能
- [ ] 仓库管理员不能编辑教材、出版社等基础数据
- [ ] 教师不能进行入库操作
- [ ] 教师只能查看自己的订单和领用记录
- [ ] 普通用户不能查看统计报表
- [ ] 普通用户只能查看自己的领用记录
- [ ] 领用审批后自动扣减库存（触发器生效）
- [ ] 入库后自动增加库存（触发器生效）
- [ ] 所有存储过程都被正确调用
- [ ] 生成的单号格式正确且连续

---

## 七、总结

### 触发器 vs 存储过程

- **触发器**：数据库自动执行，无需代码调用，用于保证数据一致性
  - 例如：入库自动更新库存、领用自动扣减库存

- **存储过程**：需要在Python代码中显式调用，用于复杂查询和业务逻辑
  - 例如：生成单号、统计查询

### 权限控制核心思想

1. **角色分离**：不同角色有明确的职责边界
2. **最小权限原则**：只给必要的权限，降低误操作风险
3. **数据隔离**：非管理员只能查看/操作自己的数据
4. **业务保护**：基础数据（教材、出版社）只有管理员可修改

---

**文档版本**: 1.0  
**创建日期**: 2026-01-10  
**维护人**: AI Assistant
