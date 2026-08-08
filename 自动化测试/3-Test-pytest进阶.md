# 2. pytest 进阶

> 从"会写用例"到"能搭测试框架"的进阶能力：fixture 作用域与工厂、参数化、mark、conftest、插件与 hook。

---

## 目录

1. [fixture 作用域](#fixture-作用域)
2. [fixture 工厂](#fixture-工厂)
3. [fixture 参数化](#fixture-参数化)
4. [参数化 paramize](#参数化-paramize)
5. [mark 标记](#mark-标记)
6. [conftest 共享](#conftest-共享)
7. [常用插件](#常用插件)
8. [hook 函数](#hook-函数)
9. [配置文件 pytest.ini](#配置文件-pytestini)

---

## fixture 作用域

fixture 可以控制创建/销毁的时机：

```python
import pytest

@pytest.fixture(scope="session")
def app_config():
    """整个测试会话只创建一次（如读取配置）"""
    return load_config()

@pytest.fixture(scope="module")
def db():
    """每个模块创建一次（如连接数据库）"""
    conn = connect()
    yield conn
    conn.close()

@pytest.fixture(scope="function")   # 默认：每个用例都新建
def user():
    return create_user()
```

| scope | 时机 | 适用 |
|-------|------|------|
| `function`（默认） | 每个用例 | 大部分临时数据 |
| `class` | 每个测试类 | 类级共享 |
| `module` | 每个模块 | 数据库连接、读取配置 |
| `session` | 整个会话 | 昂贵的全局资源 |

> ⚠️ **session/module 级 fixture 要小心**：用例间共享状态，若用例修改了共享资源会影响其他用例。只放"只读/可重置"资源。

---

## fixture 工厂

fixture 返回一个"函数"，调用时才真正创建数据：

```python
@pytest.fixture
def make_user():
    """工厂式 fixture：每次调用创建不同数据"""
    created = []

    def _make_user(name, age=20):
        u = User.create(name=name, age=age)
        created.append(u)
        return u

    yield _make_user

    for u in created:       # 清理全部
        u.delete()

def test_user_cases(make_user):
    alice = make_user("alice")
    bob = make_user("bob", age=30)
    assert alice.age != bob.age
```

**优点**：同一个 fixture 能创建多种数据，用例更灵活。

---

## fixture 参数化

同一 fixture 在不同用例用不同参数：

```python
@pytest.fixture(params=["admin", "user", "guest"])
def role(request):
    """每个参数跑一遍"""
    return Role(request.param)
```

配合 `request.param` 访问当前参数值。适合"同一套流程测多组数据"。

---

## 参数化 paramize

最常用的数据驱动方式——一条用例跑多组数据：

```python
import pytest

@pytest.mark.parametrize("a,b,expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
])
def test_add(a, b, expected):
    assert add(a, b) == expected
```

### 命名参数（更好定位失败）

```python
@pytest.mark.parametrize("username,password,expected", [
    pytest.param("user1", "123456", 200, id="正常登录"),
    pytest.param("", "123456", 400, id="用户名为空"),
    pytest.param("user1", "", 400, id="密码为空"),
])
def test_login(username, password, expected):
    assert api_login(username, password)["code"] == expected
```

### 叠加参数化（笛卡尔积）

```python
@pytest.mark.parametrize("role", ["admin", "user"])
@pytest.mark.parametrize("endpoint", ["/users", "/orders"])
def test_access(role, endpoint):
    ...
```

> 参数化是接口测试的核心手段，实战见 [[4-Test-接口测试实践]]。

---

## mark 标记

给用例打标签，用于分类和筛选：

```python
import pytest

@pytest.mark.smoke
def test_core_login():
    ...

@pytest.mark.slow
def test_heavy_job():
    ...
```

```bash
pytest -m smoke          # 只跑 smoke
pytest -m "not slow"     # 排除 slow
```

**注册自定义 mark**（避免 warning），在 `pytest.ini` 中：

```ini
[pytest]
markers =
    smoke: 冒烟测试
    slow: 慢用例
    api: 接口测试
```

内置 mark：`skip`、`skipif`、`xfail`（见 [[2-Test-pytest入门]]）。

---

## conftest 共享

`conftest.py` 是**共享 fixture 和 hook** 的载体，无需 import 即可被同目录及子目录的测试使用。

```
project/
├── conftest.py            # 全局共享
├── tests/
│   ├── test_a.py
│   └── sub/
│       ├── conftest.py    # sub 目录局部共享
│       └── test_b.py
```

```python
# conftest.py
import pytest

@pytest.fixture
def api_client():
    """所有测试共用的 API 客户端"""
    return APIClient(base_url=TEST_URL)

@pytest.fixture
def logged_in(api_client):
    api_client.login("tester", "secret")
    return api_client
```

**作用域规则**：越靠近用例的 conftest 优先；子目录 conftest 覆盖父目录同名 fixture。

---

## 常用插件

| 插件 | 作用 |
|------|------|
| `pytest-cov` | 覆盖率统计（见 [[1-Test-测试方法论核心]]） |
| `pytest-html` | HTML 报告 |
| `pytest-xdist` | 并行执行（`pytest -n auto`） |
| `pytest-timeout` | 用例超时控制 |
| `pytest-rerunfailures` | 失败重跑（处理 flaky，慎用） |
| `pytest-mock` | mock 集成（unittest.mock 封装） |
| `pytest-ordering` | 控制执行顺序（少用） |

```bash
pip install pytest-cov pytest-html pytest-xdist pytest-timeout pytest-rerunfailures
```

```bash
pytest --cov=src --cov-report=html   # 覆盖率 + HTML 报告
pytest -n 4                          # 4 进程并行
pytest --timeout=30                  # 单用例超时 30s
pytest --html=report.html            # HTML 报告
```

---

## hook 函数

hook 让你在 pytest 生命周期"插手"。最常用的是 conftest 中的 `pytest_collection_modifyitems`（按 mark 排序）和 `pytest_configure`：

```python
# conftest.py
def pytest_collection_modifyitems(config, items):
    """把 smoke 用例排到前面先跑"""
    items.sort(key=lambda item: "smoke" not in item.keywords)
```

```python
def pytest_configure(config):
    """会话开始时注入自定义信息"""
    config.option.test_url = "https://staging.example.com"
```

> hook 是高级能力，日常主要用 fixture + 插件即可。完整 hook 列表见 pytest 官方文档。

---

## 配置文件 pytest.ini

统一项目级配置：

```ini
[pytest]
# 指定测试目录与匹配规则
testpaths = tests
python_files = test_*.py *_test.py
python_functions = test_*

# 默认命令行参数
addopts = -v --tb=short -q

# 自定义 mark
markers =
    smoke: 冒烟
    api: 接口
    slow: 慢用例

# 过滤 warning
filterwarnings =
    ignore::DeprecationWarning
```

---

## 相关笔记

- 基础概念 → [[2-Test-pytest入门]]
- 接口测试实战 → [[4-Test-接口测试实践]]
- 工程化与 Page Object → [[7-Test-测试框架工程化]]
- 覆盖率与质量指标 → [[1-Test-测试方法论核心]]
