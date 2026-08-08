# 4. Appium 基础

> Appium 是跨平台移动端 UI 自动化框架，支持 Android / iOS / Web / 桌面，Python 客户端可与本套方法论的 pytest 无缝融合。本笔记覆盖架构、环境、Android 配置与第一个用例。

---

## 目录

1. [Appium 是什么](#appium-是什么)
2. [架构与工作原理](#架构与工作原理)
3. [环境准备（Android）](#环境准备android)
4. [安装 Appium 与客户端](#安装-appium-与客户端)
5. [Desired Capabilities](#desired-capabilities)
6. [第一个 Appium 用例](#第一个-appium-用例)
7. [常用 API 概览](#常用-api-概览)
8. [常见问题](#常见问题)

---

## Appium 是什么

- **跨平台**：同一套 API 测 Android、iOS、Web、桌面（WinAppDriver）
- **不修改被测应用**：通过协议驱动原生/混合应用
- **语言无关**：Python / Java / JS / Ruby 等
- 本套方法论选择 **Python + pytest**（与单元/接口测试统一）

---

## 架构与工作原理

```
测试脚本（pytest + Appium Python Client）
        │  WebDriver 协议（HTTP）
        ▼
    Appium Server
        │  ←→ 平台驱动（UiAutomator2 / XCUITest / ...）
        ▼
    真机 / 模拟器（被测 App）
```

| 组件 | 作用 |
|------|------|
| Appium Server | 接收测试指令，转发给平台驱动 |
| Appium Client（python-client） | 测试脚本侧的客户端库 |
| UiAutomator2 | Android 平台驱动（推荐） |
| XCUITest | iOS 平台驱动 |
| Android SDK / adb | 与 Android 设备/模拟器通信 |

---

## 环境准备（Android）

### 1. 安装 Java（JDK 11+）

Appium Server 2.x 需要 Java。验证：`java -version`

### 2. 安装 Android SDK 与 adb

安装 Android Studio（自带 SDK），或用命令行工具：

```bash
# 验证 adb 可用
adb --version

# 启动 Android 模拟器（或连接真机）
adb devices          # 应能看到设备
```

### 3. 配置环境变量

- `ANDROID_HOME` → Android SDK 目录
- `PATH` 加入 `platform-tools`（adb）、`emulator` 等

---

## 安装 Appium 与客户端

### 方式一：npm 全局（推荐）

```bash
npm install -g appium
appium --version        # 验证

# 安装 Android 驱动
appium driver install uiautomator2
```

启动服务：

```bash
appium            # 默认 0.0.0.0:4723
```

### 方式二：Appium Inspector（定位元素用）

从官网下载 Appium Inspector，用于可视化查看控件树、调试定位器（配合 [[6-Test-Appium元素定位与交互]]）。

### 安装 Python 客户端

```bash
pip install Appium-Python-Client
```

---

## Desired Capabilities

告诉 Appium 要连什么设备、开什么应用的关键配置：

```python
desired_caps = {
    "platformName": "Android",
    "deviceName": "emulator-5554",
    "platformVersion": "13",
    "appPackage": "com.example.app",        # 应用包名
    "appActivity": ".MainActivity",         # 启动 Activity
    "noReset": True,                        # 不重置应用数据
    "automationName": "UiAutomator2",
    "unicodeKeyboard": True,                # 支持中文输入
    "resetKeyboard": True,
}
```

| 关键 Capability | 作用 |
|----------------|------|
| `platformName` | 平台：Android / iOS |
| `deviceName` | 设备标识（模拟器名或真机） |
| `appPackage` / `appActivity` | 指定要启动的 App |
| `app` | 或直接指定 APK 路径 |
| `automationName` | 驱动：UiAutomator2 / XCUITest |
| `noReset` | 是否保留应用数据 |
| `unicodeKeyboard` | 支持非 ASCII 输入 |
| `udid` | 指定真机（多设备时） |

> 各平台完整 capability 列表见 Appium 官方文档。

---

## 第一个 Appium 用例

```python
# test_app_login.py
from appium import webdriver
from appium.webdriver.common.appiumby import AppiumBy

desired_caps = {
    "platformName": "Android",
    "deviceName": "emulator-5554",
    "platformVersion": "13",
    "appPackage": "com.example.app",
    "appActivity": ".MainActivity",
    "noReset": True,
    "automationName": "UiAutomator2",
}

def test_login():
    driver = webdriver.Remote("http://localhost:4723/wd/hub", desired_caps)
    try:
        # 输入用户名
        username = driver.find_element(AppiumBy.ID, "com.example.app:id/username")
        username.send_keys("tester")

        # 输入密码
        driver.find_element(AppiumBy.ID, "com.example.app:id/password").send_keys("123456")

        # 点击登录
        driver.find_element(AppiumBy.ID, "com.example.app:id/login_btn").click()

        # 断言：登录成功跳转
        assert driver.find_element(AppiumBy.ID, "com.example.app:id/home_title").is_displayed()
    finally:
        driver.quit()
```

---

## 常用 API 概览

```python
# 定位（完整策略见 [5-Appium元素定位与交互]）
driver.find_element(AppiumBy.ID, "id")
driver.find_element(AppiumBy.XPATH, "//android.widget.TextView[@text='登录']")

# 交互
element.click()
element.send_keys("text")
element.clear()

# 属性
element.text
element.is_displayed()
element.get_attribute("content-desc")

# 等待
driver.implicitly_wait(10)              # 隐式等待（全局）
WebDriverWait(driver, 10).until(
    EC.presence_of_element_located((AppiumBy.ID, "id"))
)

# 手势 / 滑屏（详见 [5-Appium元素定位与交互]）
driver.swipe(start_x, start_y, end_x, end_y, duration)

# 返回 / 切应用
driver.back()
driver.background_app(5)
```

---

## 常见问题

**Q：`appium` 命令找不到？**
A：确认 `npm install -g appium` 成功，node 全局 bin 在 PATH。

**Q：adb 看不到设备？**
A：模拟器需先启动；真机需开启 USB 调试并授权。

**Q：`noReset` 设了但数据还是被重置？**
A：检查模拟器是否在 App 安装前就存在旧数据；部分环境需要 `--udid` 精确指定设备。

**Q：中文输入乱码？**
A：`unicodeKeyboard` 和 `resetKeyboard` 都设为 `True`。

**Q：找不到元素？**
A：多半是等待不足或定位器不对。先看 [[6-Test-Appium元素定位与交互]] 的等待策略和定位器调试。

---

## 下一步

- 元素定位、等待、手势等交互核心 → [[6-Test-Appium元素定位与交互]]
- 用 pytest 编排 Appium 用例 + Page Object → [[7-Test-测试框架工程化]]
- 稳定性与多设备 → [[8-Test-最佳实践与FAQ]]
- 完整手册导航 → [[0-自动化测试方法论]]
