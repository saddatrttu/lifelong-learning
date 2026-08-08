# 获取微信小程序 WebView（com.tencent.mm）操作手册

> 目标：让 Appium 能识别并切换到微信小程序内部的 WebView，从而获取小程序 DOM 元素进行自动化测试。

## 一、前置条件

| 项目 | 说明 |
|---|---|
| 手机 | Android 真机（尽量避免模拟器，易触发微信风控），已开启 USB 调试 |
| 电脑 | 已安装 Appium、uiautomator2、Android SDK（adb）、匹配的 chromedriver |
| 微信 | 建议最新版本，尽量使用测试小号 |

## 二、开启小程序 WebView 调试（关键前提）

小程序侧需开启调试，否则 `contexts` 里只有 `NATIVE_APP`：

1. 小程序 `app.json` 中添加 `"debug": true`；
2. 或用微信开发者工具打开项目，勾选「启用调试」并预览；
3. 在微信里打开 `http://debugxweb.qq.com/?inspector=true` 并勾选开启 XWeb 调试（部分版本需要这一步，否则 WebView 无法暴露调试端口）；
4. 手机进入目标小程序页面后，在电脑 Chrome 打开 `chrome://inspect/`，确认能看到 `com.tencent.mm` 的 webview 且能点进 inspect（能打开 DevTools 即说明调试生效）。

## 三、Appium capabilities 配置

```python
caps = {
    "platformName": "android",
    "appPackage": "com.tencent.mm",
    "appActivity": "com.tencent.mm.ui.LauncherUI",
    "noReset": True,
    "unicodeKeyboard": True,
    "resetKeyboard": True,
    "chromedriverExecutable": "/path/to/chromedriver",      # 版本需匹配微信内核
    "chromeOptions": {"androidProcess": "com.tencent.mm:appbrand1"},  # 关键：指定渲染进程
    "enableWebviewDetailsCollection": True,
}
```

> `androidProcess` 常见取值：`com.tencent.mm:appbrand0` / `appbrand1`（随打开的小程序数量变化）；旧版微信可能为 `com.tencent.mm:tools`、X5 内核为 `com.tencent.mm:toolsmp`。

## 四、启动会话并切换到 WebView

```python
from selenium.webdriver.support.ui import WebDriverWait

driver = webdriver.Remote("http://127.0.0.1:4723/wd/hub", caps)

# 1. 等待 webview 出现（contexts 有延迟，需轮询）
WebDriverWait(driver, 30).until(lambda d: len(d.contexts) > 1)

# 2. 打印所有 context，确认拿到 WEBVIEW_com.tencent.mm
print(driver.contexts)
# 期望输出类似：['NATIVE_APP', 'WEBVIEW_com.tencent.mm:appbrand1']

# 3. 切换到微信 webview
driver.switch_to.context("WEBVIEW_com.tencent.mm:appbrand1")

# 4. 验证：能拿到页面源码即成功
print(driver.page_source)
```

## 五、常见问题排查

| 现象 | 原因 | 解决 |
|---|---|---|
| contexts 只有 `NATIVE_APP` | WebView 调试未开启 | 回到第二步开启 `debug` 开关 |
| `Failed to get sockets matching: @xweb_devtools_remote_*` | chromedriver 与 XWeb 内核不匹配 | 换匹配版本 chromedriver + 指定 `androidProcess` |
| 切 context 报"version mismatch" | chromedriver 版本与微信内核不符 | 多试几个版本（60~80 区间） |
| 切换后定位不到内部元素 | 小程序渲染模式限制 | 用 `execute_script` 走 JS 定位，或改用官方 miniprogram-automator SDK |
| 微信封号风险 | 操作频繁 / 模拟器 / 多开 | 使用测试小号、真机、正常操作频率 |

## 六、要点总结

1. 先开调试：`"debug": true` 且 chrome://inspect 可见；
2. 指定进程：`chromeOptions.androidProcess = com.tencent.mm:appbrand1`；
3. 版本匹配：chromedriver 与微信内置内核版本对应；
4. 轮询等待：contexts 列表刷新有延迟；
5. 切换后先 `page_source` 验证再写定位。
