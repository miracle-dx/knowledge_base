# Playwright 学习记录

## 一、Playwright 是什么

Playwright 是微软出品的浏览器自动化框架，支持 Chrome/Firefox/Safari，可以编程控制浏览器做各种操作。

### 安装（国内网络）
- 官方 CDN 被墙，需设镜像 `PLAYWRIGHT_DOWNLOAD_HOST=https://npmmirror.com/mirrors/playwright`
- npmmirror 上只有部分旧版 chromium，需降级 Playwright 版本匹配
- 当前可用：Playwright 1.48.0 → Chromium v1140

---

## 二、核心功能

### 1. 截图
```python
page.screenshot(path="截图.png")           # 可视区域截图
page.screenshot(path="截图.png", full_page=True)  # 整页截图（含滚动内容）
```
- 可截任何网页（百度、苹果官网等）
- 苹果官网有懒加载，需要先滚动触发再截图

### 2. 提取页面内容
```python
page.text_content("h1")                          # 取单个元素文字
page.eval_on_selector_all("li", "els => ...")    # 取多个元素
page.evaluate("document.body.innerText")         # 取全部文字
```

### 3. 模拟用户操作
```python
page.fill("#username", "小明")    # 填写输入框
page.click("button")               # 点击按钮
page.press("input", "Enter")      # 按键盘
```

### 4. 网络请求拦截（Mock 接口）
```python
def mock_api(route):
    if "/api/users" in route.request.url:
        route.fulfill(status=200, content_type="application/json",
                      body='{"users":[...]}')  # 返回假数据
    else:
        route.continue_()

page.route("**/*", mock_api)
```
- 前端请求 API → Playwright 中途拦截 → 返回你造的数据
- 适用于前端开发时后端还没写好
- **注意：只在 Playwright 的浏览器里生效，你自己的 Chrome 看不到**

### 5. 生成 PDF
```python
page.pdf(path="报告.pdf", format="A4", print_background=True)
```
- 把网页转成 PDF，适合自动生成报表

### 6. Mock 前端开发方案：MSW
- Playwright 的 Mock 只在自己浏览器里生效
- 开发调试场景用 **MSW (Mock Service Worker)**，在你的 Chrome 里也能看到 Mock 数据
- 已安装在 `industrial-distribution-dashboard` 项目中
- 新增 Mock 接口在 `src/mocks/handlers.js` 中添加

### 7. Cookie 管理（登录状态保持）
```python
# 保存登录状态
context.storage_state(path="state.json")

# 下次直接加载
context = browser.new_context(storage_state="state.json")
```
- 相当于浏览器保存密码/登录态，下次不用重新登录

### 8. 多标签页管理
```python
pages = browser.contexts[0].pages
pages[0]  # 第一个标签页
pages[1]  # 新打开的标签页
```
- 自动打开新标签页 → 爬数据 → 关掉 → 回列表

### 9. 文件上传/下载
```python
# 上传：把文件绑定到 input 元素上
page.set_input_files("input[type=file]", "图片.png")
page.click("#submit")  # 点提交才真正发送

# 下载：监听下载事件
with page.expect_download() as dl:
    page.click("#download")
dl.value.save_as("文件.txt")
```

---

## 三、进阶技巧

### 10. Locator API（更精确的定位）

```python
# 四种基础定位
page.locator("#username")                         # 1. 按 ID
page.locator("[placeholder='请输入邮箱']")          # 2. 按 placeholder
page.get_by_label("密码")                          # 3. 按 label 文字（需 for 属性）
page.get_by_role("button", name="注册")            # 4. 按按钮/链接文字

# 组合定位（筛选）
page.locator(".item[data-category='phone'] .name")   # 属性筛选
page.locator(".item").filter(has_text="9,999")       # 按内容筛选
page.locator(".item").nth(1)                         # 取第2个
page.locator(".item").first                          # 取第一个
page.locator(".item").last                           # 取最后一个
page.locator(".item").count()                        # 统计数量
page.locator(".item .name").all_text_contents()      # 取所有匹配元素的文字
```

Locator 比直接写 CSS 选择器更稳定，**自带等待**，元素没出现会自动等。

### 11. 等待策略

```python
# 等元素出现（再操作它）
page.wait_for_selector("#content")

# 等页面完全加载
page.wait_for_load_state("networkidle")

# 等自定义条件
page.wait_for_function("typeof appReady !== 'undefined'")

# Locator 自带等待（推荐！）
page.locator("#content").click()  # 自动等到元素可操作为止
```

### 12. 浏览器上下文（隔离会话）

```python
# 创建两个隔离的浏览器会话（像两个不同的用户）
context1 = browser.new_context()  # 用户A：已登录
context2 = browser.new_context()  # 用户B：未登录

# 各自的 cookie、缓存互不影响
context1.add_cookies([{"name": "token", "value": "abc", "domain": "localhost", "path": "/"}])
```

---

## 四、实际应用

### 自动生成测试报告
- 跑 `vitest run --reporter=json` 获取测试结果
- Python 拼成 HTML
- Playwright 转 PDF
- 已封装脚本：`generate_report.py`，命令 `npm run report`

### 苹果官网爬价格
- 苹果官网没有反爬，可直接爬
- iPhone 17 Pro 价格：¥8,999 起

### 国内电商反爬严重
- 京东、淘宝有强反爬措施，Playwright 难以直接爬取
- 需要使用 Browserbase 等云浏览器或官方 API

---

## 四、总结

| 功能 | 适用场景 |
|------|---------|
| 截图 | 页面快照、监控变化 |
| 提取内容 | 爬虫、数据采集 |
| 模拟操作 | 自动化测试、自动填表 |
| Mock 接口 | 前端开发、测试（Playwright浏览器内） |
| MSW | 前端开发、调试（自己 Chrome 里） |
| PDF | 自动生成报表、报告 |
| Cookie | 保持登录状态，避免重复登录 |
| 多标签页 | 爬取详情页 |
| 上传下载 | 自动化文件操作 |
