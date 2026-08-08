# 6. 测试框架工程化

> 从"散落的用例"到"可维护的测试框架"：目录分层、Page Object 模式、pytest 编排 Appium、配置管理、报告与 CI/CD 接入。这是方法论真正落地的最后一环。

---

## 目录

1. [目录结构设计](#目录结构设计)
2. [Page Object 模式](#page-object-模式)
3. [用 pytest 编排 Appium](#用-pytest-编排-appium)
4. [配置管理](#配置管理)
5. [日志与报告](#日志与报告)
6. [失败截图与视频](#失败截图与视频)
7. [CI/CD 集成](#cicd-集成)
8. [多设备并行](#多设备并行)
9. [框架演进路线](#框架演进路线)

---

## 目录结构设计

推荐的分层结构（pytest + Appium 融合）：

```
project/
├── conftest.py              # 全局 fixture（driver、客户端、环境）
├── pytest.ini               # 配置（见 [2-pytest进阶]）
├── requirements.txt
├── config/                  # 配置
│   ├── config.py            # 环境/URL/capabilities
│   └── settings.json
├── api/                     # 接口封装层（[[4-Test-接口测试实践]]）
│   ├── client.py
│   └── endpoints.py
├── pages/                   # Page Object 层（移动端）
│   ├── base_page.py
│   ├── login_page.py
│   └── home_page.py
├── testdata/                # 测试数据
│   ├── login_cases.json
│   └── users.py
├── utils/                   # 工具
│   ├── logger.py
│   └── screenshot.py
└── tests/                   # 用例层
    ├── test_api_login.py
    ├── test_app_login.py
    └── test_app_order.py
```

**依赖方向（单向）**：用例层 → Page/API 层 → utils/config。禁止用例直接散落定位器和 URL。

---

## Page Object 模式

**核心思想**：把"页面"抽象成对象——元素定位 + 页面操作封装在类里，用例只描述业务流程。

```python
# pages/login_page.py
from appium.webdriver.common.appiumby import AppiumBy

class LoginPage:
    def __init__(self, driver):
        self.driver = driver

    def input_username(self, text):
        el = self.driver.find_element(AppiumBy.ID, "com.example.app:id/username")
        el.send_keys(text)

    def input_password(self, text):
        el = self.driver.find_element(AppiumBy.ID, "com.example.app:id/password")
        el.send_keys(text)

    def click_login(self):
        self.driver.find_element(AppiumBy.ID, "com.example.app:id/login_btn").click()

    def login(self, username, password):
        """组合操作：一次登录"""
        self.input_username(username)
        self.input_password(password)
        self.click_login()
        from pages.home_page import HomePage
        return HomePage(self.driver)
```

**用例层只描述业务**：

```python
# tests/test_app_login.py
def test_login_success(driver):
    login_page = LoginPage(driver)
    home = login_page.login("tester", "123456")
    assert home.title_is_displayed()
```

### Page Object 的好处

1. **定位器集中**：UI 变了只改 Page 类，不改用例
2. **复用**：登录逻辑一处定义，处处使用
3. **可读**：用例像业务剧本，非技术细节
4. **职责清晰**：页面对象管"页面"，用例管"流程"

---

## 用 pytest 编排 Appium

Appium 用 pytest 的 **fixture 管理 driver 生命周期**：

```python
# conftest.py
import pytest
from appium import webdriver

@pytest.fixture(scope="function")
def driver():
    """每个用例一个独立 driver（互不干扰）"""
    caps = load_capabilities()          # 从 config 读取
    drv = webdriver.Remote("http://localhost:4723/wd/hub", caps)
    drv.implicitly_wait(10)
    yield drv
    drv.quit()
```

**复用场景**：登录只做一次，后续用例复用（session 级）：

```python
@pytest.fixture(scope="session")
def app_driver():
    """整个会话共用一个 driver（适合冒烟流程链）"""
    drv = webdriver.Remote("http://localhost:4723/wd/hub", load_capabilities())
    yield drv
    drv.quit()
```

> ⚠️ 一个 driver 跑多个用例时，用例间状态会串（页面停留位置不同）。推荐**每条用例独立 driver**，必要时用 `driver.reset()`。

---

## 配置管理

统一管理环境、URL、capabilities，避免散落魔法值：

```python
# config/config.py
import json, os

ENVIRONMENTS = {
    "dev":  {"api": "https://dev.example.com", "caps": {...}},
    "test": {"api": "https://test.example.com", "caps": {...}},
}

def get_env():
    return os.getenv("TEST_ENV", "test")

def load_capabilities():
    return ENVIRONMENTS[get_env()]["caps"]

def get_api_base():
    return ENVIRONMENTS[get_env()]["api"]
```

**pytest 命令行传参**（更工程化）：

```python
# conftest.py
def pytest_addoption(parser):
    parser.addoption("--env", action="store", default="test", help="环境: dev/test/staging")

@pytest.fixture
def env(request):
    return request.config.getoption("--env")
```

```bash
pytest --env=staging
```

---

## 日志与报告

### 日志

```python
# utils/logger.py
import logging

logging.basicConfig(level=logging.INFO,
                    format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger("test")
```

在用例/Page 中记录关键步骤，失败时便于回溯：

```python
logger.info(f"登录成功: {username}")
logger.error(f"断言失败，实际返回: {resp.text}")
```

### 报告

```bash
pip install pytest-html allure-pytest

# HTML 报告
pytest --html=report.html --self-contained-html

# Allure 报告
pytest --alluredir=./allure-results
allure serve ./allure-results
```

| 报告 | 特点 |
|------|------|
| pytest-html | 简单、单文件、即装即用 |
| Allure | 功能强、趋势图、失败分类、与环境/CI 集成好 |

---

## 失败截图与视频

**失败自动截图**（定位问题必备）：

```python
# conftest.py
import pytest

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    report = outcome.get_result()
    if report.when == "call" and report.failed:
        driver = item.funcargs.get("driver")
        if driver:
            driver.save_screenshot(f"screenshots/{item.name}.png")
```

**录屏**（Appium 支持）：

```python
import base64

# 开始录制
driver.start_recording_screen()

# 结束并保存
video = driver.stop_recording_screen()
with open("video.mp4", "wb") as f:
    f.write(base64.b64decode(video))
```

> 💡 截图 + page_source + 日志三件套，是排查 flaky 和定位器问题的黄金组合。

---

## CI/CD 集成

让每次提交自动跑测试（配合 [[0-Git 使用说明]]）：

```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]

jobs:
  unit-and-api:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements.txt
      - run: pytest tests/test_api* -m "not ui" --cov=api --cov-report=html

  ui:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements.txt
      - run: npm install -g appium
      - run: appium driver install uiautomator2
      - uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 33
          script: appium & sleep 5 && pytest tests/test_app* -m smoke
```

**CI 要点**：
- 单元/接口测试每次提交都跑
- UI（Appium）测试在关键分支/发布前跑（慢）
- 用 mark 分套件：`-m "not ui"`（快）vs `-m smoke`（冒烟）
- 产物上传：报告、截图、日志

---

## 多设备并行

```bash
pip install pytest-xdist

# 同一批用例分发到多设备（配合参数化设备）
pytest -n 4 --dist loadscope
```

**多设备参数化**：

```python
@pytest.mark.parametrize("device", [
    {"deviceName": "emulator-5554", "platformVersion": "13"},
    {"deviceName": "emulator-5556", "platformVersion": "12"},
])
@pytest.fixture
def driver(request):
    caps = load_capabilities()
    caps.update(request.param)
    drv = webdriver.Remote("http://localhost:4723/wd/hub", caps)
    yield drv
    drv.quit()
```

> ⚠️ 并行需保证用例**无共享状态**（独立数据），否则相互干扰。设备资源充足才并行。

---

## 框架演进路线

```
第一阶段：单层跑通
  写测试 → 断言 → 本地能跑

第二阶段：分层封装
  引入 config / utils / conftest 共享

第三阶段：Page Object + 接口封装
  定位器集中、业务流封装、数据驱动

第四阶段：工程化
  日志、报告、截图、失败自动截图

第五阶段：CI/CD + 并行
  自动执行、多设备、结果可视化
```

> 每一阶段都是前一段的稳定化，**不要一上来就搭最重的框架**。

---

## 相关笔记

- 分层方法论总纲 → [[1-Test-测试方法论核心]]
- pytest 进阶能力 → [[3-Test-pytest进阶]]
- 接口层封装 → [[4-Test-接口测试实践]]
- Appium 定位与交互 → [[6-Test-Appium元素定位与交互]]
- 版本管理与 CI 协作 → [[0-Git 使用说明]]
- 工程纪律与验证 → [[superpowers-使用说明]]
