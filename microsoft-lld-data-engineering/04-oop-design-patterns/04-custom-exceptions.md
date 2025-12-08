# Custom Exceptions

> **Interview Frequency:** ⭐⭐⭐ (Good Practice Topic)

## The Core Question

*"Design exception classes for a data pipeline that distinguishes between retryable and fatal errors"*

---

## 🏗️ Exception Hierarchy

```python
class DataPipelineError(Exception):
    """Base exception for all pipeline errors."""
    
    def __init__(self, message: str, details: dict = None):
        self.message = message
        self.details = details or {}
        super().__init__(self.message)


# Retryable Errors (transient issues)
class RetryableError(DataPipelineError):
    """Errors that may succeed on retry."""
    
    def __init__(self, message: str, retry_after_seconds: int = 60, **kwargs):
        super().__init__(message, **kwargs)
        self.retry_after_seconds = retry_after_seconds


class ConnectionError(RetryableError):
    """Failed to connect to data source."""
    pass


class RateLimitError(RetryableError):
    """API rate limit exceeded."""
    pass


class TimeoutError(RetryableError):
    """Operation timed out."""
    pass


# Non-Retryable Errors (fatal issues)
class FatalError(DataPipelineError):
    """Errors that should not be retried."""
    pass


class ValidationError(FatalError):
    """Data validation failed."""
    
    def __init__(self, message: str, failed_records: int = 0, **kwargs):
        super().__init__(message, **kwargs)
        self.failed_records = failed_records


class SchemaError(FatalError):
    """Schema mismatch detected."""
    
    def __init__(self, message: str, expected_schema: dict = None, 
                 actual_schema: dict = None, **kwargs):
        super().__init__(message, **kwargs)
        self.expected_schema = expected_schema
        self.actual_schema = actual_schema


class ConfigurationError(FatalError):
    """Invalid configuration."""
    pass
```

---

## 🔄 Using Exceptions with Retry Logic

```python
import time
from functools import wraps


def retry_on_transient(max_retries: int = 3, backoff_factor: float = 2.0):
    """Decorator that retries on RetryableError."""
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_error = None
            
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                
                except RetryableError as e:
                    last_error = e
                    wait_time = e.retry_after_seconds * (backoff_factor ** attempt)
                    print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s: {e}")
                    time.sleep(wait_time)
                
                except FatalError:
                    raise  # Don't retry fatal errors
            
            raise last_error
        
        return wrapper
    return decorator


# Usage
@retry_on_transient(max_retries=3)
def fetch_data_from_api(endpoint: str):
    response = requests.get(endpoint)
    
    if response.status_code == 429:
        raise RateLimitError("Rate limited", retry_after_seconds=60)
    
    if response.status_code >= 500:
        raise ConnectionError("Server error", retry_after_seconds=30)
    
    if response.status_code == 400:
        raise ValidationError("Bad request - invalid parameters")  # Fatal, no retry
    
    return response.json()
```

---

## 🎯 Interview Answer

> *"I'd create a hierarchy distinguishing retryable vs fatal errors:*
> - *`RetryableError`: Transient issues (timeout, rate limit)*
> - *`FatalError`: Won't succeed on retry (validation, schema)*
>
> *Benefits:*
> - *Retry logic can catch `RetryableError` only*
> - *Fatal errors fail fast, don't waste compute*
> - *Rich context in exception attributes (retry_after, failed_records)"*

---

## 📖 Next Topic

Continue to [Rate Limiter Design](./05-rate-limiter-design.md) for API ingestion throttling.
