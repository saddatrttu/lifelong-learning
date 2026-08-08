# 3. 接口测试实践

> 接口（API）测试是测试金字塔里"性价比最高"的一层——覆盖业务规则、鉴权、契约，速度快且稳定。本笔记用 pytest + requests 讲一套完整的接口测试实践。

---

## 目录

1. [为什么接口测试价值高](#为什么接口测试价值高)
2. [基础：requests 客户端](#基础requests-客户端)
3. [接口测试结构](#接口测试结构)
4. [数据驱动](#数据驱动)
5. [环境管理](#环境管理)
6. [鉴权处理](#鉴权处理)
7. [断言设计](#断言设计)
8. [接口用例设计清单](#接口用例设计清单)
9. [常见问题](#常见问题)

---

## 为什么接口测试价值高

| 对比 | 接口测试 | UI 测试 |
|------|---------|---------|
| 速度 | 秒级 | 分钟级 |
| 稳定性 | 高 | 易 flaky |
| 成本 | 低 | 高 |
| 覆盖业务规则 | 直接 | 间接 |
| 定位问题 | 精确到接口 | 需层层排查 |

**定位**：UI 测试验证"用户能不能完成操作"，接口测试验证"业务规则是否正确"。两者互补，接口测试是主力。

---

## 基础：requests 客户端

```python
import requests

BASE_URL = "https://api.example.com"

# GET
resp = requests.get(f"{BASE_URL}/users/1")
print(resp.status_code, resp.json())

# POST（JSON）
resp = requests.post(f"{BASE_URL}/login", json={
    "username": "tester",
    "password": "123456"
})
print(resp.status_code, resp.json())

# 带请求头 / 参数
resp = requests.get(f"{BASE_URL}/search",
    params={"q": "pytest", "page": 1},
    headers={"Authorization": "Bearer xxx"}
)
```

---

## 接口测试结构

**推荐分层**（工程化详见 [[7-Test-测试框架工程化]]）：

```
tests/
├── conftest.py          # 共享 fixture（客户端、鉴权）
├── api/                 # 接口封装层
│   ├── client.py        # 请求封装
│   └── endpoints.py     # 各接口定义
└── test_login.py        # 用例层
```

**接口封装示例**（避免用例里散落 URL 和参数）：

```python
# api/client.py
import requests

class APIClient:
    def __init__(self, base_url, token=None):
        self.base_url = base_url
        self.token = token

    def request(self, method, path, **kwargs):
        headers = kwargs.pop("headers", {})
        if self.token:
            headers["Authorization"] = f"Bearer {self.token}"
        return requests.request(method, f"{self.base_url}{path}",
                                headers=headers, **kwargs)

    def login(self, username, password):
        return self.request("POST", "/login",
                            json={"username": username, "password": password})
```

**用例示例**：

```python
# test_login.py
import pytest

@pytest.fixture
def client():
    from api.client import APIClient
    return APIClient(BASE_URL)

def test_login_success(client):
    resp = client.login("tester", "123456")
    assert resp.status_code == 200
    data = resp.json()
    assert data["code"] == 0
    assert data["data"]["token"]          # 有 token
    assert data["data"]["username"] == "tester"
```

---

## 数据驱动

用参数化批量验证多种输入（进阶用法见 [[3-Test-pytest进阶]]）：

```python
@pytest.mark.parametrize("username,password,expected_code", [
    pytest.param("tester", "123456", 0, id="正常"),
    pytest.param("", "123456", 1001, id="用户名为空"),
    pytest.param("tester", "", 1002, id="密码为空"),
    pytest.param("nobody", "wrong", 1003, id="账号不存在"),
])
def test_login_cases(client, username, password, expected_code):
    resp = client.login(username, password)
    assert resp.json()["code"] == expected_code
```

### 数据文件化（大数据集）

```python
# data/login_cases.json
# [
#   {"username": "tester", "password": "123456", "expected": 0},
#   ...
# ]

import json
import pytest

with open("data/login_cases.json", encoding="utf-8") as f:
    LOGIN_CASES = json.load(f)

@pytest.mark.parametrize("case", LOGIN_CASES, ids=lambda c: c.get("id", ""))
def test_login_ddt(client, case):
    resp = client.login(case["username"], case["password"])
    assert resp.json()["code"] == case["expected"]
```

> 💡 数据与代码分离，改测试数据不需要改代码。这是数据驱动（DDT）的核心价值。

---

## 环境管理

不同环境（开发/测试/预发布/生产）用不同地址，配置化：

```python
# config.py
ENVIRONMENTS = {
    "dev":  "https://dev.example.com",
    "test": "https://test.example.com",
    "staging": "https://staging.example.com",
}

ENV = ENVIRONMENTS.get(os.getenv("TEST_ENV", "test"))
```

```bash
# 运行时切换
TEST_ENV=staging pytest
```

**更优雅**：pytest 命令行参数 + fixture（见 [[7-Test-测试框架工程化]] 的配置管理）。

---

## 鉴权处理

### 登录获取 token，全局复用

```python
# conftest.py
import pytest

@pytest.fixture(scope="session")
def token():
    """整个会话复用同一个登录 token"""
    resp = requests.post(f"{BASE_URL}/login",
                         json={"username": "admin", "password": "admin123"})
    return resp.json()["data"]["token"]

@pytest.fixture
def auth_client(token):
    return APIClient(BASE_URL, token=token)
```

### 用例里灵活控制

```python
def test_access_without_token(client):
    resp = client.request("GET", "/admin/users")
    assert resp.status_code == 401          # 未鉴权应拒绝

def test_access_with_token(auth_client):
    resp = auth_client.request("GET", "/admin/users")
    assert resp.status_code == 200
```

---

## 断言设计

**四层断言**，从粗到细：

| 层 | 验证内容 | 示例 |
|----|---------|------|
| 1. 状态码 | HTTP 层 | `assert resp.status_code == 200` |
| 2. 业务码 | 业务层 | `assert data["code"] == 0` |
| 3. 核心字段 | 关键数据 | `assert data["data"]["token"]` |
| 4. 契约结构 | 完整结构 | 校验 schema（如 jsonschema） |

```python
import jsonschema

def test_response_schema():
    resp = client.get_order("123")
    schema = {
        "type": "object",
        "required": ["order_no", "amount", "status"],
        "properties": {
            "order_no": {"type": "string"},
            "amount": {"type": "number"},
            "status": {"type": "string"}
        }
    }
    jsonschema.validate(resp.json(), schema)
```

> 💡 用 schema 校验接口契约，改接口字段时测试会立刻报警——这是**契约测试**的核心思想。

---

## 接口用例设计清单

针对每个接口，覆盖这些维度：

| 维度 | 用例 |
|------|------|
| 正常路径 | 合法参数返回预期结果 |
| 参数校验 | 缺失、类型错误、越界、空值 |
| 边界值 | 最大值、最小值、0、负数 |
| 鉴权 | 无 token、过期 token、错误 token |
| 权限 | 普通用户访问管理员接口 |
| 状态流转 | 依赖前置状态的接口 |
| 幂等性 | 重复调用结果一致 |
| 异常 | 404 资源不存在、500 服务异常 |

---

## 常见问题

**Q：接口测试依赖前置数据怎么办？**
A：fixture 前置创建（用户/订单），或用接口自身造数据；避免依赖 UI 操作。

**Q：测试环境数据互相污染？**
A：每个用例独立数据（随机后缀）；或用事务/回滚；参数化时注意隔离。

**Q：token 过期导致测试偶发失败？**
A：session 级 fixture 拉长过期时间；或做 token 自动刷新逻辑。

**Q：接口测试和 UI 测试怎么选？**
A：业务规则走接口测试（快而稳）；端到端用户路径才用 UI 测试（见 [[1-Test-测试方法论核心]] 金字塔）。

---

## 相关笔记

- 接口测试的基础框架 → [[2-Test-pytest入门]]、[[3-Test-pytest进阶]]
- 工程化：Page Object / 配置管理 / CI → [[7-Test-测试框架工程化]]
- 质量指标 → [[1-Test-测试方法论核心]]
- 与移动端 UI 测试融合 → [[5-Test-Appium基础]]
