# 抖音 CLI 全链路详解

本文逐命令拆解以下五条 CLI 调用的完整执行路径：

```bash
sau douyin login --account my_douyin
sau douyin login --account my_douyin --headed
sau douyin check --account my_douyin
sau douyin upload-video --account my_douyin --file videos/demo.mp4 --title "示例标题" --desc "示例简介" --tags 运动,训练
sau douyin upload-note --account my_douyin --images videos/demo1.png videos/demo2.png --title "图文标题" --note "图文示例" --tags 图文,测试
```

---

## 一、整体架构

```
pyproject.toml
  [project.scripts]
  sau = "sau_cli:main"        ← CLI 入口注册
         │
         ▼
sau_cli.py                    ← 唯一 CLI 主文件
  build_parser()              ← argparse 构造所有子命令
  dispatch(args)              ← 路由到各平台/操作
  login_douyin_account()
  check_douyin_account()
  upload_video()
  upload_note()
         │
         ▼
uploader/douyin_uploader/main.py   ← 抖音核心实现
  douyin_setup()             ← cookie 检验 + 登录入口
  cookie_auth()              ← cookie 有效性检测
  douyin_cookie_gen()        ← 扫码登录流程
  DouYinVideo.upload()       ← 视频上传自动化
  DouYinNote.upload()        ← 图文上传自动化
         │
         ▼
utils/
  base_social_media.py       ← set_init_script()（注入 stealth.min.js）
  login_qrcode.py            ← 二维码保存 / 解码 / 终端打印
  log.py                     ← douyin_logger
uploader/base_video.py       ← BaseVideoUploader（文件校验 / 时间校验）
conf.py                      ← BASE_DIR / LOCAL_CHROME_PATH / headless 默认值
```

账号文件路径规则（`resolve_account_file`）：

```
cookies/douyin_{account_name}.json
# 示例：cookies/douyin_my_douyin.json
```

---

## 二、`sau douyin login --account my_douyin`

### 2.1 参数解析

`build_parser()` 注册子命令 `douyin → login`，`add_runtime_flags()` 附加：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--account` | 必填 | 账号名，决定 cookie 文件名 |
| `--headless` / `--headed` | `headless=True` | 浏览器是否有头，互斥组 |
| `--debug` | False | 调试模式 |

无 `--headed`，`args.headless = True`。

### 2.2 dispatch → login_douyin_account

```python
# sau_cli.py:dispatch
result = await login_douyin_account(args.account, headless=args.headless)
# args.headless = True
```

```python
# sau_cli.py:login_douyin_account
account_file = resolve_account_file("douyin", "my_douyin")
# → BASE_DIR/cookies/douyin_my_douyin.json
return await douyin_setup(str(account_file), handle=True, return_detail=True, headless=True)
```

### 2.3 douyin_setup：cookie 检查 + 登录分支

```python
# uploader/douyin_uploader/main.py:douyin_setup
if not os.path.exists(account_file) or not await cookie_auth(account_file):
    # handle=True → 进入扫码登录
    result = await douyin_cookie_gen(account_file, headless=True)
    return result   # return_detail=True
```

`cookie_auth` 判定逻辑：

1. 启动无头 Chromium（`headless=True`，`channel="chrome"`）
2. 加载 `storage_state=account_file`（JSON cookie）
3. 调 `set_init_script` → 注入 `utils/stealth.min.js`（反检测）
4. 导航 `https://creator.douyin.com/creator-micro/content/upload`
5. 等待 URL 精确匹配（5 秒超时）
6. 检查页面是否出现"手机号登录"或"扫码登录"文字
7. 有则返回 `False`，无则返回 `True`

首次登录 → cookie 文件不存在 → 跳过 `cookie_auth` → 直接进入 `douyin_cookie_gen`。

### 2.4 douyin_cookie_gen：扫码登录全流程

```
启动 Playwright Chromium（headless=True）
  │
  ├─ new_context()
  ├─ set_init_script()        ← 注入 stealth.min.js
  ├─ new_page()
  └─ goto("https://creator.douyin.com/")
        │
        ▼
_save_douyin_qrcode(page, account_file)
  ├─ 等待"扫码登录"tab（30s 超时）
  ├─ 定位 img[aria-label="二维码"]
  ├─ 读取 src 属性（data:image/png;base64,...）
  ├─ 写文件 → cookies/douyin_my_douyin_login_qrcode_TIMESTAMP.png
  ├─ cv2.QRCodeDetector 解码二维码内容
  └─ segno.make() 在终端打印 ASCII/Unicode 二维码
        │
        ▼
_wait_for_douyin_login(page, account_file, qrcode_info)
  轮询（3s间隔，最多100次 ≈ 5分钟）：
  ├─ _is_douyin_login_completed()
  │     ├─ URL 是否以 creator-micro/home 开头
  │     └─ 是否无登录弹窗 → 返回 True
  ├─ 检测"二维码失效"→ 点击刷新，重新保存二维码
  └─ 超时 → 返回 failure result
        │
        ▼（登录成功）
context.storage_state(path=account_file)  ← 持久化 cookie
asyncio.sleep(2)
cookie_auth(account_file)                 ← 二次验证
        │
        ▼（finally）
remove_qrcode_file(qrcode_path)           ← 清理临时 PNG
context.close() / browser.close()
```

