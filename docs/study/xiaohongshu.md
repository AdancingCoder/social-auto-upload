# 小红书 CLI 全链路详解

> 覆盖以下五条命令的完整执行路径：
> ```
> sau xiaohongshu login --account my_xiaohongshu
> sau xiaohongshu login --account my_xiaohongshu --headless
> sau xiaohongshu check --account my_xiaohongshu
> sau xiaohongshu upload-video --account my_xiaohongshu --file videos/demo.mp4 --title "示例标题" --desc "示例简介" --tags 运动,训练
> sau xiaohongshu upload-note --account my_xiaohongshu --images videos/demo1.png videos/demo2.png --title "图文标题" --note "图文示例" --tags 图文,测试
> ```

---

## 一、架构总览

```
用户输入 sau 命令
    │
    ▼
pyproject.toml  [project.scripts] sau = "sau_cli:main"
    │
    ▼
sau_cli.py  ── build_parser() 解析参数
    │            dispatch(args)  分发到具体平台 + 动作
    │
    ▼
uploader/xiaohongshu_uploader/main.py
    ├── cookie_auth()            ── 校验 cookie
    ├── xiaohongshu_setup()      ── 登录前门卫
    ├── xiaohongshu_cookie_gen() ── 生成新 cookie（二维码登录）
    ├── XiaoHongShuVideo         ── 视频上传类
    └── XiaoHongShuNote          ── 图文上传类
    
utils/
    ├── login_qrcode.py          ── 二维码保存 / 解码 / 终端打印
    └── base_social_media.py     ── set_init_script()（注入反爬脚本）

uploader/base_video.py           ── 文件校验 / 发布时间校验基类
conf.py                          ── BASE_DIR / LOCAL_CHROME_PATH / LOCAL_CHROME_HEADLESS
```

---

## 二、`sau` 命令是怎么注册进来的

`pyproject.toml` 声明了一条 console_scripts 入口：

```toml
[project.scripts]
sau = "sau_cli:main"
```

执行 `uv pip install -e .` 后，`sau` 可执行文件被写入虚拟环境的 `Scripts/` 目录。每次调用 `sau ...`，Python 解释器直接执行 `sau_cli.py` 中的 `main()` 函数。

---

## 三、CLI 解析层（`sau_cli.py`）

### `build_parser()`

使用 `argparse` 构建两级子命令树：

```
sau
└── xiaohongshu
    ├── login      --account  [--headless | --headed]  [--debug]
    ├── check      --account
    ├── upload-video  --account --file --title  [--desc] [--tags] [--schedule] [--thumbnail] [--headless] [--debug]
    └── upload-note   --account --images ...  --title  [--note] [--tags] [--schedule] [--headless] [--debug]
```

关键细节：
- `--headless` / `--headed` 互斥，默认 `headless=True`
- `--tags` 是逗号分隔字符串，`parse_tags()` 分割后去掉 `#` 前缀
- `--schedule` 经 `schedule_value()` 转为 `datetime`，格式 `YYYY-MM-DD HH:MM`
- `--file` / `--images` / `--thumbnail` 使用 `existing_file_path()` 类型检查，文件不存在立即报错退出
- `--images` 使用 `nargs="+"` 接受多个路径

### `dispatch(args)`

分发入口，`args.platform == "xiaohongshu"` 时：

| action | 调用 | 返回 |
|--------|------|------|
| `login` | `login_xiaohongshu_account()` | `dict`，含 success / message / account_file |
| `check` | `check_xiaohongshu_account()` | `bool` |
| `upload-video` | `upload_xiaohongshu_video()` | `Path`（account_file） |
| `upload-note` | `upload_xiaohongshu_note()` | `Path`（account_file） |

所有异步函数统一由 `asyncio.run(dispatch(args))` 驱动。

### `resolve_account_file()`

```python
account_file = BASE_DIR / "cookies" / f"xiaohongshu_{account_name}.json"
```

