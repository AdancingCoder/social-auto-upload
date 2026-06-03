# Selector 获取方法与新平台接入指南

> 以 `div[class^='upload-content'] input[class='upload-input']` 为切入点，
> 系统讲解项目里所有 selector 类型的来源，以及接入新平台时如何完整获取所需的 selector。

---

## 一、`div[class^='upload-content']` 是怎么来的

这类写法来自 **浏览器 DevTools 的 Elements 面板手工检查**，是最基础也最常用的获取方式。

具体步骤：

1. **打开目标页面**  
   比如 `https://creator.xiaohongshu.com/publish/publish?from=homepage&target=video`

2. **打开 DevTools**  
   `F12` 或右键 → 检查

3. **用元素选取工具定位**  
   点击 DevTools 左上角的箭头图标（或按 `Ctrl+Shift+C`），然后在页面上点击「上传视频」按钮区域

4. **在 Elements 面板里查看 HTML 结构**  
   你会看到类似：
   ```html
   <div class="upload-content--abc123">
     <input class="upload-input" type="file" accept="video/*">
   </div>
   ```

5. **分析 class 的稳定性**  
   - `upload-content--abc123` 末尾的 `abc123` 是构建工具（通常是 webpack 或 vite）生成的哈希后缀，每次构建都可能变
   - 但前缀 `upload-content` 是语义性的，相对稳定
   - 所以用 `[class^='upload-content']`（`^=` 表示"以此开头"）而不是完整的 `[class='upload-content--abc123']`

这就是 `div[class^='upload-content']` 的来源：**手工看到了 class 有哈希后缀，主动选择用前缀匹配**。

---

## 二、CSS 属性选择器语法速查

项目里用到的所有 `[attr...]` 写法：

| 写法 | 含义 | 项目示例 |
|------|------|---------|
| `[class^='foo']` | class **以 foo 开头** | `div[class^='upload-content']` |
| `[class*='foo']` | class **包含 foo** | `input[type="file"][accept*="image"]` |
| `[class='foo']` | class **完全等于 foo** | `input[class='upload-input']` |
| `[placeholder*='foo']` | placeholder 包含 foo | `input[placeholder*="填写标题"]` |
| `[data-placeholder*='foo']` | 自定义属性包含 foo | `p[data-placeholder*="输入正文描述"]` |
| `[id*='foo']` | id 包含 foo | `div[id*="creator-content-modal"]` |
| `[accept*='foo']` | accept 属性包含 foo | `input[accept*="image"]` |
| `[role='foo']` | ARIA role 等于 foo | `div[role="listbox"]` |

**选择哪种取决于哪个属性最稳定**：`placeholder`、`data-*`、`id`、`role` 这类属性通常比 `class` 更稳定，优先用这些。

---

## 三、项目里的 Selector 类型全览

项目里混合使用了四种定位策略，每种的来源不同：

### 类型 1：CSS Selector（最常见）

```python
# 来自 DevTools 手工检查
page.locator("div[class^='upload-content'] input[class='upload-input']")
page.locator('input[placeholder*="填写标题"]')
page.locator('.custom-switch-card')
page.locator('button:has-text("发布")')
```

**来源**：DevTools → Elements 面板查看结构，手工组合写出来。

---

### 类型 2：XPath

```python
# 来自需要"向上找祖先"或"找兄弟节点"时
page.locator('.login-box-container')
    .get_by_text("APP扫一扫登录")
    .locator("xpath=..//following-sibling::div//img")

page.locator("div.cover-plugin-title")
    .filter(has_text="设置封面")
    .locator("xpath=ancestor::div[contains(@class, 'cover-plugin-preview')]")
```

**来源**：CSS 无法表达"找父节点/兄弟节点"的关系时，切换到 XPath。  
在 DevTools 的 Elements 面板右键元素 → Copy → Copy XPath 可以得到完整 XPath，再根据需要简化。

---

### 类型 3：Playwright 语义定位器（推荐）

```python
# 不依赖 DOM 结构，直接按用户可见文本/角色定位
page.get_by_text("扫码登录", exact=True)
page.get_by_text("作品描述", exact=True)
page.get_by_role("img", name="二维码")
page.get_by_text("发布图文", exact=True)
```

**来源**：这些是 Playwright 内置的高级定位器，从页面可见内容出发，不依赖 class/id。  
优点是最稳定，不受前端重构影响，**优先考虑用这种**。

---

### 类型 4：链式过滤（组合定位）

```python
# 先定位一个区域，再在区域内过滤
page.locator('.custom-switch-card').filter(has_text="定时发布").locator('.d-switch')
page.locator("div.cover-plugin-title").filter(has_text="设置封面")
page.get_by_text("添加标签").locator("..").locator("..").locator(".semi-select")
```

**来源**：单个 selector 无法唯一定位目标元素时，先缩小范围再精确定位。  
`locator("..")` 是 XPath `..` 的封装，表示"父节点"。

---

## 四、新平台接入：selector 获取完整流程

### Step 0：准备工作

```bash
# 先安装依赖，确保 patchright 和 Chromium 可用
uv pip install -e .
patchright install chromium
```

---

### Step 1：用 Playwright codegen 自动录制（最快）

Playwright 自带代码生成工具，边操作边自动生成 selector：

```bash
# 启动录制，指定目标 URL
patchright codegen https://新平台的创作者后台URL
```

会弹出两个窗口：
- 左侧：真实浏览器（可以正常操作）
- 右侧：Playwright Inspector，实时显示生成的代码

你在浏览器里点击什么，Inspector 里就生成对应的 `page.click()`、`page.fill()` 等代码，**selector 是自动生成的**。

> `patchright` 的 codegen 和 playwright 的 codegen 用法完全一致。

---

