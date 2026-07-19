<p align="center">
  <h1>imgnorm</h1>
  <b>AI 图像处理工具集 —— 涵盖图片等比缩放、<code>Flux Kontext / GPT</code> 图像尺寸适配、比例分析、违禁词检测、<code>ComfyUI</code> 服务管理及统一错误码体系，适用于 AI 图像生成预处理、批量归一化等场景。</b>
  <br><br>
  <a href="https://pypi.org/project/imgnorm/"><img src="https://img.shields.io/pypi/v/imgnorm.svg" alt="PyPI version"></a>
  <a href="https://pypi.org/project/imgnorm/"><img src="https://img.shields.io/badge/Python-3.8-3776AB?logo=python&logoColor=white" alt="Python"></a>
  <a href="https://github.com/zhenzi0322/imgnorm/blob/main/LICENSE"><img src="https://img.shields.io/pypi/l/imgnorm.svg" alt="License"></a>
  <a href="https://tool.long920.cn/imgnorm"><img src="https://app.readthedocs.org/projects/zhenzi0322-tool/badge/?version=latest" alt="Documentation Status"></a>
</p>

-----

## 特性

* **图片缩放与归一化**
  * `image_resize`：等比缩放并居中放置到指定画布，支持自定义背景色
  * `ImageTool`：多功能比例调整，支持标准比例匹配、`fill`/`resize`双模式、自定义放大百分比

* **AI 模型尺寸适配**
  * `FluxKontextImageScale`：自动匹配 17 个 Flux Kontext 优选分辨率并缩放
  * `GPTImageSizeCalculator`：根据比例 + 分辨率档位（1k/2k/4k）生成 `gpt-image-2` 合法尺寸

* **图片比例分析**
  * `ImageRatioAnalyzer`：获取精确最简比例（GCD 约分）或近似简单整数比例

* **内容安全**
  * `PromptBanChecker`：正则预编译违禁词检测，支持忽略大小写、长词优先

* **ComfyUI 服务管理**
  * `ComfyUIManager`：一键触发重启 + 轮询等待服务恢复

* **统一错误码与响应体系**
  * `ErrorCode`：20+ 内置错误码，覆盖通用、图片生成、内容安全、账户配额等场景
  * `ServiceResult`：统一接口返回，支持动态错误规则注入与自动消息替换
  * `CustomErrorCode` / `ErrorCodeRegistry`：自定义错误码扩展与全局注册

* **工具函数**
  * `str_to_md5`：字符串 / bytes 转 MD5
  * `is_valid_image`：图片有效性检测，损坏时抛出带上下文的 `InvalidImageError`

* **完整异常体系**
  * 10 个异常类继承自 `ImgnormError`，支持 `**kwargs` 透传任意上下文，可统一捕获

## 安装

```bash
pip install imgnorm
```

## 使用示例

### image_resize — 等比缩放居中

```python
from PIL import Image
from imgnorm import image_resize

# 打开图片
img = Image.open("input.jpg")

# 缩放到 640x480，默认黑色背景
result = image_resize(img, 640, 480)
result.save("output.jpg")

# 自定义背景颜色（白色）
result = image_resize(img, 640, 480, bg_color=(255, 255, 255))
result.save("output_white.jpg")
```

### FluxKontextImageScale — Flux Kontext 优选分辨率缩放

```python
from PIL import Image
from imgnorm import FluxKontextImageScale

img = Image.open("input.jpg")

# 自动匹配最接近原图宽高比的 Flux Kontext 优选分辨率并缩放
scaler = FluxKontextImageScale(img)
result = scaler.start()
result.save("output_kontext.jpg")
```

### ImageTool — 多功能比例调整工具

