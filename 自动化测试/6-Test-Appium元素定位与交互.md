# 5. Appium 元素定位与交互

> UI 自动化的核心能力：找到元素、等待元素、操作元素。本章是 Appium 用例稳定性与效率的关键——定位策略选得好，用例就稳。

---

## 目录

1. [元素定位策略](#元素定位策略)
2. [定位器优先级建议](#定位器优先级建议)
3. [使用 Appium Inspector 调试](#使用-appium-inspector-调试)
4. [等待策略](#等待策略)
5. [常见交互操作](#常见交互操作)
6. [手势与滑屏](#手势与滑屏)
7. [多平台注意事项](#多平台注意事项)
8. [WebView 与混合应用](#webview-与混合应用)
9. [定位常见问题](#定位常见问题)

---

## 元素定位策略

Appium 常用定位方式（`AppiumBy`）：

```python
from appium.webdriver.common.appiumby import AppiumBy

# 1. ID（Android: resource-id）
driver.find_element(AppiumBy.ID, "com.example.app:id/username")

# 2. XPATH（最通用）
driver.find_element(AppiumBy.XPATH, "//android.widget.TextView[@text='登录']")
driver.find_element(AppiumBy.XPATH, "//android.widget.Button[@resource-id='login_btn']")

# 3. 文本（text）
driver.find_element(AppiumBy.ANDROID_UIAUTOMATOR,
                    'new UiSelector().text("登录")')

# 4. 描述（content-desc，无障碍）
driver.find_element(AppiumBy.ACCESSIBILITY_ID, "login_button")

# 5. 类名（class name）
driver.find_element(AppiumBy.CLASS_NAME, "android.widget.Button")

# 6. 复数：找到多个
elements = driver.find_elements(AppiumBy.CLASS_NAME, "android.widget.TextView")
```

**Android UiAutomator2 额外定位器**：

```python
# 包含文本
'new UiSelector().textContains("登录")'
# 正则匹配
'new UiSelector().textMatches(".*[登].*")'
# 索引（少用）
'new UiSelector().className("android.widget.Button").index(2)'
```

---

## 定位器优先级建议

| 优先级 | 方式 | 理由 |
|--------|------|------|
| 1 | ID / resource-id | 稳定、唯一、快 |
| 2 | accessibility-id / content-desc | 稳定，且利于无障碍 |
| 3 | 文本（text） | 直观，但可能重复/国际化 |
| 4 | XPATH（相对路径） | 通用，但脆（结构变化易失效） |
| 5 | 类名 / 索引 | 极易失效，尽量不用 |

> 💡 **能用 ID 就不用 XPATH**。XPATH 越具体越脆弱，UI 一改就崩。保留稳定的属性组合，避免依赖完整层级路径。

---

## 使用 Appium Inspector 调试

Appium Inspector 是定位元素的可视化工具：

1. 启动 Appium Server + 模拟器
2. 打开 Appium Inspector，填入与测试相同的 Desired Capabilities
3. 连接后可视化查看控件树（XML）
4. 点击控件看到其属性（resource-id、text、content-desc）
5. 复制/验证定位器（支持 ID、XPATH 等）

**调试技巧**：
- 先看控件树，找稳定属性
- 用 `driver.page_source` 导出当前页面 XML 排查
- 定位失败时，截图 + 打印 page_source 定位问题

```python
driver.save_screenshot("/tmp/error.png")     # 失败时截图
print(driver.page_source)                    # 打印当前控件树
```

---

## 等待策略

**UI 自动化 90% 的 flaky 源于"元素还没出现就操作"**。等待是稳定性的核心。

### 隐式等待（全局设置）

```python
driver.implicitly_wait(10)   # 每次 find_element 最多等 10 秒
```

### 显式等待（推荐，精确控制）

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

wait = WebDriverWait(driver, 10)

# 等待元素可见
element = wait.until(EC.visibility_of_element_located((AppiumBy.ID, "id")))

# 等待元素可点击
wait.until(EC.element_to_be_clickable((AppiumBy.ID, "login_btn")))

# 等待元素消失（如 loading）
wait.until(EC.invisibility_of_element_located((AppiumBy.ID, "loading")))
```

### 等待要点

1. **等待"期望状态"**（可点击、可见），而不是固定 `time.sleep`
2. 点击前等 `element_to_be_clickable`，输入前等 `visibility_of_element_located`
3. 轮询间隔与超时可配：`WebDriverWait(driver, 10, poll_frequency=0.5)`

---

## 常见交互操作

```python
element.click()                # 点击
element.send_keys("text")      # 输入
element.clear()                # 清空
element.get_attribute("text")  # 取属性
element.is_displayed()         # 是否可见
element.is_enabled()           # 是否可用

# 键盘
driver.press_keycode(66)       # Enter（Android 键码）
driver.hide_keyboard()         # 隐藏键盘

# 页面
driver.back()                  # 返回
driver.current_activity        # 当前 Activity
driver.get_window_size()       # 窗口尺寸
```

---

## 手势与滑屏

```python
from appium.webdriver.common.touch_action import TouchAction

# 简单滑动
driver.swipe(x1, y1, x2, y2, duration_ms)

# 按坐标滑动（基于窗口尺寸计算）
size = driver.get_window_size()
width, height = size["width"], size["height"]
driver.swipe(width * 0.8, height * 0.5, width * 0.2, height * 0.5)   # 左滑

# 长按 / 多点触控（TouchAction）
action = TouchAction(driver)
action.long_press(element).perform()
```

**通用封装（滚动找元素）**：

```python
def scroll_to_text(driver, text):
    """向下滚动直到找到指定文本元素"""
    for _ in range(10):
        try:
            return driver.find_element(
                AppiumBy.ANDROID_UIAUTOMATOR,
                f'new UiSelector().text("{text}")')
        except Exception:
            driver.swipe(500, 1500, 500, 500, 300)
    raise AssertionError(f"未找到文本: {text}")
```

---

## 多平台注意事项

| 平台 | 注意点 |
|------|--------|
| Android | resource-id 定位；`appPackage`/`appActivity`；权限弹窗处理 |
| iOS | accessibility-id 为主；XCUITest 驱动；`bundleId` |
| 通用 | 避免依赖屏幕坐标；用相对计算；文本可能因本地化变化 |

**跨平台代码策略**：用平台判断封装定位器：

```python
PLATFORM = driver.capabilities["platformName"]

def login_btn(driver):
    if PLATFORM == "Android":
        return driver.find_element(AppiumBy.ID, "com.example.app:id/login_btn")
    return driver.find_element(AppiumBy.ACCESSIBILITY_ID, "login")
```

---

## WebView 与混合应用

混合应用（Hybrid App）需要在 WebView 上下文中操作 H5 页面：

```python
# 查看可用上下文
print(driver.contexts)          # ['NATIVE_APP', 'WEBVIEW_com.example']

# 切换到 WebView
driver.switch_to.context("WEBVIEW_com.example")

# 此时可用 Selenium 定位方式
driver.find_element(AppiumBy.CSS_SELECTOR, "#login-input").send_keys("tester")

# 切回原生
driver.switch_to.context("NATIVE_APP")
```

**注意**：
- 需要在 Appium 中启用 WebView 调试（`setWebContentsDebuggingEnabled(true)`）
- Android WebView 定位用 Chrome DevTools 协议

---

## 定位常见问题

| 问题 | 排查 |
|------|------|
| 元素找不到 | 等待不足？定位器不对？先截图 + page_source |
| 元素找到但不可点击 | 被遮挡 / 还在 loading，等 `element_to_be_clickable` |
| XPATH 失效 | UI 结构变了，改用 ID / 文本 / content-desc |
| 两个元素同 ID | 用复数定位 + 索引，或找父级缩小范围 |
| 文本在 WebView 里 | 需切换 context（见上） |
| 滚动列表定位不到 | 用滑动封装滚动查找 |

---

## 相关笔记

- Appium 环境与第一个用例 → [[5-Test-Appium基础]]
- 用 pytest 编排 + Page Object 封装 → [[7-Test-测试框架工程化]]
- flaky 处理与稳定性 → [[8-Test-最佳实践与FAQ]]
