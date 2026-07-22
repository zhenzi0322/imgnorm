<p align="center">
  <h1>imgnorm</h1>
  <a href="https://pypi.org/project/imgnorm/"><img src="https://img.shields.io/pypi/v/imgnorm.svg" alt="PyPI version"></a>
  <a href="https://pypi.org/project/imgnorm/"><img src="https://img.shields.io/badge/Python-3.8-3776AB?logo=python&logoColor=white" alt="Python"></a>
  <a href="https://github.com/zhenzi0322/imgnorm/blob/main/LICENSE"><img src="https://img.shields.io/pypi/l/imgnorm.svg" alt="License"></a>
          <a href="https://tool.long920.cn/imgnorm"><img src="https://app.readthedocs.org/projects/zhenzi0322-tool/badge/?version=latest" alt="Documentation Status"></a>
</p>

> `AI`图像处理工具集 —— 涵盖图片等比缩放、`Flux Kontext / GPT` 图像尺寸适配、比例分析、违禁词检测、`ComfyUI`服务管理等场景。

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