- `account_name` 是用户自定义字符串，不要求固定值
- 文件夹不存在时自动创建
- 对应 `my_xiaohongshu` → `cookies/xiaohongshu_my_xiaohongshu.json`

---

## 四、命令一：`sau xiaohongshu login --account my_xiaohongshu`

### 调用链

```
dispatch()
  └── login_xiaohongshu_account(account_name, headless=True)  # 默认无头
        └── xiaohongshu_setup(account_file, handle=True, return_detail=True, headless=True)
              ├── cookie_auth(account_file)  ← 先校验现有 cookie
              │     └── [cookie 有效] 直接返回成功 dict
              └── [cookie 无效 / 不存在] → xiaohongshu_cookie_gen(account_file, headless=True)
```

### `cookie_auth()` 工作原理

1. 文件不存在 → 直接返回 `False`
2. 启动无头 Chromium，加载 `account_file` 里的 storage_state
3. 注入 `stealth.min.js`（防检测）
4. 跳转 `https://creator.xiaohongshu.com/publish/publish?from=homepage&target=video`
5. 等 3 秒，判断：
   - 当前 URL 是否跳回到 `/login` → cookie 失效
   - 是否出现 `.login-box` 弹窗 → cookie 失效
   - 都没有 → cookie 有效

### `xiaohongshu_cookie_gen()` 工作原理（二维码登录）

1. **启动浏览器**  
   ```python
   browser = await playwright.chromium.launch(headless=headless, channel="chrome")
   ```
   使用 patchright（playwright 的反检测 fork），`channel="chrome"` 指向系统 Chrome。

2. **打开登录页**  
   `https://creator.xiaohongshu.com/login`

3. **切换到扫码面板**  
   - 等待 `.login-box` 出现
   - 如果页面当前显示的是手机号登录，找到切换按钮 `img.css-wemwzq` 并点击
   - 等待「扫一扫」文字出现

4. **提取二维码图片**  
   通过 XPath 找到 `.login-box-container` 内「APP扫一扫登录」区域的 `<img>` 标签，提取 `src` 属性。

5. **保存二维码**  
   - 若 `src` 是 `data:image/...` 格式 → base64 解码写入本地 PNG 文件
   - 否则 → Playwright `screenshot` 截图保存
   - 文件名格式：`cookies/xiaohongshu_my_xiaohongshu_xhs_login_qrcode_20260603_120000.png`

6. **终端打印二维码**  
   - `cv2.QRCodeDetector` 解码 PNG 里的二维码内容
   - `segno.make()` 生成终端可打印的二维码字符画
   - 打印到 stdout，同时提示本地图片路径

7. **轮询等待扫码**  
   每 3 秒检查一次，最多 100 次（约 5 分钟），判断条件：
   - 当前 URL 不再是 `/login`
   - 且 `.login-box` 消失或不可见

8. **保存 cookie**  
   ```python
   await context.storage_state(path=account_file)
   ```
   将 Chromium context 的完整 storage（cookies + localStorage）写入 JSON 文件。

9. **二次校验**  
   调用 `cookie_auth()` 验证刚写入的 cookie 是否真的有效。

10. **清理**  
    删除临时二维码 PNG 文件，关闭 browser。

---

## 五、命令二：`sau xiaohongshu login --account my_xiaohongshu --headless`

与命令一完全相同，仅 `--headless` 显式传入（其实默认就是 `headless=True`）。  
加这个参数是为了代码可读性和脚本可移植性，行为无差异。

> 如需有头模式（浏览器窗口可见），改用 `--headed`，此时登录窗口会弹出，可直接在界面上操作扫码。

---

## 六、命令三：`sau xiaohongshu check --account my_xiaohongshu`

### 调用链

```
dispatch()
  └── check_xiaohongshu_account(account_name)
        ├── account_file 不存在 → return False
        └── xiaohongshu_cookie_auth(account_file)  ← 同 cookie_auth()
              → True / False
```

### 输出

