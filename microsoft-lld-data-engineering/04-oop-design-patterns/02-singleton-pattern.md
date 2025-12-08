# Singleton Pattern

> **Interview Frequency:** ⭐⭐⭐⭐ (Common Pattern)

## The Core Question

*"Ensure only one instance of the config loader exists across the application"*

---

## 🤔 When to Use Singleton

| Use Case | Why Singleton |
|----------|---------------|
| **Config Loader** | Single source of truth for settings |
| **Database Connection Pool** | Expensive to create, share across threads |
| **Logger** | Consistent logging across modules |
| **Cache** | Share cached data globally |

---

## 🐍 Python Implementations

### Method 1: Module-Level Singleton (Simplest)

Python modules are singletons by nature!

```python
# config.py
class _Config:
    def __init__(self):
        self.settings = self._load_settings()
    
    def _load_settings(self):
        import json
        with open('config.json', 'r') as f:
            return json.load(f)
    
    def get(self, key: str, default=None):
        return self.settings.get(key, default)

# Module-level singleton instance
config = _Config()

# Usage (anywhere in code):
from config import config
db_host = config.get('database.host')
```

### Method 2: Decorator Singleton

```python
def singleton(cls):
    """Decorator that makes a class a singleton."""
    instances = {}
    
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    
    return get_instance


@singleton
class ConfigLoader:
    def __init__(self):
        print("Loading config...")
        self.settings = self._load()
    
    def _load(self):
        return {"env": "prod", "debug": False}


# Usage
config1 = ConfigLoader()  # Prints "Loading config..."
config2 = ConfigLoader()  # No print - same instance!
assert config1 is config2  # True
```

### Method 3: Metaclass Singleton (Thread-Safe)

```python
import threading

class SingletonMeta(type):
    """Thread-safe singleton metaclass."""
    
    _instances = {}
    _lock = threading.Lock()
    
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            with cls._lock:
                # Double-check locking
                if cls not in cls._instances:
                    instance = super().__call__(*args, **kwargs)
                    cls._instances[cls] = instance
        return cls._instances[cls]


class DatabasePool(metaclass=SingletonMeta):
    """Thread-safe singleton database connection pool."""
    
    def __init__(self, connection_string: str, pool_size: int = 5):
        self.connection_string = connection_string
        self.pool_size = pool_size
        self._connections = []
        self._initialize_pool()
    
    def _initialize_pool(self):
        print(f"Creating pool with {self.pool_size} connections")
        for _ in range(self.pool_size):
            self._connections.append(self._create_connection())
    
    def _create_connection(self):
        # Simulated connection
        return {"connection": self.connection_string}
    
    def get_connection(self):
        if self._connections:
            return self._connections.pop()
        raise Exception("No available connections")
    
    def return_connection(self, conn):
        self._connections.append(conn)
```

---

## 🏗️ Config Loader Example

```python
import json
import os
from pathlib import Path
from typing import Any, Dict, Optional
import threading


class ConfigLoader(metaclass=SingletonMeta):
    """
    Singleton configuration loader with environment support.
    """
    
    def __init__(self, config_path: str = "config.json"):
        self._config_path = config_path
        self._config: Dict[str, Any] = {}
        self._lock = threading.Lock()
        self._load()
    
    def _load(self) -> None:
        """Load configuration from file."""
        path = Path(self._config_path)
        if path.exists():
            with open(path, 'r') as f:
                self._config = json.load(f)
        
        # Override with environment variables
        self._apply_env_overrides()
    
    def _apply_env_overrides(self) -> None:
        """Override config values with environment variables."""
        # Convention: CONFIG_SECTION_KEY maps to {"section": {"key": value}}
        for key, value in os.environ.items():
            if key.startswith("CONFIG_"):
                parts = key[7:].lower().split("_")
                self._set_nested(parts, value)
    
    def _set_nested(self, keys: list, value: str) -> None:
        """Set a nested config value."""
        current = self._config
        for key in keys[:-1]:
            current = current.setdefault(key, {})
        current[keys[-1]] = value
    
    def get(self, key: str, default: Any = None) -> Any:
        """
        Get config value using dot notation.
        
        Example: config.get("database.host") → config["database"]["host"]
        """
        keys = key.split(".")
        value = self._config
        
        for k in keys:
            if isinstance(value, dict):
                value = value.get(k)
            else:
                return default
            
            if value is None:
                return default
        
        return value
    
    def reload(self) -> None:
        """Thread-safe config reload."""
        with self._lock:
            self._load()


# Usage
config = ConfigLoader()
db_host = config.get("database.host", "localhost")
db_port = config.get("database.port", 5432)
```

---

## ⚠️ Singleton Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| **Global mutable state** | Hard to test | Use dependency injection |
| **Hidden dependencies** | Code depends on singleton implicitly | Pass as parameter |
| **Thread unsafety** | Race conditions | Use locks or metaclass |

### Better Alternative: Dependency Injection

```python
class ETLPipeline:
    # BAD: Hidden singleton dependency
    def __init__(self):
        self.config = ConfigLoader()  # Tight coupling
    
    # GOOD: Explicit dependency injection
    def __init__(self, config: ConfigLoader):
        self.config = config  # Can inject mock for testing


# Testing becomes easy
def test_pipeline():
    mock_config = MockConfig({"batch_size": 100})
    pipeline = ETLPipeline(config=mock_config)
```

---

## 📖 Next Topic

Continue to [Factory Pattern](./03-factory-pattern.md) for creating data source objects.