### Step 2：用 Playwright Inspector 逐步调试（精确）

在代码里加一行 `await page.pause()`，运行后浏览器会暂停，弹出 Inspector 工具栏：

```python
async def upload(self, playwright):
    browser = await playwright.chromium.launch(headless=False)  # 必须有头
    context = await browser.new_context(storage_state=self.account_file)
    page = await context.new_page()
    await page.goto("https://新平台发布页URL")

    await page.pause()  # ← 加这一行，运行后会暂停在这里

    # 后面的上传逻辑...
```

Inspector 工具栏里有一个「Pick locator」按钮，点击后在页面上悬停任何元素，Inspector 会实时显示推荐的 selector，并高亮匹配的元素数量。

---

### Step 3：用 DevTools 手工检查（最可靠）

上面两种自动工具适合快速起步，但生成的 selector 有时过于具体（包含大量 class 哈希），需要手工简化。

**实际操作流程：**

**3-1. 打开目标页面，用有头模式运行到关键节点**

```python
await page.pause()  # 在要分析的步骤前暂停
```

**3-2. 在 DevTools Elements 面板里找到目标元素**

- 按 `Ctrl+Shift+C` 进入元素选取模式
- 点击页面上的目标元素（如「上传视频」的 input）
- Elements 面板会高亮对应的 HTML 节点

**3-3. 分析 HTML 结构，判断 class 稳定性**

```html
<!-- 看到这种结构 -->
<div class="upload-content--3xKf9">      ← 有哈希，用 ^= 前缀匹配
  <div class="upload-inner">
    <input class="upload-input"           ← 没有哈希，可以完整匹配
           type="file"
           accept="video/mp4,video/quicktime">
  </div>
</div>
```

**判断规则：**

| class 形态 | 稳定性 | 建议写法 |
|-----------|--------|---------|
| `upload-content` | 稳定 | `[class='upload-content']` 或 `.upload-content` |
| `upload-content--3xKf9` | 不稳定（有哈希） | `[class^='upload-content']` |
| `upload-content upload-content--3xKf9` | 不稳定 | `[class*='upload-content']` |
| `d-switch` | 稳定（设计系统命名） | `.d-switch` |

**3-4. 验证 selector 唯一性**

在 DevTools Console 里执行：

```javascript
// CSS Selector
document.querySelectorAll("div[class^='upload-content'] input[class='upload-input']").length
// 期望返回 1，如果返回多个说明不够精确

// XPath
$x("//div[contains(@class,'upload-content')]//input[@class='upload-input']").length
```

或者在 Playwright Inspector 的输入框里输入 selector，它会实时显示匹配到几个元素。

**3-5. 测试 selector 在代码里的行为**

```python
locator = page.locator("div[class^='upload-content'] input[class='upload-input']")
count = await locator.count()
print(f"匹配到 {count} 个元素")  # 期望 1
```

---

### Step 4：专项情况处理

**情况 A：上传 input 被隐藏**

文件上传的 `<input type="file">` 通常被 CSS 隐藏，DevTools 里看不到它被选中的效果。  
但 Playwright 的 `set_input_files()` 不需要元素可见，直接操作 DOM 级别。

找它的方法：在 Elements 面板按 `Ctrl+F` 搜索 `type="file"` 或 `accept=`。

**情况 B：动态加载的元素**

有些元素上传后才出现（如上传进度条、成功状态），需要等待：

```python
# 等待元素出现
await page.wait_for_selector('div.upload-success', timeout=60000)

# 等待元素消失
await page.wait_for_selector('div.upload-loading', state="hidden", timeout=60000)
```

在 DevTools 里触发操作后，观察 Elements 面板里 DOM 的变化（有高亮闪烁）。

**情况 C：iframe 里的内容**

如果发布页用了 `<iframe>`，需要先切入 frame：

```python
frame = page.frame_locator('iframe[src*="editor"]')
await frame.locator('input[placeholder="标题"]').fill("测试")
```

在 DevTools 里，iframe 在 Elements 面板里会显示为独立的文档树（有 `#document` 标识）。

**情况 D：Shadow DOM**

现代前端框架有时用 Shadow DOM（在 Elements 面板里显示为 `#shadow-root`）。  
Playwright 的 `locator()` 默认能穿透 Shadow DOM，但 `document.querySelector()` 不能。

---

## 五、新平台接入 Checklist

新增一个平台（以"某平台"为例），按以下顺序确认 selector：

```
□ 登录页
    - 登录框容器  →  用于判断是否已登录
    - 二维码图片元素（img[src]）
    - 扫码成功后的跳转 URL 规律

□ 发布页（视频）
    - 发布页 URL
    - 文件上传 input（type="file"，可能被隐藏）
    - 上传进度/完成状态的判断元素
    - 标题输入框
    - 描述/正文输入框
    - 话题/标签输入框 + 联想下拉列表的选项
    - 封面设置入口（如有）
    - 定时发布开关（如有）
    - 定时时间输入框（如有）
    - 发布按钮（立即/定时分两个状态）
    - 发布成功后的跳转 URL 规律

□ 发布页（图文，如有）
    - 图文发布 URL
    - 图片上传 input
    - 图片上传完成判断
    - 其余同上
```

每个 selector 确认时：
1. DevTools 里目测 class 是否有哈希后缀
2. Console 里 `querySelectorAll(selector).length === 1` 验证唯一性
3. 考虑是否有更语义化的替代（`placeholder`、`role`、`has-text`）

---

## 六、调试 selector 最快的完整流程（一句话版）

```
有头模式运行 → page.pause() 暂停 → DevTools 点目标元素 → 看 class 有无哈希
→ Console 里验证 querySelectorAll 唯一 → 写入代码 → 去掉 page.pause()
```