### 2.5 返回与输出

```python
# dispatch
if not result["success"]:
    raise RuntimeError(result["message"])
print(f"Douyin login flow completed: {result['account_file']}")
return 0
```

result 结构：

```python
{
    "success": True,
    "status": "success",
    "message": "抖音扫码登录成功",
    "account_file": "...cookies/douyin_my_douyin.json",
    "qrcode": {"image_path": "...", "image_data_url": "data:image/..."},
    "current_url": "https://creator.douyin.com/creator-micro/home",
}
```

---

## 三、`sau douyin login --account my_douyin --headed`

与上一命令完全相同，唯一区别：

```
--headed  →  args.headless = False
```

`douyin_cookie_gen` 中：

```python
browser = await playwright.chromium.launch(headless=False, channel="chrome")
```

浏览器窗口可见，用户可以直接在窗口内观察登录页面。适合调试或自动化扫码失败时人工干预。

---

## 四、`sau douyin check --account my_douyin`

### 4.1 参数解析

子命令 `douyin → check`，只有 `--account`，无 headless 参数（check 始终无头）。

### 4.2 完整调用链

```
dispatch(args)
  └─ check_douyin_account("my_douyin")
        ├─ resolve_account_file("douyin", "my_douyin")
        │     → BASE_DIR/cookies/douyin_my_douyin.json
        ├─ 文件不存在 → return False
        └─ douyin_cookie_auth(str(account_file))
              ↕ 即 cookie_auth()
              启动无头 Chromium
              加载 storage_state
              注入 stealth.min.js
              goto creator.douyin.com/content/upload
              等待 URL 匹配（5s）
              检查"手机号登录"/"扫码登录"
              返回 True / False
```

### 4.3 输出

```python
print("valid" if is_valid else "invalid")
return 0 if is_valid else 1   # 进程退出码
```

- `valid` + 退出码 0：cookie 有效，可直接上传
- `invalid` + 退出码 1：需要重新 login

---

## 五、`sau douyin upload-video`

### 5.1 参数解析

子命令 `douyin → upload-video`，`add_runtime_flags()` 附加 headless/debug：

| 参数 | 类型 | 说明 |
|------|------|------|
| `--account` | str | 账号名 |
| `--file` | `existing_file_path` | 视频文件（检查存在性）|
| `--title` | str | 标题（上传后截断 30 字）|
| `--desc` | str | 描述，默认空 |
| `--tags` | str | 逗号分隔，`parse_tags()` 去 `#` 前缀 |
| `--schedule` | `schedule_value` | `%Y-%m-%d %H:%M` 格式，`parse_schedule()` 转 datetime |
| `--thumbnail` | `existing_file_path` | 竖版封面 |
| `--product-link` | str | 商品链接 |
| `--product-title` | str | 商品标题 |
| `--headless` / `--headed` | bool | 默认 headless |

无 `--schedule` → `publish_strategy = DOUYIN_PUBLISH_STRATEGY_IMMEDIATE = "immediate"`

### 5.2 dispatch → upload_video

```python
request = DouyinVideoUploadRequest(
    account_name="my_douyin",
    video_file=Path("videos/demo.mp4"),
    title="示例标题",
    description="示例简介",
    tags=["运动", "训练"],
    publish_date=0,               # 无 --schedule
    publish_strategy="immediate",
    headless=True,
    debug=False,
)
await upload_video(request)
```

```python
# sau_cli.py:upload_video
account_file = resolve_account_file("douyin", "my_douyin")
is_ready = await douyin_setup(str(account_file), handle=False)
# handle=False → cookie 无效时不登录，直接返回 False
if not is_ready:
    raise RuntimeError("Douyin cookie is missing or expired...")

app = DouYinVideo(
    title="示例标题",
    file_path="videos/demo.mp4",
    tags=["运动", "训练"],
    publish_date=0,
    account_file=str(account_file),
    desc="示例简介",
    publish_strategy="immediate",
    headless=True,
    debug=False,
)
await app.douyin_upload_video()
```