```python
from imgnorm import ImageTool

# 读取图片二进制数据
with open("input.jpg", "rb") as f:
    image_bytes = f.read()

# 自动匹配最接近的标准比例，最小边放大 10%（默认）并填充透明背景
tool = ImageTool()
result_bytes = tool.adjust_size(image_bytes)

# 自定义放大百分比（放大 20%）
result_bytes = tool.adjust_size(image_bytes, expand_percent=20)

# 不缩放（expand_percent=0）
result_bytes = tool.adjust_size(image_bytes, expand_percent=0)

# 指定目标比例 3:4，使用 resize 模式
tool = ImageTool(proportion="3:4", need_adjust="resize")
result_bytes = tool.adjust_size(image_bytes)

# 指定目标比例 1:1，自定义填充色（白色不透明）
tool = ImageTool(proportion="1:1", need_adjust="fill", fill_color=(255, 255, 255, 255))
result_bytes, matched_ratio = tool.adjust_size(image_bytes, return_origin_ratio=True)
print(f"原图匹配比例: {matched_ratio}")

# 保存结果
with open("output.png", "wb") as f:
    f.write(result_bytes)
```

### GPTImageSizeCalculator — gpt-image-2 尺寸计算器

```python
from imgnorm import GPTImageSizeCalculator

# 计算 16:9 比例、2K 分辨率的合法尺寸
result = GPTImageSizeCalculator.calc_size(aspect_ratio="16:9", resolution="2k")
print(result)
# {'width': 2560, 'height': 1440, 'size': '2560x1440', 'pixels': 3686400, 'aspect_ratio': '16:9', 'resolution': '2k', 'valid': True}

# 计算 1:1 比例、4K 分辨率的合法尺寸
result = GPTImageSizeCalculator.calc_size(aspect_ratio="1:1", resolution="4k")
print(result['size'])  # 2880x2880

# 校验尺寸是否合法
is_valid = GPTImageSizeCalculator.is_valid_image_size(1024, 1024)  # True
```

### ImageRatioAnalyzer — 图片比例分析器

```python
from imgnorm import ImageRatioAnalyzer

with open("input.jpg", "rb") as f:
    image_bytes = f.read()

analyzer = ImageRatioAnalyzer()

raw_ratio = analyzer.get_raw_simple_ratio(image_bytes)
# 精确比例: 74:55
print(f"精确比例: {raw_ratio}")

approx_ratio = analyzer.get_approx_simple_ratio(image_bytes)
# 近似比例: 4:3
print(f"近似比例: {approx_ratio}")

# 调整搜索范围（默认 1~10，增大可匹配更复杂比例）
analyzer = ImageRatioAnalyzer(approx_search_range=20)
# 19:14
print(analyzer.get_approx_simple_ratio(image_bytes))
# 74:55
print(analyzer.get_raw_simple_ratio(image_bytes))
```

### ComfyUIManager — ComfyUI 服务管理

```python
import logging
from imgnorm import ComfyUIManager

# 配置日志（可选）
logger = logging.getLogger("comfyui")
logging.basicConfig(level=logging.DEBUG)

# 创建管理器（默认连接本地 8188 端口）
manager = ComfyUIManager(logger=logger)

# 发起重启并等待服务恢复
success = manager.run_reboot_and_wait()
if success:
    print("服务重启成功")
else:
    print("服务重启超时")

# 自定义配置
manager = ComfyUIManager(
    base_url="http://127.0.0.1:8188",
    timeout=10,
    max_wait_seconds=120,
    poll_interval=3.0,
    logger=logger
)
```

### PromptBanChecker — 提示词违禁词检测

```python
from imgnorm import PromptBanChecker, PromptBanError

# 初始化（需提供违禁词文件路径，每行一个违禁词）
checker = PromptBanChecker(ban_word_file="banned_words.txt")

# 校验提示词
try:
    checker.check("some prompt text here")
    print("校验通过")
except PromptBanError as e:
    print(f"违禁词拦截: {e.error_msg}")
    print(f"命中词: {e.extend_data}")

# 查找所有命中词（不抛异常）
hits = checker.find_matched_words("text to check")
print(f"命中: {hits}")

# 重新加载违禁词文件
checker.load_banned_words()
```

### ErrorCode + ServiceResult — 统一错误码与响应