| 结果 | stdout | 退出码 |
|------|--------|--------|
| cookie 有效 | `valid` | 0 |
| cookie 无效 | `invalid` | 1 |

这条命令不启动登录流程，只做只读校验。适合在上传前快速判断是否需要重新登录。

---

## 七、命令四：`sau xiaohongshu upload-video`

```
sau xiaohongshu upload-video \
  --account my_xiaohongshu \
  --file videos/demo.mp4 \
  --title "示例标题" \
  --desc "示例简介" \
  --tags 运动,训练
```

### 调用链

```
dispatch()
  └── upload_xiaohongshu_video(XiaohongshuVideoUploadRequest)
        ├── xiaohongshu_setup(account_file, handle=False)  ← 不触发登录，只校验
        │     └── [False] → 抛出 RuntimeError，提示先执行 login
        └── XiaoHongShuVideo(...).main()
              └── xiaohongshu_upload_video()
                    └── upload(playwright)
```

### 参数映射

| CLI 参数 | Request 字段 | 传入 `XiaoHongShuVideo` |
|---------|-------------|------------------------|
| `--account` | `account_name` | 用于 resolve account_file |
| `--file` | `video_file` | `file_path` |
| `--title` | `title` | `title`（截取前 20 字） |
| `--desc` | `description` | `desc` |
| `--tags 运动,训练` | `tags = ["运动", "训练"]` | `tags` |
| `--schedule` | `publish_date` | 影响 `publish_strategy` |
| `--thumbnail` | `thumbnail_file` | `thumbnail_path` |

未传 `--schedule` → `publish_strategy = "immediate"`（立即发布）

### `XiaoHongShuVideo.upload()` 执行步骤

**1. 前置校验 `validate_upload_args()`**

- 检查 cookie 文件存在且有效（同 `cookie_auth()`）
- 检查 `title` 非空
- 检查视频文件存在，扩展名在白名单（`.mp4 .mov .avi .mkv .m4v .webm .flv .wmv`）
- 如有 thumbnail，检查图片文件存在，扩展名在白名单（`.jpg .jpeg .png .webp .bmp`）
- 定时发布时检查时间 > 当前时间 + 2 小时

**2. 启动浏览器**

```python
browser = await playwright.chromium.launch(headless=self.headless, channel="chrome")
context = await browser.new_context(
    permissions=["geolocation"],
    storage_state=self.account_file,   # 加载已保存的 cookie
)
context = await set_init_script(context)  # 注入 stealth.min.js
```

**3. 跳转视频发布页**

`https://creator.xiaohongshu.com/publish/publish?from=homepage&target=video`

**4. 上传视频文件**

```python
await page.locator("div[class^='upload-content'] input[class='upload-input']")
    .set_input_files(self.file_path)
```

**5. 等待上传完成**

轮询判断预览区域是否出现以下关键词之一：

```
上传成功 | 分辨率 | 重新上传 | 编辑封面 | 已上传 | 已选择 | 100%
```

或者检测标题输入框是否已可见（进入编辑状态则认为上传完成）。

**6. 填写元数据 `fill_meta()`**

| 方法 | 操作 |
|------|------|
| `fill_title()` | 填入 `input[placeholder*="填写标题"]`，最多 20 字 |
| `fill_desc()` | 点击 `p[data-placeholder*="输入正文描述"]`，全选后键入描述文字 |
| `fill_tags()` | 每个 tag 前加 `#` 后键入，等待话题联想弹窗 `#creator-editor-topic-container` 出现，点选第一项 |

**7. 设置封面（可选）**

- 找到「设置封面」区域的默认封面占位元素
- 点击弹出封面上传 modal
- `set_input_files()` 上传封面图片
- 点击「确定」按钮确认

**8. 定时发布（可选）**

如果传了 `--schedule`：

```python
await page.locator('.custom-switch-card').filter(has_text="定时发布").locator('.d-switch').click()
# 填入时间字符串，格式 YYYY-MM-DD HH:MM
await page.locator('.d-datepicker-input-filter input.d-text').fill(publish_date_hour)
```

