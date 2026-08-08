# 1. pytest 入门

> pytest 是 Python 最流行的测试框架：简洁、强大、生态丰富。本笔记覆盖从安装到第一个用例、断言、fixture 基础与运行方式。

---

## 目录

1. [安装](#安装)
2. [第一个测试](#第一个测试)
3. [运行测试](#运行测试)
4. [断言](#断言)
5. [fixture 基础](#fixture-基础)
6. [异常测试](#异常测试)
7. [跳过与预期失败](#跳过与预期失败)
8. [常见问题](#常见问题)

---

## 安装

```bash
pip install pytest
```

验证：

```bash
pytest --version
```

常用配套插件（进阶见 [[3-Test-pytest进阶]]）：

```bash
pip install pytest-cov          # 覆盖率
pip install pytest-html         # HTML 报告
pip install pytest-xdist        # 并行执行
pip install requests            # 接口测试用
```

---

## 第一个测试

pytest 的约定：**文件名以 `test_` 开头，函数名以 `test_` 开头**。

```python
# test_math.py
def test_addition():
    assert 1 + 1 == 2

def test_string():
    assert "hello".upper() == "HELLO"
```

运行：

```bash
pytest                  # 自动发现 test_*.py / *_test.py
pytest test_math.py     # 指定文件
pytest -v               # 详细输出
```

**发现规则**：
- 文件：`test_*.py` 或 `*_test.py`
- 函数：`test_*`
- 类：`Test*` 中的方法

---

## 运行测试

```bash
pytest                      # 全部
pytest -v                   # 详细（每个用例一行）
pytest -k "user"            # 按名称筛选
pytest -k "not slow"        # 排除
pytest test_a.py::test_func # 指定单个用例
pytest -x                   # 第一个失败即停止
pytest --maxfail=3          # 最多失败 3 个后停止
pytest -s                   # 显示 print 输出
pytest -q                   # 静默模式
pytest --tb=short           # 简短堆栈
pytest -m slow              # 按 mark 筛选（见 [2-pytest进阶]）
```

---

## 断言

pytest 用 Python 原生 `assert`，失败时自动给出详细对比信息：

```python
def test_examples():
    assert 5 > 3
    assert "pytest" in "pytest is great"
    assert {"a": 1} == {"a": 1}

    result = {"status": "ok", "items": [1, 2, 3]}
    assert result["status"] == "ok"
    assert len(result["items"]) == 3
    assert 2 in result["items"]
```

**断言失败信息优化**：`assert` 后加描述字符串：

```python
def test_login():
    resp = login("user", "wrong_pw")
    assert resp["code"] == 200, f"期望 200，实际 {resp['code']}: {resp['msg']}"
```

---

## fixture 基础

fixture 是 pytest 的**依赖注入**机制：把前置准备/清理逻辑抽出来复用。

```python
import pytest

@pytest.fixture
def user():
    """每个用例独立的测试用户"""
    u = create_user(name="test_user")   # 前置
    yield u                              # 提供给用例
    delete_user(u.id)                    # 清理

def test_user_info(user):
    assert user.name == "test_user"
```

### fixture 的三个要点

1. **用例参数名 = fixture 名**，pytest 自动注入
2. `yield` 之前的代码是"前置"，之后的代码是"清理"
3. 不写 `yield`（直接 `return`）则无清理逻辑

### 多个 fixture

```python
def test_order(user, db):
    order = db.create_order(user.id)
    assert order is not None
```

> fixture 进阶（作用域、工厂、参数化、conftest 共享）见 [[3-Test-pytest进阶]]。

---

## 异常测试

用 `pytest.raises` 验证"应该抛异常"的场景：

```python
import pytest

def divide(a, b):
    if b == 0:
        raise ValueError("除数不能为零")
    return a / b

def test_divide_by_zero():
    with pytest.raises(ValueError) as exc_info:
        divide(10, 0)
    assert "零" in str(exc_info.value)   # 验证异常信息

def test_divide_ok():
    assert divide(10, 2) == 5
```

---

## 跳过与预期失败

### 跳过（不满足条件时跳过）

```python
import pytest

@pytest.mark.skip(reason="功能未实现")
def test_pending():
    pass

@pytest.mark.skipif(sys.version_info < (3, 9), reason="需要 Python 3.9+")
def test_new_api():
    pass
```

### 预期失败（已知 bug，先标记）

```python
@pytest.mark.xfail(reason="已知问题 #123，待修复")
def test_known_bug():
    assert complex_function() == expected
```

- xfail 用例失败 → 报告 XFAIL（不阻塞）
- xfail 用例反而通过 → 报告 XPASS（提醒你该移除标记了）

---

## 常见问题

**Q：为什么 pytest 找不到我的测试？**
A：检查文件名/函数名是否以 `test_` 开头；确认在正确的目录运行。

**Q：`assert` 失败信息看不懂？**
A：加描述字符串 `assert cond, "期望...实际..."`；或用 `-vv` 看更多上下文。

**Q：测试之间有共享状态怎么办？**
A：用 fixture 隔离；每个用例创建独立数据（见 [[1-Test-测试方法论核心]] 的"确定性"原则）。

**Q：怎么测接口/API？**
A：pytest + requests 是标配，见 [[4-Test-接口测试实践]]。

---

## 下一步

- fixture/参数化/mark 等进阶能力 → [[3-Test-pytest进阶]]
- 接口测试实战 → [[4-Test-接口测试实践]]
- 搭完整工程化框架 → [[7-Test-测试框架工程化]]
- 完整手册导航 → [[0-自动化测试方法论]]