```python
from imgnorm import ErrorCode, ServiceResult, ResultStatus

# 构建成功响应
result = ServiceResult.success(data={"image_url": "https://xxx.com"})
# {'status': 'success', 'data': {'image_url': 'https://xxx.com'}, 'code': 'SUCCESS', 'msg': 'success', 'status_code': 200}
print(result.to_dict())

# 构建失败响应（使用 ErrorCode 默认消息）
result = ServiceResult.fail(ErrorCode.SERVICE_BUSY)
# {'status': 'fail', 'data': None, 'code': 'SERVICE_BUSY', 'msg': '服务繁忙，请稍后再试', 'status_code': 500}
print(result.to_dict())

# 构建失败响应（自定义消息覆盖）
result = ServiceResult.fail(ErrorCode.IMAGE_GENERATE_FAIL, msg="自定义错误信息")
# {'status': 'fail', 'data': None, 'code': 'IMAGE_GENERATE_FAIL', 'msg': '自定义错误信息', 'status_code': 500}
print(result.to_dict())

# 自动识别上游错误并替换为友好提示
result: ServiceResult = ServiceResult.fail(
    ErrorCode.IMAGE_GENERATE_FAIL,
    msg="Upstream model timed out. Try again later."
)
# {'status': 'fail', 'data': None, 'code': 'IMAGE_GENERATE_FAIL', 'msg': '服务超时，请稍后重试', 'status_code': 500}
print(result.to_dict())
# fail IMAGE_GENERATE_FAIL 服务超时，请稍后重试 500
print(result.status, result.code, result.msg, result.status_code)
# msg 自动替换为 "服务超时，请稍后重试"
```

**动态注入自定义错误匹配规则**

> 格式1：(关键词列表, ErrorCode) — 匹配时替换为 ErrorCode 默认消息
>
> 格式2：(关键词列表, ErrorCode, 自定义消息) — 匹配时替换为自定义消息

```python
from imgnorm import ErrorCode, ServiceResult

# 方式一：直接创建自定义错误码
ServiceResult._error_rules = [
    (["insufficient balance", "余额不足"], ErrorCode.INSUFFICIENT_BALANCE),
    (["upstream model timed out"], ErrorCode.TIMEOUT),
    (["no image", "未返回图片"], ErrorCode.NO_IMAGE_RESULT, "未返回图片"),
]
result = ServiceResult.fail(ErrorCode.INSUFFICIENT_BALANCE, msg='no image xxx')
# {'status': 'fail', 'data': None, 'code': 'INSUFFICIENT_BALANCE', 'msg': '未返回图片', 'status_code': 500}
print(result.to_dict())
```

### CustomErrorCode — 自定义错误码扩展

```python
from imgnorm import CustomErrorCode, ErrorCodeRegistry, ServiceResult

# 方式一：直接创建自定义错误码
MY_ERROR = CustomErrorCode("MY_ERROR", "自定义错误消息")
result = ServiceResult.fail(MY_ERROR)
# {'status': 'fail', 'data': None, 'code': 'MY_ERROR', 'msg': '自定义错误消息', 'status_code': 500}
print(result.to_dict())

# 方式二：通过注册表注册（可全局查找）
RATE_LIMIT = ErrorCodeRegistry.register("RATE_LIMIT", "请求过于频繁，请稍后再试")
result = ServiceResult.fail(RATE_LIMIT)
# {'status': 'fail', 'data': None, 'code': 'RATE_LIMIT', 'msg': '请求过于频繁，请稍后再试', 'status_code': 500}
print(result.to_dict())

# 查找已注册的错误码
found = ErrorCodeRegistry.get("RATE_LIMIT")
# <class 'imgnorm.response.error_code_registry.CustomErrorCode'> CustomErrorCode('RATE_LIMIT', '请求过于频繁，请稍后再试')
print(type(found), found)
# RATE_LIMIT 请求过于频繁，请稍后再试
print(found.code, found.msg)

# 列出所有已注册的错误码
all_codes = ErrorCodeRegistry.list_all()
# {'RATE_LIMIT': CustomErrorCode('RATE_LIMIT', '请求过于频繁，请稍后再试')}
print(all_codes)
```

### str_to_md5 — 字符串转 MD5

```python
from imgnorm import str_to_md5

# 字符串转 MD5
hash_str = str_to_md5("hello world")
print(hash_str)  # 5eb63bbbe01eeed093cb22bb8f5acdc3

# bytes 输入
hash_bytes = str_to_md5(b"binary data")
```