**9. 点击发布按钮**

- 立即发布：点击 `button:has-text("发布")`
- 定时发布：点击 `button:has-text("定时发布")`
- 等待页面跳转到 `/publish/success?...`

**10. 更新 cookie**

```python
await context.storage_state(path=self.account_file)
```

每次上传后自动刷新 cookie 文件，保持登录态的新鲜度。

---

## 八、命令五：`sau xiaohongshu upload-note`

```
sau xiaohongshu upload-note \
  --account my_xiaohongshu \
  --images videos/demo1.png videos/demo2.png \
  --title "图文标题" \
  --note "图文示例" \
  --tags 图文,测试
```

### 调用链

```
dispatch()
  └── upload_xiaohongshu_note(XiaohongshuNoteUploadRequest)
        ├── xiaohongshu_setup(account_file, handle=False)
        └── XiaoHongShuNote(...).main()
              └── xiaohongshu_upload_note()
                    └── upload(playwright)
```

### 与 upload-video 的差异

| 维度 | upload-video | upload-note |
|------|-------------|-------------|
| 发布页 URL | `?target=video` | `?target=image` |
| 上传接口 | `input.upload-input`（视频） | `input[type="file"][accept*="image"]`（图片） |
| 多文件 | 单文件 | 支持多张图片（`nargs="+"`) |
| 等待标志 | 预览区域关键词 | 标题输入框 `visible` |
| 描述字段 | `--desc` → `desc` | `--note` → `note`（同时赋值给 `desc`） |
| 封面设置 | 支持 `--thumbnail` | 不支持 |

### `XiaoHongShuNote.__init__()` 字段逻辑

```python
self.note = note or ""
self.desc = desc if desc is not None else self.note  # desc 兜底用 note
self.title = title or (self.desc or self.note)[:20]  # title 兜底取 desc/note 前 20 字
```

CLI 传入 `--note "图文示例"` → `note = "图文示例"`，`desc = "图文示例"`，`title = "图文标题"`（显式传入）。

### `upload_note_content()` 执行步骤

1. 跳转 `https://creator.xiaohongshu.com/publish/publish?from=homepage&target=image`
2. 找到图片上传 input，`set_input_files(self.image_paths)`（一次传所有图片）
3. 轮询等待标题框可见（图片上传完成的信号）
4. 填写元数据（`fill_title` + `fill_desc` + `fill_tags`，流程同视频）
5. 定时发布（可选）
6. 点击发布按钮，等待跳转 `/publish/success?...`
7. 更新 cookie

---

## 九、底层依赖详解

### patchright vs playwright

项目使用 `patchright` 而非原生 `playwright`。patchright 是对 playwright 的 fork，主要差异在于修复了一些浏览器自动化特征（navigator.webdriver、CDP 暴露等），降低被平台反爬检测到的概率。API 完全兼容 playwright。

### stealth.min.js

每个 browser context 创建后都调用：

```python
await context.add_init_script(path=str(BASE_DIR / "utils/stealth.min.js"))
```

这个脚本在页面加载时执行，进一步隐藏 headless 浏览器特征（如修改 `navigator.plugins`、`window.chrome` 等属性）。

### cookie 存储格式

`storage_state` 是 Playwright 的标准格式，JSON 文件包含：

```json
{
  "cookies": [...],
  "origins": [
    {
      "origin": "https://creator.xiaohongshu.com",
      "localStorage": [...]
    }
  ]
}
```

每次上传结束后都会回写这个文件，维持最新的登录态。

### 二维码工具链

```
cv2.QRCodeDetector  ── 从 PNG 图片解码二维码内容（URL 字符串）
segno.make()        ── 将 URL 重新编码为可在终端打印的二维码字符画
```

两者组合实现"从页面截图中提取内容 → 在终端渲染"的二维码流转。

---

## 十、cookie 文件生命周期

