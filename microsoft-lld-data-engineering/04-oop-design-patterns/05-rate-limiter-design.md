# Rate Limiter Design

> **Interview Frequency:** ⭐⭐⭐⭐ (Classic LLD Question)

## The Core Question

*"Design a Rate Limiter for an API ingestion script that respects 100 requests/minute"*

---

## 📊 Rate Limiting Algorithms

| Algorithm | Description | Use Case |
|-----------|-------------|----------|
| **Token Bucket** | Tokens refill at fixed rate | Bursty traffic allowed |
| **Sliding Window** | Count requests in rolling window | Smooth rate limiting |
| **Fixed Window** | Count per fixed time interval | Simple, edge case issues |

---

## 🪣 Token Bucket Implementation

```python
import threading
import time
from typing import Optional


class TokenBucketRateLimiter:
    """
    Token Bucket rate limiter.
    
    Tokens are added at a fixed rate. Requests consume tokens.
    Allows bursts up to bucket capacity.
    """
    
    def __init__(self, rate: float, capacity: int):
        """
        Args:
            rate: Tokens added per second
            capacity: Maximum tokens (burst size)
        """
        self.rate = rate
        self.capacity = capacity
        self.tokens = capacity
        self.last_refill = time.time()
        self._lock = threading.Lock()
    
    def _refill(self) -> None:
        """Add tokens based on elapsed time."""
        now = time.time()
        elapsed = now - self.last_refill
        tokens_to_add = elapsed * self.rate
        
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now
    
    def acquire(self, tokens: int = 1, blocking: bool = True) -> bool:
        """
        Acquire tokens for a request.
        
        Args:
            tokens: Number of tokens needed
            blocking: If True, wait for tokens; if False, return immediately
        
        Returns:
            True if tokens acquired, False if not (non-blocking mode)
        """
        while True:
            with self._lock:
                self._refill()
                
                if self.tokens >= tokens:
                    self.tokens -= tokens
                    return True
                
                if not blocking:
                    return False
                
                # Calculate wait time
                tokens_needed = tokens - self.tokens
                wait_time = tokens_needed / self.rate
            
            time.sleep(wait_time)
    
    def wait_time(self) -> float:
        """Return seconds until next token available."""
        with self._lock:
            self._refill()
            if self.tokens >= 1:
                return 0
            return (1 - self.tokens) / self.rate


# Usage: 100 requests per minute = 1.67 per second
limiter = TokenBucketRateLimiter(rate=100/60, capacity=10)

def fetch_with_rate_limit(url: str):
    limiter.acquire()  # Blocks until token available
    return requests.get(url)
```

---

## 📊 Sliding Window Implementation

```python
from collections import deque
import time
import threading


class SlidingWindowRateLimiter:
    """
    Sliding window rate limiter.
    
    Counts requests in a rolling time window.
    More accurate than fixed window.
    """
    
    def __init__(self, max_requests: int, window_seconds: int):
        """
        Args:
            max_requests: Maximum requests allowed in window
            window_seconds: Size of sliding window
        """
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests = deque()  # Timestamps of requests
        self._lock = threading.Lock()
    
    def _cleanup_old_requests(self) -> None:
        """Remove requests outside the window."""
        now = time.time()
        cutoff = now - self.window_seconds
        
        while self.requests and self.requests[0] < cutoff:
            self.requests.popleft()
    
    def acquire(self, blocking: bool = True) -> bool:
        """
        Try to acquire permission for a request.
        """
        while True:
            with self._lock:
                now = time.time()
                self._cleanup_old_requests()
                
                if len(self.requests) < self.max_requests:
                    self.requests.append(now)
                    return True
                
                if not blocking:
                    return False
                
                # Wait until oldest request expires
                wait_time = self.requests[0] + self.window_seconds - now
            
            if wait_time > 0:
                time.sleep(wait_time)
    
    def current_usage(self) -> tuple:
        """Return (current_count, max_requests)."""
        with self._lock:
            self._cleanup_old_requests()
            return len(self.requests), self.max_requests


# Usage
limiter = SlidingWindowRateLimiter(max_requests=100, window_seconds=60)
```

---

## 🔧 API Client with Rate Limiting

```python
import requests
from typing import Any, Dict


class RateLimitedAPIClient:
    """HTTP client with built-in rate limiting."""
    
    def __init__(self, base_url: str, requests_per_minute: int = 100):
        self.base_url = base_url
        self.limiter = TokenBucketRateLimiter(
            rate=requests_per_minute / 60,
            capacity=min(10, requests_per_minute // 10)
        )
        self.session = requests.Session()
    
    def get(self, endpoint: str, **kwargs) -> Dict[str, Any]:
        """Rate-limited GET request."""
        self.limiter.acquire()
        url = f"{self.base_url}/{endpoint}"
        response = self.session.get(url, **kwargs)
        response.raise_for_status()
        return response.json()
    
    def post(self, endpoint: str, data: Dict, **kwargs) -> Dict[str, Any]:
        """Rate-limited POST request."""
        self.limiter.acquire()
        url = f"{self.base_url}/{endpoint}"
        response = self.session.post(url, json=data, **kwargs)
        response.raise_for_status()
        return response.json()


# Usage
client = RateLimitedAPIClient("https://api.example.com", requests_per_minute=100)

# These calls automatically respect rate limit
for item_id in item_ids:
    data = client.get(f"items/{item_id}")
```

---

## 🎯 Interview Answer

> *"For API rate limiting, I'd implement Token Bucket:*
> - *Tokens refill at rate (100/60 = 1.67/sec)*
> - *Capacity allows bursts (e.g., 10 tokens)*
> - *`acquire()` blocks until token available*
> - *Thread-safe with locks*
>
> *Trade-offs:*
> - *Token bucket allows bursts, sliding window is smoother*
> - *Distributed systems need centralized counter (Redis)*"

---

## 📖 Next Section

Move to [05 - Azure Services LLD](../05-azure-services-lld/README.md) for Azure-specific deep dives.