### 5.3 DouYinVideo.upload：Playwright 自动化全流程

```
validate_upload_args()
  ├─ validate_base_args()
  │     ├─ 检查 account_file 存在
  │     ├─ cookie_auth() 验证 cookie
  │     ├─ publish_strategy 合法性
  │     └─ publish_strategy==immediate → publish_date=0
  └─ 验证视频文件（BaseVideoUploader.validate_video_file）
        ├─ 文件存在性
        └─ 后缀在 {.mp4,.mov,.avi,.mkv,.m4v,.webm,.flv,.wmv}

playwright.chromium.launch(headless=True, channel="chrome")
context = new_context(storage_state=account_file, permissions=["geolocation"])
set_init_script(context)   ← stealth.min.js

page.goto("https://creator.douyin.com/creator-micro/content/upload")
wait_for_url(该 URL)

# 设置视频文件
page.locator("div[class^='container'] input").set_input_files("videos/demo.mp4")

# 等待跳转到发布页（两个 URL 版本）
wait_for_url("...content/publish?enter_from=publish_page")    # version_1
 或
wait_for_url("...content/post/video?enter_from=publish_page") # version_2

# 填写元数据
fill_title_and_description(page, "示例标题", "示例简介", ["运动","训练"])
  ├─ 定位"作品描述" → title input → fill("示例标题"[:30])
  ├─ 定位 contenteditable → 清空 → keyboard.type("示例简介")
  └─ for tag in ["运动","训练"]:
         keyboard.type(" #运动") → keyboard.press("Space")
         keyboard.type(" #训练") → keyboard.press("Space")

# 轮询上传进度
while True:
    if "重新上传" visible → 上传完成 break
    if "上传失败" → handle_upload_error()（重新设置文件）
    sleep(2)

# 可选：商品链接（本例无）
# 可选：封面（本例无）

# 第三方分享开关（若存在且未开启则点击）

# 发布策略 = immediate → 跳过定时设置

# 点击发布
while True:
    click 发布按钮
    wait_for_url("...content/manage**", timeout=3000)
    → 成功 break
    except → handle_auto_video_cover()（若提示需封面则自动选推荐封面）
    sleep(0.5)

# 更新 cookie
context.storage_state(path=account_file)
sleep(2)
context.close() / browser.close()
```

### 5.4 输出

```
Douyin video upload submitted: videos/demo.mp4
```

---

## 六、`sau douyin upload-note`

### 6.1 参数解析

子命令 `douyin → upload-note`：

| 参数 | 类型 | 说明 |
|------|------|------|
| `--account` | str | 账号名 |
| `--images` | `nargs="+"` | 一个或多个图片路径（检查存在性）|
| `--title` | str | 图文标题 |
| `--note` | str | 正文内容，默认空 |
| `--tags` | str | 逗号分隔 |
| `--schedule` | datetime | 定时发布 |

### 6.2 dispatch → upload_note

```python
request = DouyinNoteUploadRequest(
    account_name="my_douyin",
    image_files=[Path("videos/demo1.png"), Path("videos/demo2.png")],
    title="图文标题",
    note="图文示例",
    tags=["图文", "测试"],
    publish_date=0,
    publish_strategy="immediate",
    headless=True,
    debug=False,
)
await upload_note(request)
```

```python
# sau_cli.py:upload_note
account_file = resolve_account_file("douyin", "my_douyin")
is_ready = await douyin_setup(str(account_file), handle=False)
# cookie 验证，不通过则 raise RuntimeError

app = DouYinNote(
    image_paths=["videos/demo1.png", "videos/demo2.png"],
    title="图文标题",
    note="图文示例",
    tags=["图文", "测试"],
    publish_date=0,
    account_file=str(account_file),
    publish_strategy="immediate",
    headless=True,
    debug=False,
)
await app.douyin_upload_note()
```

### 6.3 DouYinNote.upload：Playwright 自动化全流程