```
[首次 login]
  xiaohongshu_cookie_gen()
    → 扫码成功
    → context.storage_state() 写入 cookies/xiaohongshu_my_xiaohongshu.json

[每次 upload-video / upload-note]
  validate_upload_args()
    → cookie_auth() 校验有效性
  上传完成
    → context.storage_state() 刷新 cookie 文件

[check]
  cookie_auth() 只读校验，不写文件

[cookie 失效]
  重新执行 login 命令覆盖旧文件
```

---

## 十一、`--schedule` 定时发布完整流程

1. CLI 解析：`parse_schedule("2026-06-10 14:00")` → `datetime(2026, 6, 10, 14, 0)`
2. `dispatch()` 判断：`publish_strategy = XIAOHONGSHU_PUBLISH_STRATEGY_SCHEDULED`
3. `validate_publish_date()` 校验：
   - 必须是 `datetime` 类型
   - 必须晚于当前时间
   - 必须大于当前时间 + 2 小时
4. Playwright 操作：
   - 点击「定时发布」switch
   - 向时间输入框填入 `"2026-06-10 14:00"`
5. 点击 `button:has-text("定时发布")` 提交（而非「发布」按钮）

---

## 十二、错误处理与退出码

| 情况 | 行为 |
|------|------|
| `--file` 指定的文件不存在 | `argparse.ArgumentTypeError`，命令直接退出，不启动浏览器 |
| cookie 不存在或失效（upload 时） | `RuntimeError`，提示先执行 `login` |
| 定时时间格式错误 | `argparse.ArgumentTypeError` |
| 定时时间不足 2 小时 | `ValueError` in `validate_publish_date()` |
| 二维码等待超时 | `status: "timeout"`，login 返回非 success |
| `check` cookie 有效 | 退出码 0，stdout 输出 `valid` |
| `check` cookie 无效 | 退出码 1，stdout 输出 `invalid` |

---

## 十三、常用调试技巧

### 启用有头模式查看浏览器行为

```bash
sau xiaohongshu upload-video --account my_xiaohongshu --file demo.mp4 --title "测试" --headed
```

### 启用 debug 模式截图

传入 `--debug` 后，上传/发布过程中遇到重试时会自动截图（`page.screenshot(full_page=True)`），便于分析页面状态。

### 手动激活虚拟环境后调用

```powershell
.\.venv\Scripts\Activate.ps1
sau xiaohongshu check --account my_xiaohongshu
```

### 跳过 sau，直接用 uv run

```bash
uv run sau xiaohongshu check --account my_xiaohongshu
```

### 直接调用 Python 模块测试

```python
import asyncio
from uploader.xiaohongshu_uploader.main import cookie_auth
print(asyncio.run(cookie_auth("cookies/xiaohongshu_my_xiaohongshu.json")))
```

---

## 十四、目录结构速查

```
social-auto-upload/
├── pyproject.toml                        # sau = "sau_cli:main" 注册入口
├── sau_cli.py                            # CLI 解析 + dispatch
├── conf.py                               # BASE_DIR / headless 配置
├── cookies/
│   └── xiaohongshu_my_xiaohongshu.json  # 账号 cookie 文件（运行时生成）
├── uploader/
│   └── xiaohongshu_uploader/
│       ├── __init__.py                   # 创建 cookies/xiaohongshu_uploader/ 目录
│       └── main.py                       # 所有核心逻辑
├── uploader/
│   └── base_video.py                     # 文件校验 / 时间校验基类
├── utils/
│   ├── login_qrcode.py                   # 二维码工具
│   ├── base_social_media.py              # set_init_script()
│   └── stealth.min.js                    # 反检测脚本
└── skills/xiaohongshu-upload/
    ├── SKILL.md                          # skill 说明（agent 优先读）
    └── references/
        ├── cli-contract.md               # 命令契约
        ├── runtime-requirements.md       # 运行前提
        └── troubleshooting.md            # 故障排查
```
