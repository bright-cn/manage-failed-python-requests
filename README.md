# 在 Python 中管理失败的请求

[![推广](https://github.com/bright-cn/Awesome-Web-Scraping/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner%20CN.png)](https://www.bright.cn/) 

本指南介绍如何使用重试策略和自定义逻辑处理 Python 中失败的 HTTP 请求。

- [什么是状态码？](#什么是状态码)
- [重试策略](#重试策略)
- [HTTPAdapter](#httpadapter)
- [Tenacity](#tenacity)
- [构建自定义重试机制](#构建自定义重试机制)
- [总结](#总结)

## 什么是状态码？

状态码是各种协议中用于表示请求结果的标准化三位数字。根据 [Mozilla](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) 的说明，HTTP 状态码可以分为以下几类：

- **100-199**：信息响应
- **200-299**：成功响应
- **300-399**：重定向消息
- **400-499**：客户端错误消息
- **500-599**：服务器错误消息

开发网页爬虫等客户端应用程序时，务必注意 400 和 500 范围内的状态码。400 范围内的状态码通常表示客户端错误，例如身份验证失败、速率限制、超时或广为人知的 _404：未找到错误_。500 范围内的状态码则表示服务器端问题，可能需要重试或采用其他处理策略。

以下是执行网页爬虫时常见的错误代码列表（摘自 Mozilla 的[官方文档](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#client_error_responses)）：

| **状态码** | **含义** | **说明** |
| --- | --- | --- |
| 400 | 错误请求 | 检查你的请求格式 |
| [401](https://www.bright.cn/faqs/proxy-errors/error-401-how-to-avoid) | 未授权 | 检查你的 API 密钥 |
| [403](https://www.bright.cn/faqs/proxy-errors/403-status-error-how-to-avoid) | 禁止访问 | 你无法访问这些数据 |
| 404 | 未找到 | 站点/端点不存在 |
| [408](https://www.bright.cn/faqs/proxy-errors/error-408-how-to-avoid) | 请求超时 | 请求已超时，请重试 |
| [429](https://www.bright.cn/faqs/proxy-errors/429-error-how-to-avoid) | 请求过多 | 降低请求频率 |
| 500 | 服务器内部错误 | 常规服务器错误，请重试请求 |
| 501 | 未实现 | 服务器暂不支持此功能 |
| [502](https://www.bright.cn/faqs/proxy-errors/502-error-how-to-avoid) | 错误网关 | 上游服务器响应失败 |
| [503](https://www.bright.cn/faqs/proxy-errors/503-error-how-to-avoid) | 服务不可用 | 服务器暂时停机，请稍后重试 |
| [504](https://www.bright.cn/faqs/proxy-errors/504-error-how-to-avoid) | 网关超时 | 等待上游服务器响应超时 |

## 重试策略

在 Python 中实现重试机制时，你可以使用 `HTTPAdapter` 和 `Tenacity` 等预构建库。你也可以根据具体需求开发自定义重试逻辑。

设计良好的重试策略应同时包含重试次数限制和退避机制。重试次数限制可以防止无限循环，确保失败的请求不会一直重试。退避策略会逐渐延长重试之间的等待时间，有助于避免因请求过多而被屏蔽或导致服务器过载。

- **重试次数限制**：定义重试次数限制至关重要。尝试指定次数 (X) 后，爬虫工具应停止重试，以免陷入无限循环。  
- **退避算法**：逐渐延长重试之间的等待时间，有助于避免服务器不堪重负。可以从较短的延迟开始，例如 0.3 秒，然后逐步增加到 0.6 秒、1.2 秒，依此类推。

## HTTPAdapter

使用 `HTTPAdapter` 时，我们需要配置三个参数：`total`、`backoff_factor` 和 `status_forcelist`。严格来说，`allowed_methods` 并非必需，但它有助于定义重试条件，从而提高代码的安全性。在以下代码中，我们使用 [httpbin](https://httpbin.org/) 自动触发错误并启动重试逻辑。

```python
import logging
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

# Configure logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")
logger = logging.getLogger(__name__)

# Create a session
session = requests.Session()

# Configure retry settings
retry = Retry(
    total=3,  # Maximum retries
    backoff_factor=0.3,  # Time between retries (exponential backoff)
    status_forcelist=(429, 500, 502, 503, 504),  # Status codes to trigger a retry
    allowed_methods={"GET", "POST"}  # Allow retries for GET and POST
)

# Mount the adapter with our custom settings
adapter = HTTPAdapter(max_retries=retry)
session.mount("http://", adapter)
session.mount("https://", adapter)

# Function to make a request and test retry logic
def make_request(url, method="GET"):
    try:
        logger.info(f"Making a {method} request to {url} with retry logic...")
        
        if method == "GET":
            response = session.get(url)
        elif method == "POST":
            response = session.post(url)
        else:
            logger.error("Unsupported HTTP method: %s", method)
            return
        
        response.raise_for_status()
        logger.info("✅ Request successful: %s", response.status_code)
    
    except requests.exceptions.RequestException as e:
        logger.error("❌ Request failed after retries: %s", e)
        logger.info("Retries attempted: %d", len(response.history) if response else 0)

# Test Cases
make_request("https://httpbin.org/status/200")  # ✅ Should succeed without retries
make_request("https://httpbin.org/status/500")  # ❌ Should retry 3 times and fail
make_request("https://httpbin.org/status/404")  # ❌ Should fail immediately (no retries)
make_request("https://httpbin.org/status/500", method="POST")  # ❌ Should retry 3 times and fail
```

创建 `Session` 对象后，执行以下操作：

- 创建 `Retry` 对象并定义：
    - `total`：请求的最大重试次数。
    - `backoff_factor`：两次重试之间的等待时间。随着重试次数增加，该时间会按指数调整。
    - `status_forcelist`：错误状态码列表。该列表中的任何状态码都会自动触发重试。
- 使用 `retry` 变量创建 `HTTPAdapter` 对象：`adapter = HTTPAdapter(max_retries=retry)`。
- 创建 `adapter` 后，使用 `session.mount()` 将其挂载到 HTTP 和 HTTPS 方法。

运行此代码时，系统将执行三次重试 (`total=3`)，然后你会看到以下输出。

```
2024-06-10 12:00:00 - INFO - Making a GET request to https://httpbin.org/status/200 with retry logic...
2024-06-10 12:00:00 - INFO - ✅ Request successful: 200

2024-06-10 12:00:01 - INFO - Making a GET request to https://httpbin.org/status/500 with retry logic...
2024-06-10 12:00:02 - ERROR - ❌ Request failed after retries: 500 Server Error: INTERNAL SERVER ERROR for url: ...
2024-06-10 12:00:02 - INFO - Retries attempted: 3

2024-06-10 12:00:03 - INFO - Making a GET request to https://httpbin.org/status/404 with retry logic...
2024-06-10 12:00:03 - ERROR - ❌ Request failed after retries: 404 Client Error: NOT FOUND for url: ...
2024-06-10 12:00:03 - INFO - Retries attempted: 0

2024-06-10 12:00:04 - INFO - Making a POST request to https://httpbin.org/status/500 with retry logic...
2024-06-10 12:00:05 - ERROR - ❌ Request failed after retries: 500 Server Error: INTERNAL SERVER ERROR for url: ...
2024-06-10 12:00:05 - INFO - Retries attempted: 3
```

## Tenacity

你还可以使用 [`Tenacity`](https://tenacity.readthedocs.io/en/latest/)，这是一个流行的 Python 开源重试库。它并不局限于 HTTP，并且提供了一种灵活清晰的重试实现方式。

首先安装 `Tenacity`：

```bash
pip install tenacity
```

安装完成后，创建一个_装饰器_，并用它封装请求函数。在 `@retry` 装饰器中添加 `stop`、`wait` 和 `retry` 参数。

```python
import logging
import requests
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type, retry_if_result, RetryError

# Configure logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")
logger = logging.getLogger(__name__)

# Define a retry strategy
@retry(
    stop=stop_after_attempt(3),  # Retry up to 3 times
    wait=wait_exponential(multiplier=0.3),  # Exponential backoff
    retry=(
        retry_if_exception_type(requests.exceptions.RequestException) |  # Retry on request failures
        retry_if_result(lambda r: r.status_code in {500, 502, 503, 504})  # Retry on specific HTTP status codes
    ),
)
def make_request(url):
    logger.info("Making a request with retry logic to %s...", url)
    response = requests.get(url)
    response.raise_for_status()
    logger.info("✅ Request successful: %s", response.status_code)
    return response

# Attempt to make the request
try:
    make_request("https://httpbin.org/status/500")  # Test with a failing status code
except RetryError as e:
    logger.error("❌ Request failed after all retries: %s", e)    
```

此处的逻辑和设置与第一个使用 `HTTPAdapter` 的示例非常相似：

- `stop=stop_after_attempt(3)`：指示 `tenacity` 在 3 次重试失败后停止尝试。
- `wait=wait_exponential(multiplier=0.3)` 使用与之前相同的等待方式，并且也会像之前一样进行指数退避。
- `retry=retry_if_exception_type(requests.exceptions.RequestException)` 指示 `tenacity` 每次发生 `RequestException` 时都使用此逻辑。
- `make_request()` 向我们的错误端点发送请求。它会继承上方所创建装饰器的全部特性。

运行此代码时，你会得到类似的输出：

```
2024-06-10 12:00:00 - INFO - Making a request with retry logic to https://httpbin.org/status/500...
2024-06-10 12:00:01 - WARNING - Retrying after 0.3 seconds...
2024-06-10 12:00:01 - INFO - Making a request with retry logic to https://httpbin.org/status/500...
2024-06-10 12:00:02 - WARNING - Retrying after 0.6 seconds...
2024-06-10 12:00:02 - INFO - Making a request with retry logic to https://httpbin.org/status/500...
2024-06-10 12:00:03 - ERROR - ❌ Request failed after all retries: RetryError[...]
```

## 构建自定义重试机制

你也可以创建自定义重试机制。在处理专用代码时，这通常是最佳方法。只需编写相对少量的代码，即可实现现有库提供的相同功能，同时根据具体需求进行调整。

以下代码演示了如何导入 `sleep` 以实现指数退避、完成配置（`total`、`backoff_factor` 和 `bad_codes`），以及使用 `while` 循环承载重试逻辑。`while` 循环会在你仍有剩余尝试次数且尚未成功时继续发送请求。

```python
import logging
import requests
from time import sleep

# Configure logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")
logger = logging.getLogger(__name__)

# Create a session
session = requests.Session()

# Define retry settings
TOTAL_RETRIES = 3
INITIAL_BACKOFF = 0.3
BAD_CODES = {429, 500, 502, 503, 504}

def make_request(url):
    current_tries = 0
    backoff = INITIAL_BACKOFF
    success = False

    while current_tries < TOTAL_RETRIES and not success:
        try:
            logger.info("Making a request with retry logic to %s...", url)
            response = session.get(url)
            
            if response.status_code in BAD_CODES:
                raise requests.exceptions.HTTPError(f"Received {response.status_code}, triggering retry")
            
            response.raise_for_status()
            logger.info("✅ Request successful: %s", response.status_code)
            success = True
            return response

        except requests.exceptions.RequestException as e:
            logger.error("❌ Request failed: %s, retries left: %d", e, TOTAL_RETRIES - current_tries - 1)
            if current_tries < TOTAL_RETRIES - 1:
                logger.info("⏳ Retrying in %.1f seconds...", backoff)
                sleep(backoff)
                backoff *= 2  # Exponential backoff
            current_tries += 1

    logger.error("🚨 Request failed after all retries.")
    return None

# Test Cases
make_request("https://httpbin.org/status/500")  # ❌ Should retry 3 times and fail
make_request("https://httpbin.org/status/200")  # ✅ Should succeed without retries
```

此处的实际逻辑由一个简单的 `while` 循环处理。

- 如果 `response.status_code` 位于 `bad_codes` 列表中，脚本将抛出异常。
- 如果请求失败，脚本将：
    - 在控制台中输出错误消息。
    - `sleep(backoff_factor)` 会在发送下一个请求前等待。
    - `backoff_factor = backoff_factor * 2` 会将下一次尝试的 `backoff_factor` 翻倍。
    - 递增 `current_tries`，避免程序无限停留在循环中。

以下是自定义重试代码的输出。

```
2024-06-10 12:00:00 - INFO - Making a request with retry logic to https://httpbin.org/status/500...
2024-06-10 12:00:01 - ERROR - ❌ Request failed: Received 500, triggering retry, retries left: 2
2024-06-10 12:00:01 - INFO - ⏳ Retrying in 0.3 seconds...
2024-06-10 12:00:02 - INFO - Making a request with retry logic to https://httpbin.org/status/500...
2024-06-10 12:00:03 - ERROR - ❌ Request failed: Received 500, triggering retry, retries left: 1
2024-06-10 12:00:03 - INFO - ⏳ Retrying in 0.6 seconds...
2024-06-10 12:00:04 - INFO - Making a request with retry logic to https://httpbin.org/status/500...
2024-06-10 12:00:05 - ERROR - ❌ Request failed: Received 500, triggering retry, retries left: 0
2024-06-10 12:00:05 - ERROR - 🚨 Request failed after all retries.
```

## 总结

为了避免各种请求失败，我们开发了[网络解锁器 API](https://www.bright.cn/products/web-unlocker)和[抓取浏览器](https://www.bright.cn/products/scraping-browser)等产品。这些工具会自动处理反爬措施、CAPTCHA 验证和 IP 屏蔽，即使面对最具挑战性的网站，也能确保网页爬虫顺畅高效地运行。

立即注册，开始免费试用。
