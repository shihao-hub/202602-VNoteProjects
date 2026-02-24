### ✅ 方案 2：使用 `loguru` 的 `InterceptHandler`（适合混合场景）

如果你的项目中有些地方用 `logging`，有些想用 `loguru`，可以用官方提供的拦截器：

```python
import logging
from loguru import logger

class InterceptHandler(logging.Handler):
    def emit(self, record):
        # Get corresponding Loguru level if it exists
        try:
            level = logger.level(record.levelname).name
        except ValueError:
            level = record.levelno

        # Find caller from where originated the logged message
        frame, depth = logging.currentframe(), 2
        while frame.f_code.co_filename == logging.__file__:
            frame = frame.f_back
            depth += 1

        logger.opt(depth=depth, exception=record.exc_info).log(level, record.getMessage())

# 将所有 logging 调用重定向到 loguru
logging.basicConfig(handlers=[InterceptHandler()], level=0)
```

这样，即使你继续用 `logging.error(..., exc_info=True)`，底层也会由 `loguru` 输出！

> ✅ 优点：无需修改任何 `logging` 调用，日志统一由 `loguru` 管理。<br>🔗 官方文档：[https://loguru.readthedocs.io/en/stable/overview.html#entirely-compatible-with-standard-logging](https://loguru.readthedocs.io/en/stable/overview.html#entirely-compatible-with-standard-logging)