### is_valid_image — 图片有效性检测

```python
from imgnorm import is_valid_image, InvalidImageError

with open("input.jpg", "rb") as f:
    image_bytes = f.read()

# 传递额外上下文（如 task_id）
try:
    is_valid_image(image_bytes, task_id="abc123", user_id=42)
    print("图片有效")
except InvalidImageError as e:
    print(f"图片损坏: {e}")
    print(f"文件大小: {e.image_size} bytes")
    print(f"任务ID: {e.extra_data.get('task_id')}")  # abc123
    print(f"用户ID: {e.extra_data.get('user_id')}")  # 42
```

## API 说明

### `image_resize(image, target_w, target_h, bg_color=(0, 0, 0))`

将图片等比缩放并居中放置到指定尺寸的画布上。

| 参数 | 类型 | 说明 |
|------|------|------|
| `image` | `PIL.Image.Image` | 输入图片对象 |
| `target_w` | `int` | 目标宽度 |
| `target_h` | `int` | 目标高度 |
| `bg_color` | `tuple` | 画布背景颜色（RGB 元组），默认黑色 `(0, 0, 0)` |

**返回值：** 缩放并居中后的 `PIL.Image.Image` 对象。

**处理逻辑：**
1. 创建指定尺寸和背景色的画布
2. 计算等比缩放比例（取宽、高缩放比的最小值），确保图片完整放入画布
3. 使用 LANCZOS 算法高质量缩放图片
4. 计算居中偏移量，将缩放后的图片粘贴到画布中央

### `FluxKontextImageScale(image)`

根据原图宽高比，将图片缩放到最接近的 Flux Kontext 优选分辨率。

| 参数 | 类型 | 说明 |
|------|------|------|
| `image` | `PIL.Image.Image` | 输入图片对象 |

**方法 `start()`：** 计算原图宽高比，从 17 个优选分辨率中匹配最接近的一个，使用 LANCZOS 算法缩放并返回结果图片。

**优选分辨率列表（宽×高）：**
`672×1568` · `688×1504` · `720×1456` · `752×1392` · `800×1328` · `832×1248` · `880×1184` · `944×1104` · `1024×1024` · `1104×944` · `1184×880` · `1248×832` · `1328×800` · `1392×752` · `1456×720` · `1504×688` · `1568×672`

**返回值：** 缩放后的 `PIL.Image.Image` 对象。

### `ImageTool(proportion=None, need_adjust="fill", fill_color=(255,255,255,0))`

多功能图片尺寸调整工具，支持标准比例匹配和两种调整模式。

| 参数 | 类型 | 说明 |
|------|------|------|
| `proportion` | `str \| None` | 目标比例，如 `"3:4"` / `"1:1"`；空字符串或 `None` 表示自动匹配 |
| `need_adjust` | `str` | 调整模式：`"fill"`（画布填充）或 `"resize"`（拉伸） |
| `fill_color` | `tuple` | 填充背景色 RGBA，默认透明 `(255, 255, 255, 0)` |

**方法 `adjust_size(image_bytes, return_origin_ratio=False, expand_percent=10)`：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `image_bytes` | `bytes` | 图片二进制数据 |
| `return_origin_ratio` | `bool` | 是否同时返回原图匹配的标准比例字符串 |
| `expand_percent` | `float` | 放大百分比，默认 10（放大 10%）；传 0 不缩放，负数缩小 |

**返回值：** `bytes`（处理后图片数据）；当 `return_origin_ratio=True` 时返回 `(bytes, str)`。

**标准比例池：** `1:1` · `2:3` · `3:2` · `3:4` · `4:3` · `4:5` · `5:4` · `9:16` · `16:9` · `21:9`

**处理逻辑：**
1. `proportion` 存在且与原图最接近标准比例不一致 → 直接返回原图
2. `proportion` 存在且一致 → 反转比例后执行 resize/fill
3. `proportion` 为空且 `need_adjust="fill"` → 最小边按 `expand_percent` 放大，画布填充
4. `proportion` 为空且 `need_adjust="resize"` → 单边按 `expand_percent` 放大后拉伸

