# API使用示例

## 基础信息

- **Base URL**: `http://localhost:5000/api/v1`
- **认证方式**: JWT Bearer Token
- **Content-Type**: `application/json`

## 1. 认证接口

### 1.1 用户登录

```bash
curl -X POST http://localhost:5000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "admin123"
  }'
```

**响应示例**:
```json
{
  "code": 200,
  "message": "登录成功",
  "data": {
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
    "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
    "user": {
      "user_id": 1,
      "username": "admin",
      "real_name": "系统管理员",
      "role": "管理员"
    }
  },
  "timestamp": 1704708000
}
```

## 2. 出版社接口

### 2.1 获取出版社列表

```bash
curl -X GET "http://localhost:5000/api/v1/publishers?page=1&per_page=10" \
  -H "Authorization: Bearer <your_token>"
```

### 2.2 创建出版社

```bash
curl -X POST http://localhost:5000/api/v1/publishers \
  -H "Authorization: Bearer <your_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "publisher_name": "北京大学出版社",
    "contact_person": "李四",
    "contact_phone": "13800138000",
    "address": "北京市海淀区",
    "email": "contact@pku.edu.cn"
  }'
```

## 3. 教材接口

### 3.1 创建教材

```bash
curl -X POST http://localhost:5000/api/v1/textbooks \
  -H "Authorization: Bearer <your_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "isbn": "ISBN9787040005",
    "textbook_name": "算法导论",
    "author": "Thomas H. Cormen",
    "publisher_id": 1,
    "type_id": 1,
    "edition": "第3版",
    "publication_date": "2023-01-01",
    "price": "128.00",
    "description": "经典算法教材"
  }'
```

### 3.2 获取教材列表（带筛选）

```bash
curl -X GET "http://localhost:5000/api/v1/textbooks?keyword=算法&type_id=1&page=1" \
  -H "Authorization: Bearer <your_token>"
```

### 3.3 获取教材详情

```bash
curl -X GET http://localhost:5000/api/v1/textbooks/1 \
  -H "Authorization: Bearer <your_token>"
```

### 3.4 更新教材

```bash
curl -X PUT http://localhost:5000/api/v1/textbooks/1 \
  -H "Authorization: Bearer <your_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "price": "138.00",
    "description": "更新的描述"
  }'
```

## 4. 订购接口

### 4.1 创建订单

```bash
curl -X POST http://localhost:5000/api/v1/purchase-orders \
  -H "Authorization: Bearer <your_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "textbook_id": 1,
    "order_quantity": 100,
    "order_date": "2024-01-15",
    "expected_date": "2024-02-01",
    "order_person": "张管理",
    "remarks": "春季学期订购"
  }'
```

### 4.2 审核订单

```bash
curl -X POST http://localhost:5000/api/v1/purchase-orders/1/approve \
  -H "Authorization: Bearer <your_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "approver": "admin"
  }'
```

### 4.3 取消订单

```bash
curl -X POST http://localhost:5000/api/v1/purchase-orders/1/cancel \
  -H "Authorization: Bearer <your_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "reason": "供应商无法按时交货"
  }'
```

## 5. 统计接口

### 5.1 获取仪表盘数据

```bash
curl -X GET http://localhost:5000/api/v1/statistics/dashboard \
  -H "Authorization: Bearer <your_token>"
```

**响应示例**:
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "textbook_count": 15,
    "pending_orders": 3,
    "pending_requisitions": 2,
    "inventory_value": 125680.50,
    "warning_count": 5
  }
}
```

### 5.2 按类型统计

```bash
curl -X GET http://localhost:5000/api/v1/statistics/by-type \
  -H "Authorization: Bearer <your_token>"
```

### 5.3 按日期范围统计

```bash
curl -X GET "http://localhost:5000/api/v1/statistics/by-date?start_date=2024-01-01&end_date=2024-12-31" \
  -H "Authorization: Bearer <your_token>"
```

### 5.4 库存预警查询

```bash
curl -X GET http://localhost:5000/api/v1/statistics/inventory-warnings \
  -H "Authorization: Bearer <your_token>"
```

## 6. 教材类型接口

### 6.1 获取类型树形结构

```bash
curl -X GET http://localhost:5000/api/v1/textbook-types/tree \
  -H "Authorization: Bearer <your_token>"
```

### 6.2 获取类型列表

```bash
curl -X GET http://localhost:5000/api/v1/textbook-types \
  -H "Authorization: Bearer <your_token>"
```

## Python 示例代码

```python
import requests

# 基础URL
BASE_URL = "http://localhost:5000/api/v1"

# 1. 登录获取Token
def login():
    response = requests.post(
        f"{BASE_URL}/auth/login",
        json={
            "username": "admin",
            "password": "admin123"
        }
    )
    return response.json()['data']['access_token']

# 2. 获取教材列表
def get_textbooks(token):
    headers = {"Authorization": f"Bearer {token}"}
    response = requests.get(
        f"{BASE_URL}/textbooks",
        headers=headers,
        params={"page": 1, "per_page": 10}
    )
    return response.json()

# 3. 创建教材
def create_textbook(token):
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    data = {
        "isbn": "ISBN9787040006",
        "textbook_name": "数据结构",
        "author": "严蔚敏",
        "publisher_id": 1,
        "type_id": 1,
        "price": "45.00"
    }
    response = requests.post(
        f"{BASE_URL}/textbooks",
        headers=headers,
        json=data
    )
    return response.json()

# 使用示例
if __name__ == "__main__":
    # 登录
    token = login()
    print(f"Token: {token}")
    
    # 获取教材列表
    textbooks = get_textbooks(token)
    print(f"教材数量: {textbooks['data']['pagination']['total']}")
    
    # 创建教材
    result = create_textbook(token)
    print(f"创建结果: {result['message']}")
```

## JavaScript 示例代码

```javascript
// 基础URL
const BASE_URL = "http://localhost:5000/api/v1";

// 1. 登录获取Token
async function login() {
  const response = await fetch(`${BASE_URL}/auth/login`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      username: 'admin',
      password: 'admin123'
    })
  });
  const data = await response.json();
  return data.data.access_token;
}

// 2. 获取教材列表
async function getTextbooks(token) {
  const response = await fetch(`${BASE_URL}/textbooks?page=1&per_page=10`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
}

// 3. 创建教材
async function createTextbook(token) {
  const response = await fetch(`${BASE_URL}/textbooks`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      isbn: 'ISBN9787040007',
      textbook_name: '操作系统',
      author: '汤小丹',
      publisher_id: 1,
      type_id: 1,
      price: '48.00'
    })
  });
  return await response.json();
}

// 使用示例
(async () => {
  try {
    // 登录
    const token = await login();
    console.log('Token:', token);
    
    // 获取教材列表
    const textbooks = await getTextbooks(token);
    console.log('教材数量:', textbooks.data.pagination.total);
    
    // 创建教材
    const result = await createTextbook(token);
    console.log('创建结果:', result.message);
  } catch (error) {
    console.error('Error:', error);
  }
})();
```

## 错误码说明

| 状态码 | 说明 |
|-------|------|
| 200 | 请求成功 |
| 201 | 创建成功 |
| 400 | 请求参数错误 |
| 401 | 未认证或Token过期 |
| 403 | 权限不足 |
| 404 | 资源不存在 |
| 500 | 服务器内部错误 |

## 注意事项

1. 所有接口（除登录和注册外）都需要携带JWT Token
2. Token默认有效期为1小时
3. 分页参数：page从1开始，per_page默认20，最大100
4. 日期格式统一使用：YYYY-MM-DD
5. 价格使用字符串格式，保留两位小数

