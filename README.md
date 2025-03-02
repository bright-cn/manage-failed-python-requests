# 在 Python 中管理失败的请求

[![Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://www.bright.cn/) 

本指南将介绍如何在 Python 中处理失败的 HTTP 请求，包括重试策略和自定义逻辑。

- [什么是状态码？](#什么是状态码)
- [重试策略](#重试策略)
- [HTTPAdapter](#httpadapter)
- [Tenacity](#tenacity)
- [构建自定义重试机制](#构建自定义重试机制)
- [结论](#结论)

## 什么是状态码？

状态码是用于各种协议、以三位数字表示请求结果的标准化方式。根据 [Mozilla](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) 的描述，HTTP 状态码可分为以下几个类别：

- **100-199**：信息性响应  
- **200-299**：成功响应  
- **300-399**：重定向消息  
- **400-499**：客户端错误消息  
- **500-599**：服务器错误消息  

在开发客户端应用（如网络爬虫）时，需要格外关注 400 和 500 范围内的状态码。400 范围通常表示客户端错误，如身份验证失败、被限流、请求超时或常见的 “404: Not Found（未找到）” 等。而 500 范围则表示服务器端错误，可能需要再次重试或使用其他应对策略。

以下是从 Mozilla [官方文档](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#client_error_responses) 中整理的一些常见错误码列表，你在进行网页抓取时可能会遇到：

| **状态码** | **含义** | **描述** |
| --- | --- | --- |
| 400 | Bad Request | 检查请求格式是否正确 |
| [401](https://www.bright.cn/faqs/proxy-errors/error-401-how-to-avoid) | Unauthorized | 检查你的 API Key |
| [403](https://brightdata.com/faqs/proxy-errors/403-status-error-how-to-avoid) | Forbidden | 你无权访问该数据 |
| 404 | Not Found | 站点/接口不存在 |
| [408](https://www.bright.cn/faqs/proxy-errors/error-408-how-to-avoid) | Request Timeout | 请求超时，请重试 |
| [429](https://www.bright.cn/faqs/proxy-errors/429-error-how-to-avoid) | Too Many Requests | 请求过于频繁，请放慢速度 |
| 500 | Internal Server Error | 通用服务器错误，可重试 |
| 501 | Not Implemented | 服务器尚不支持此功能 |
| [502](https://www.bright.cn/faqs/proxy-errors/502-error-how-to-avoid) | Bad Gateway | 上游服务器返回错误 |
| [503](https://www.bright.cn/faqs/proxy-errors/503-error-how-to-avoid) | Service Unavailable | 服务器暂时不可用，请稍后重试 |
| [504](https://www.bright.cn/faqs/proxy-errors/504-error-how-to-avoid) | Gateway Timeout | 上游服务器请求超时 |

## 重试策略

在 Python 中实施重试机制时，你可以使用现有库（例如 `HTTPAdapter` 或 `Tenacity`），或者根据自身需求编写自定义重试逻辑。

一个设计良好的重试策略应包含重试次数限制和退避（Backoff）机制。重试次数限制可以防止无限循环导致请求不断失败；而退避策略会逐步增加重试间隔，从而避免过多且频率过高的请求，减少被阻断或给服务器带来额外压力的风险。

- **重试次数限制**：需要设定一个最大重试次数（X）。超过该次数后，爬虫应停止重试，以防止陷入无限循环。  
- **退避算法**：在重试间隔上采用渐增策略可避免过度触发服务器。可以先以 0.3 秒为起点，然后逐渐增加到 0.6 秒、1.2 秒等等。

## HTTPAdapter

使用 `HTTPAdapter` 时，需要配置 `total`、`backoff_factor` 和 `status_forcelist` 三个主要参数。`allowed_methods` 并不是必需，但它能帮助我们明确需要触发重试的请求方法，从而让代码更安全。下面的示例使用 [httpbin](https://httpbin.org/) 来故意返回错误，以测试重试逻辑。

```python
import logging
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

# 配置日志
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")
logger = logging.getLogger(__name__)

# 创建会话
session = requests.Session()

# 配置重试设置
retry = Retry(
    total=3,  # 最大重试次数
    backoff_factor=0.3,  # 重试之间的等待时间（指数退避）
    status_forcelist=(429, 500, 502, 503, 504),  # 遇到这些错误码会触发重试
    allowed_methods={"GET", "POST"}  # 允许对 GET 和 POST 方法进行重试
)

# 将自定义设置与适配器进行绑定
adapter = HTTPAdapter(max_retries=retry)
session.mount("http://", adapter)
session.mount("https://", adapter)

# 发送请求并测试重试逻辑的函数
def make_request(url, method="GET"):
    try:
        logger.info(f"正在向 {url} 发送 {method} 请求，带重试逻辑...")
        
        if method == "GET":
            response = session.get(url)
        elif method == "POST":
            response = session.post(url)
        else:
            logger.error("不支持的 HTTP 方法: %s", method)
            return
        
        response.raise_for_status()
        logger.info("✅ 请求成功: %s", response.status_code)
    
    except requests.exceptions.RequestException as e:
        logger.error("❌ 重试后请求仍然失败: %s", e)
        logger.info("已尝试的重试次数: %d", len(response.history) if response else 0)

# 测试用例
make_request("https://httpbin.org/status/200")  # ✅ 无需重试即可成功
make_request("https://httpbin.org/status/500")  # ❌ 将重试 3 次后失败
make_request("https://httpbin.org/status/404")  # ❌ 不会重试，立即失败
make_request("https://httpbin.org/status/500", method="POST")  # ❌ 将重试 3 次后失败
```

在创建 `Session` 对象后，你需要进行如下步骤：

1. 创建一个 `Retry` 对象，定义：
    - `total`：请求最多重试的次数上限。  
    - `backoff_factor`：重试之间的等待时间，该数值会根据重试次数实现指数级增长。  
    - `status_forcelist`：会触发重试的状态码列表。
2. 创建一个 `HTTPAdapter` 对象并将我们定义的 `retry` 传入：  
   `adapter = HTTPAdapter(max_retries=retry)`
3. 使用 `session.mount()` 将创建好的 `adapter` 分别挂载到 HTTP 和 HTTPS 协议上。

当你运行上述代码，配置为 3 次重试（`total=3`）后，失败的请求会重试 3 次，然后你会看到类似下面的输出：

```
2024-06-10 12:00:00 - INFO - 正在向 https://httpbin.org/status/200 发送 GET 请求，带重试逻辑...
2024-06-10 12:00:00 - INFO - ✅ 请求成功: 200

2024-06-10 12:00:01 - INFO - 正在向 https://httpbin.org/status/500 发送 GET 请求，带重试逻辑...
2024-06-10 12:00:02 - ERROR - ❌ 重试后请求仍然失败: 500 Server Error: INTERNAL SERVER ERROR for url: ...
2024-06-10 12:00:02 - INFO - 已尝试的重试次数: 3

2024-06-10 12:00:03 - INFO - 正在向 https://httpbin.org/status/404 发送 GET 请求，带重试逻辑...
2024-06-10 12:00:03 - ERROR - ❌ 重试后请求仍然失败: 404 Client Error: NOT FOUND for url: ...
2024-06-10 12:00:03 - INFO - 已尝试的重试次数: 0

2024-06-10 12:00:04 - INFO - 正在向 https://httpbin.org/status/500 发送 POST 请求，带重试逻辑...
2024-06-10 12:00:05 - ERROR - ❌ 重试后请求仍然失败: 500 Server Error: INTERNAL SERVER ERROR for url: ...
2024-06-10 12:00:05 - INFO - 已尝试的重试次数: 3
```

## Tenacity

你也可以使用 [Tenacity](https://tenacity.readthedocs.io/en/latest/) 这个流行的第三方重试库。它不仅仅局限于处理 HTTP 请求，还能以更加灵活优雅的方式为各种操作实现重试逻辑。

首先安装 `Tenacity`：

```bash
pip install tenacity
```

安装完成后，可以通过创建一个装饰器来包裹请求函数，并在 `@retry` 装饰器里添加 `stop`、`wait` 和 `retry` 等参数。例如：

```python
import logging
import requests
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type, retry_if_result, RetryError

# 配置日志
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")
logger = logging.getLogger(__name__)

# 定义重试策略
@retry(
    stop=stop_after_attempt(3),  # 最多重试 3 次
    wait=wait_exponential(multiplier=0.3),  # 指数退避，初始间隔为 0.3
    retry=(
        retry_if_exception_type(requests.exceptions.RequestException) |  # 请求异常时重试
        retry_if_result(lambda r: r.status_code in {500, 502, 503, 504})  # 指定状态码时重试
    ),
)
def make_request(url):
    logger.info("正在向 %s 发送请求，带重试逻辑...", url)
    response = requests.get(url)
    response.raise_for_status()
    logger.info("✅ 请求成功: %s", response.status_code)
    return response

# 测试发起请求
try:
    make_request("https://httpbin.org/status/500")  # 测试失败状态码
except RetryError as e:
    logger.error("❌ 所有重试都失败: %s", e)    
```

与在 `HTTPAdapter` 中的设置类似：

- `stop=stop_after_attempt(3)`：告诉 `tenacity` 最多重试 3 次。  
- `wait=wait_exponential(multiplier=0.3)`：与之前的指数退避相同，初始退避时间为 0.3 秒，并逐步增加。  
- `retry=retry_if_exception_type(requests.exceptions.RequestException)`：遇到请求异常时会触发重试。  
- `make_request()` 函数实际发起请求并抛出异常或返回结果。

当你运行此示例时，你也会看到类似的输出：

```
2024-06-10 12:00:00 - INFO - 正在向 https://httpbin.org/status/500 发送请求，带重试逻辑...
2024-06-10 12:00:01 - WARNING - Retrying after 0.3 seconds...
2024-06-10 12:00:01 - INFO - 正在向 https://httpbin.org/status/500 发送请求，带重试逻辑...
2024-06-10 12:00:02 - WARNING - Retrying after 0.6 seconds...
2024-06-10 12:00:02 - INFO - 正在向 https://httpbin.org/status/500 发送请求，带重试逻辑...
2024-06-10 12:00:03 - ERROR - ❌ 所有重试都失败: RetryError[...]
```

## 构建自定义重试机制

如果你需要更灵活或更具针对性的策略，也可以自行编写一个简单的自定义重试机制。只需少量代码，就可以达到与第三方库类似的功能，并且可根据需求进行裁剪。

下面的示例展示了如何导入 `sleep` 来实现指数退避，定义配置（`total`、`backoff_factor` 和 `bad_codes`），并使用 `while` 循环实现重试逻辑。代码逻辑是：只要还有重试次数且未成功，就继续尝试请求。

```python
import logging
import requests
from time import sleep

# 配置日志
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")
logger = logging.getLogger(__name__)

# 创建会话
session = requests.Session()

# 定义重试设置
TOTAL_RETRIES = 3
INITIAL_BACKOFF = 0.3
BAD_CODES = {429, 500, 502, 503, 504}

def make_request(url):
    current_tries = 0
    backoff = INITIAL_BACKOFF
    success = False

    while current_tries < TOTAL_RETRIES and not success:
        try:
            logger.info("正在向 %s 发送请求，带重试逻辑...", url)
            response = session.get(url)
            
            if response.status_code in BAD_CODES:
                raise requests.exceptions.HTTPError(f"收到状态码 {response.status_code}，触发重试")
            
            response.raise_for_status()
            logger.info("✅ 请求成功: %s", response.status_code)
            success = True
            return response

        except requests.exceptions.RequestException as e:
            logger.error("❌ 请求失败: %s，剩余重试次数: %d", e, TOTAL_RETRIES - current_tries - 1)
            if current_tries < TOTAL_RETRIES - 1:
                logger.info("⏳ %.1f 秒后开始重试...", backoff)
                sleep(backoff)
                backoff *= 2  # 指数退避
            current_tries += 1

    logger.error("🚨 所有重试均失败。")
    return None

# 测试用例
make_request("https://httpbin.org/status/500")  # ❌ 将重试 3 次后失败
make_request("https://httpbin.org/status/200")  # ✅ 无需重试即可成功
```

实际逻辑由一个简单的 `while` 循环驱动：

- 如果 `response.status_code` 在 `BAD_CODES` 中，就主动抛出异常。  
- 如果请求抛出异常，脚本会：  
  - 打印错误信息；  
  - 等待 `sleep(backoff_factor)` 指定的时间后重试；  
  - `backoff_factor` 翻倍，形成指数退避；  
  - 递增 `current_tries`，避免无限循环。

运行上述自定义重试代码后，你会看到如下输出：

```
2024-06-10 12:00:00 - INFO - 正在向 https://httpbin.org/status/500 发送请求，带重试逻辑...
2024-06-10 12:00:01 - ERROR - ❌ 请求失败: 收到状态码 500，触发重试，剩余重试次数: 2
2024-06-10 12:00:01 - INFO - ⏳ 0.3 秒后开始重试...
2024-06-10 12:00:02 - INFO - 正在向 https://httpbin.org/status/500 发送请求，带重试逻辑...
2024-06-10 12:00:03 - ERROR - ❌ 请求失败: 收到状态码 500，触发重试，剩余重试次数: 1
2024-06-10 12:00:03 - INFO - ⏳ 0.6 秒后开始重试...
2024-06-10 12:00:04 - INFO - 正在向 https://httpbin.org/status/500 发送请求，带重试逻辑...
2024-06-10 12:00:05 - ERROR - ❌ 请求失败: 收到状态码 500，触发重试，剩余重试次数: 0
2024-06-10 12:00:05 - ERROR - 🚨 所有重试均失败。
```

## 结论

为了避免各种失败请求，我们提供了 [网络解锁器 API](https://www.bright.cn/products/web-unlocker) 和 [抓取浏览器](https://www.bright.cn/products/scraping-browser) 等产品。它们自动处理反爬虫措施、验证码挑战和 IP 封禁问题，即使应对高难度网站，也能保证抓取过程流畅高效。

立即注册并开启你的免费试用体验吧！