### `GPTImageSizeCalculator`

gpt-image-2 尺寸计算器，根据比例和分辨率自动生成合法图片尺寸。

**类方法：**

| 方法 | 说明 |
|------|------|
| `calc_size(aspect_ratio="1:1", resolution="2k")` | 根据比例和分辨率计算合法尺寸 |
| `is_valid_image_size(width, height)` | 校验尺寸是否合法 |
| `round_to_multiple(value, multiple=16)` | 四舍五入到指定倍数 |
| `floor_to_multiple(value, multiple=16)` | 向下对齐到指定倍数，用于像素超限时压缩 |

**`calc_size` 参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `aspect_ratio` | `str` | 宽高比，如 `"1:1"` / `"16:9"` / `"4:3"` |
| `resolution` | `str` | 分辨率档位：`"1k"` / `"2k"` / `"4k"` |

**返回值：** `dict`，包含 `width`、`height`、`size`、`pixels`、`aspect_ratio`、`resolution`、`valid`。

**合法性约束：**
- 最大边 ≤ 3840
- 宽高为 16 的倍数
- 宽高比 ≤ 3:1
- 像素范围：655,360 ~ 8,294,400

### `ImageRatioAnalyzer(approx_search_range=10)`

图片比例分析工具，获取图片的宽高比信息。

| 参数 | 类型 | 说明 |
|------|------|------|
| `approx_search_range` | `int` | 近似比例搜索整数范围，默认 1~10 |

**方法：**

| 方法 | 说明 |
|------|------|
| `get_raw_simple_ratio(image_bytes)` | 获取精确最简宽高比（GCD 约分） |
| `get_approx_simple_ratio(image_bytes)` | 自动拟合近似简单整数比例 |

**`get_raw_simple_ratio`：** 通过最大公约数约分，返回精确的最简比例字符串，如 `1920x1080` → `"16:9"`。

**`get_approx_simple_ratio`：** 遍历 `1~approx_search_range` 范围内的整数组合，找到与图片实际比例最接近的简单整数比，返回如 `"2:3"`、`"16:9"`。

**返回值：** 比例字符串（`str`）。

### `ComfyUIManager`

ComfyUI 服务管理工具，位于 `imgnorm.comfyui` 子模块。

| 参数 | 类型 | 说明 |
|------|------|------|
| `base_url` | `str` | 服务地址，默认 `"http://127.0.0.1:8188"` |
| `timeout` | `int` | 请求超时秒数，默认 `5` |
| `max_wait_seconds` | `int` | 最长等待恢复时间，默认 `60` |
| `poll_interval` | `float` | 轮询间隔秒数，默认`2.0` |
| `logger` | `logging.Logger` | 日志记录器，默认使用模块`logger` |

**方法：**

| 方法 | 说明 |
|------|------|
| `trigger_reboot()` | 发起重启请求，远程断开视为成功 |
| `wait_service_ready()` | 轮询检测服务是否恢复，超时返回`False` |
| `run_reboot_and_wait()` | 完整流程：重启 + 等待，返回`bool` |

**工作原理：**
1. 调用 `/api/manager/reboot` 接口触发重启
2. 轮询 `/api/system_stats` 接口，状态码 200 判定恢复
3. 超时未恢复返回 `False`

**安装：** 使用 comfyui 功能需额外安装 `requests`：
```bash
pip install imgnorm[comfyui]
# 或
pip install requests>=2.20.0
```

### `PromptBanChecker(ban_word_file, ignore_case=True)`

提示词违禁词校验工具，从文件加载违禁词列表，使用正则预编译实现高效匹配。

| 参数 | 类型 | 说明 |
|------|------|------|
| `ban_word_file` | `str` | 违禁词文件路径（每行一个词） |
| `ignore_case` | `bool` | 是否忽略大小写，默认 `True` |

**方法：**

| 方法 | 说明 |
|------|------|
| `check(prompt)` | 校验提示词，命中则抛出 `PromptBanError` |
| `find_matched_words(text)` | 查找所有命中词，返回列表 |
| `load_banned_words()` | 重新加载违禁词文件 |