```
validate_upload_args()
  ├─ validate_base_args()（同视频流程）
  ├─ title 不为空
  ├─ image_paths 不为空
  ├─ 图片数量 ≤ 35
  └─ 每张图片：validate_image_file()
        ├─ 文件存在性
        └─ 后缀在 {.jpg,.jpeg,.png,.webp,.bmp}

playwright.chromium.launch(headless=True)
context = new_context(storage_state=account_file, permissions=["geolocation"])
set_init_script(context)

page.goto("https://creator.douyin.com/creator-micro/content/upload")
wait_for_url(上传页)

upload_note_content(page)：
  # 1. 切换到图文模式
  page.get_by_text("发布图文", exact=True).click()
  wait_for_timeout(1000)

  # 2. 上传图片
  page.locator("div[class^='container'] input[accept*='image']")
      .set_input_files(["demo1.png", "demo2.png"])

  # 3. 等待跳转图文发布页
  wait_for_url("**/creator-micro/content/post/image?**")

  sleep(1)

  # 4. 填写元数据
  fill_title_and_description(page, "图文标题", "图文示例", ["图文","测试"])

  # 5. 定时发布（本例无 --schedule → 跳过）

  # 6. 发布循环
  while True:
      # 检测短信验证码弹窗（大量发布时平台可能触发）
      sms_title = get_by_text("短信验证码").first
      if visible → 打印警告，等人工填写，sleep(2) continue

      # 点击发布
      publish_button = get_by_role("button", name=re.compile(r"发布"))
      click()
      wait_for_url("**/creator-micro/content/manage?enter_from=publish**", timeout=3000)
      → 成功 break
      except → 兜底点击 name="发布" exact=True 按钮

# 成功后更新 cookie
context.storage_state(path=account_file)
sleep(2)
context.close() / browser.close()
```

### 6.4 输出

```
Douyin note upload submitted: 2 images
```

---

## 七、关键设计点汇总

### 7.1 account_file 命名规则

```
BASE_DIR/cookies/{platform}_{account_name}.json
```

不同 `account_name` 互相隔离，支持同平台多账号并发使用。

### 7.2 stealth.min.js 反检测

所有 Playwright context 创建后都调用 `set_init_script(context)`，注入 `utils/stealth.min.js`，规避抖音的 headless 检测。

### 7.3 cookie 持久化时机

| 时机 | 操作 |
|------|------|
| 登录成功后 | `context.storage_state(path=account_file)` |
| 视频上传成功后 | `context.storage_state(path=account_file)`（刷新 session）|
| 图文上传成功后 | 同上 |

upload 流程会刷新 cookie，延长有效期。

### 7.4 publish_strategy 决定逻辑

```python
# sau_cli.py:dispatch
publish_strategy = DOUYIN_PUBLISH_STRATEGY_SCHEDULED if args.schedule else DOUYIN_PUBLISH_STRATEGY_IMMEDIATE
```

只要传了 `--schedule`，自动切换定时策略；不传默认立即发布，无需手动指定策略。

### 7.5 定时发布时间约束

`BaseVideoUploader.validate_publish_date()`：

- 必须是 `datetime` 类型
- 必须 > 当前时间
- 必须 > 当前时间 + 2 小时（`MIN_SCHEDULE_LEAD_TIME`）

### 7.6 标题长度截断

`fill_title_and_description()` 中：

```python
await title_input.fill(title[:30])
```

超过 30 字自动截断，不报错。

### 7.7 上传失败重试

视频上传检测到"上传失败"后，自动调用 `handle_upload_error(page)` 重新 `set_input_files`，无需人工干预。

### 7.8 QR 码处理流程

```
page 中提取 data:image/png;base64,...
  → base64 decode → 写 PNG 文件（cookies/ 目录下）
  → cv2.QRCodeDetector 解码内容
  → segno.make() 终端打印（Unicode 失败则降级 ASCII）
  → 等待扫码成功
  → 清理临时 PNG 文件
```

---

## 八、命令对比速查

| 命令 | 核心函数 | 是否需要已有 cookie | 是否打开浏览器 |
|------|----------|---------------------|----------------|
| `login` | `douyin_cookie_gen` | 否（新建）| 是（用于扫码）|
| `login --headed` | `douyin_cookie_gen` | 否（新建）| 是（可见窗口）|
| `check` | `cookie_auth` | 是 | 是（无头验证）|
| `upload-video` | `DouYinVideo.upload` | 是 | 是（无头上传）|
| `upload-note` | `DouYinNote.upload` | 是 | 是（无头上传）|

标准使用顺序：

```
login → check → upload-video / upload-note
```

---

## 九、错误处理

| 场景 | 错误信息 | 处理方式 |
|------|----------|----------|
| cookie 文件不存在 | `cookie is missing` | 重新 login |
| cookie 已失效 | `cookie is expired` | 重新 login |
| 视频文件不存在 | `视频文件不存在: ...` | 检查 --file 路径 |
| 视频格式不支持 | `不支持的视频格式: ...` | 转换为 .mp4 等 |
| 图片超过 35 张 | `最多只支持上传 35 张图片` | 分批上传 |
| 定时时间太近 | `必须大于当前时间 2 小时` | 调整 --schedule |
| 扫码超时 | `等待抖音扫码登录超时` | 重新 login |
| 短信验证码弹窗 | 日志警告 + 等待 | 手动在浏览器中填写 |