### `PromptBanError`

违禁词拦截异常，继承自 `Exception`。

| 属性 | 类型 | 说明 |
|------|------|------|
| `error_msg` | `str` | 错误描述 |
| `extend_data` | `Any` | 命中的违禁词列表 |

```python
from imgnorm import PromptBanError
# 或
from imgnorm.exceptions import PromptBanError
```

### `InvalidImageError`

图片无效异常，继承自 `Exception`。当图片损坏、格式错误或无法解码时抛出。

| 属性 | 类型 | 说明 |
|------|------|------|
| `error_msg` | `str` | 错误描述 |
| `image_size` | `int` | 图片数据大小（字节数） |
| `original_error` | `Exception` | 原始异常 |
| `extra_data` | `dict` | 额外上下文数据（如 task_id 等） |

```python
from imgnorm import InvalidImageError
# 或
from imgnorm.exceptions import InvalidImageError
```

### `ErrorCode`

统一错误码枚举，位于 `imgnorm.response`。

**通用：**

| 成员 | code | 默认消息 |
|------|------|----------|
| `SUCCESS` | `"SUCCESS"` | 服务完成 |
| `SERVICE_BUSY` | `"SERVICE_BUSY"` | 服务繁忙，请稍后再试 |
| `TIMEOUT` | `"TIMEOUT"` | 服务超时，请稍后重试 |

**图片生成：**

| 成员 | code | 默认消息 |
|------|------|----------|
| `IMAGE_GENERATE_FAIL` | `"IMAGE_GENERATE_FAIL"` | 图片生成失败，请稍后重试 |
| `NO_IMAGE_RESULT` | `"NO_IMAGE_RESULT"` | 模型未返回图片结果,请调整图像或提示词后重试 |

**图片输入：**

| 成员 | code | 默认消息 |
|------|------|----------|
| `IMAGE_INVALID` | `"IMAGE_INVALID"` | 图片无效或已损坏，请上传有效的图片 |
| `IMAGE_FORMAT_UNSUPPORTED` | `"IMAGE_FORMAT_UNSUPPORTED"` | 不支持的图片格式 |
| `IMAGE_SIZE_EXCEEDED` | `"IMAGE_SIZE_EXCEEDED"` | 图片尺寸超出限制，请调整图片大小后重试 |

**内容安全：**

| 成员 | code | 默认消息 |
|------|------|----------|
| `CONTENT_POLICY_VIOLATION` | `"CONTENT_POLICY_VIOLATION"` | 您的请求包含违反平台政策的内容,请调整图像或提示词后重试 |
| `IMAGE_BANNED` | `"IMAGE_BANNED"` | 图片包含违禁内容，请更换图片后重试 |
| `PROMPT_BANNED` | `"PROMPT_BANNED"` | 提示词包含违禁内容，请修改后重试 |

**提示词：**

| 成员 | code | 默认消息 |
|------|------|----------|
| `PROMPT_EMPTY` | `"PROMPT_EMPTY"` | 提示词不能为空 |
| `PROMPT_TOO_LONG` | `"PROMPT_TOO_LONG"` | 提示词超出长度限制，请精简后重试 |

**参数：**

| 成员 | code | 默认消息 |
|------|------|----------|
| `INVALID_PARAMS` | `"INVALID_PARAMS"` | 请求参数无效，请检查后重试 |

**账户/配额：**

| 成员 | code | 默认消息 |
|------|------|----------|
| `INSUFFICIENT_BALANCE` | `"INSUFFICIENT_BALANCE"` | 服务API余额不足，联系客服充值后重试 |
| `QUOTA_EXCEEDED` | `"QUOTA_EXCEEDED"` | 请求配额已用完，请稍后重试或升级套餐 |
| `RATE_LIMIT` | `"RATE_LIMIT"` | 请求过于频繁，请稍后再试 |
| `AUTH_FAILED` | `"AUTH_FAILED"` | 认证失败，请检查 API Key 是否有效 |

**属性：** `.code`（错误标识）、`.msg`（用户可读描述）。

### `ServiceResult`

统一接口返回类，位于 `imgnorm.response`。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `status` | `str` | `"wait"` | 状态：success/fail/wait |
| `data` | `Any` | `None` | 返回数据 |
| `code` | `str` | `"SUCCESS"` | 错误码 |
| `msg` | `str` | `""` | 提示信息 |
| `status_code` | `int` | `400` | HTTP 状态码 |

**类方法：**

| 方法 | 说明 |
|------|------|
| `ServiceResult.success(data, msg)` | 快速构建成功响应 |
| `ServiceResult.fail(error, msg, status_code)` | 快速构建失败响应 |
| `result.to_dict()` | 转为字典 |

**动态错误规则：** 通过 `ServiceResult._error_rules` 注入自定义匹配规则：
- `(关键词列表, ErrorCode)` — 匹配时替换为 ErrorCode 默认消息
- `(关键词列表, ErrorCode, 自定义消息)` — 匹配时替换为自定义消息

**默认规则（`get_default_error_rules()`）：**
- `["insufficient balance", "余额不足"]` → `ErrorCode.INSUFFICIENT_BALANCE`
- `["upstream model timed out"]` → `ErrorCode.TIMEOUT`

### `ResultStatus`

返回状态枚举：`SUCCESS` / `FAIL` / `WAIT`。

### `CustomErrorCode(code, msg)`

自定义错误码类，用于扩展内置 `ErrorCode` 枚举。

| 参数 | 类型 | 说明 |
|------|------|------|
| `code` | `str` | 错误码标识字符串 |
| `msg` | `str` | 用户可读的错误描述 |

**属性：** `.code`、`.msg`。可直接传给 `ServiceResult.fail()` 使用。

### `ErrorCodeRegistry`

错误码注册表，提供全局注册和查找自定义错误码的功能。

| 方法 | 说明 |
|------|------|
| `register(code, msg)` | 注册并返回 CustomErrorCode 实例 |
| `get(code)` | 根据 code 查找已注册的错误码 |
| `list_all()` | 返回所有已注册的错误码字典 |
| `clear()` | 清空所有注册的错误码 |

### `str_to_md5(content)`

字符串转 MD5 哈希值。

| 参数 | 类型 | 说明 |
|------|------|------|
| `content` | `str \| bytes` | 待转换的字符串或 bytes |

**返回值：** 32 位小写 MD5 十六进制字符串。

### `is_valid_image(image_bytes, **kwargs)`

判断图片是否有效（未损坏）。

| 参数 | 类型 | 说明 |
|------|------|------|
| `image_bytes` | `bytes` | 图片二进制数据 |
| `**kwargs` | `Any` | 额外上下文数据，透传到 `InvalidImageError.extra_data` |

**返回值：** `True` 表示图片有效。

**异常：** 图片损坏时抛出 `InvalidImageError`，包含 `image_size` 和 `extra_data` 属性。

## 异常体系

所有异常继承自 `ImgnormError`，可通过 `except ImgnormError` 统一捕获。

```
ImgnormError                      # 基础异常（所有异常的父类）
├── ServiceError                  # 通用服务错误
├── AuthenticationError           # 认证失败，API Key 无效或过期
├── RateLimitError                # 请求频率超限
├── QuotaExceededError            # 配额/余额不足
├── ServiceTimeoutError           # 服务超时
├── InvalidParamsError            # 请求参数无效
├── PromptBanError                # 提示词违禁（含 extend_data 命中词列表）
├── ImageBanError                 # 图片包含违禁内容
└── InvalidImageError             # 图片损坏/格式错误（含 image_size、original_error）
```

```python
from imgnorm import ImgnormError, ServiceError, AuthenticationError
from imgnorm import RateLimitError, QuotaExceededError, ServiceTimeoutError
from imgnorm import InvalidParamsError, PromptBanError, ImageBanError, InvalidImageError

# 统一捕获所有 imgnorm 异常
try:
    ...
except ImgnormError as e:
    print(f"imgnorm 错误: {e.error_msg}")
    print(f"附加数据: {e.extra_data}")
```

每个异常都支持 `error_msg`（错误描述）和 `extra_data`（额外上下文数据）属性，创建时可通过 `**kwargs` 透传任意上下文。

