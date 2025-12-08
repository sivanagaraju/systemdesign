# Class Design Fundamentals

> **Interview Frequency:** ⭐⭐⭐⭐ (Type B LLD)

## The Core Question

*"Design a Logger interface with FileLogger and ConsoleLogger implementations"*

This tests your understanding of **OOP principles** applied to data engineering.

---

## 🔤 SOLID Principles Quick Reference

| Principle | Description | Data Engineering Example |
|-----------|-------------|--------------------------|
| **S**ingle Responsibility | Class does one thing | `FileReader` only reads files, doesn't transform |
| **O**pen/Closed | Open for extension, closed for modification | Add new source via factory, don't modify factory |
| **L**iskov Substitution | Subclasses work where parent expected | `S3Reader` works wherever `DataReader` is used |
| **I**nterface Segregation | Small, focused interfaces | `Readable` and `Writable` instead of `ReadWritable` |
| **D**ependency Inversion | Depend on abstractions | Pipeline depends on `DataReader`, not `S3Reader` |

---

## 🏗️ Logger Design Example

### Step 1: Define the Interface

```python
from abc import ABC, abstractmethod
from datetime import datetime
from enum import Enum

class LogLevel(Enum):
    DEBUG = 10
    INFO = 20
    WARNING = 30
    ERROR = 40
    CRITICAL = 50


class Logger(ABC):
    """Abstract base class for all loggers."""
    
    def __init__(self, name: str, level: LogLevel = LogLevel.INFO):
        self.name = name
        self.level = level
    
    @abstractmethod
    def write(self, message: str) -> None:
        """Write message to output destination."""
        pass
    
    def _should_log(self, level: LogLevel) -> bool:
        """Check if message level meets threshold."""
        return level.value >= self.level.value
    
    def _format_message(self, level: LogLevel, message: str) -> str:
        """Standard log message format."""
        timestamp = datetime.now().isoformat()
        return f"{timestamp} | {level.name:8} | {self.name} | {message}"
    
    def log(self, level: LogLevel, message: str) -> None:
        """Log a message if level meets threshold."""
        if self._should_log(level):
            formatted = self._format_message(level, message)
            self.write(formatted)
    
    # Convenience methods
    def debug(self, message: str) -> None:
        self.log(LogLevel.DEBUG, message)
    
    def info(self, message: str) -> None:
        self.log(LogLevel.INFO, message)
    
    def warning(self, message: str) -> None:
        self.log(LogLevel.WARNING, message)
    
    def error(self, message: str) -> None:
        self.log(LogLevel.ERROR, message)
    
    def critical(self, message: str) -> None:
        self.log(LogLevel.CRITICAL, message)
```

### Step 2: Implement Concrete Classes

```python
class ConsoleLogger(Logger):
    """Logger that writes to stdout."""
    
    def write(self, message: str) -> None:
        print(message)


class FileLogger(Logger):
    """Logger that writes to a file."""
    
    def __init__(self, name: str, file_path: str, level: LogLevel = LogLevel.INFO):
        super().__init__(name, level)
        self.file_path = file_path
    
    def write(self, message: str) -> None:
        with open(self.file_path, 'a') as f:
            f.write(message + '\n')


class AzureLogAnalyticsLogger(Logger):
    """Logger that sends to Azure Log Analytics."""
    
    def __init__(self, name: str, workspace_id: str, shared_key: str, 
                 level: LogLevel = LogLevel.INFO):
        super().__init__(name, level)
        self.workspace_id = workspace_id
        self.shared_key = shared_key
    
    def write(self, message: str) -> None:
        # Build and send HTTP request to Log Analytics API
        # (Simplified - real implementation would use requests library)
        self._send_to_azure(message)
    
    def _send_to_azure(self, message: str) -> None:
        # Implementation details...
        pass
```

### Step 3: Composite Logger (Multiple Destinations)

```python
class CompositeLogger(Logger):
    """Logger that writes to multiple destinations."""
    
    def __init__(self, name: str, loggers: list[Logger], level: LogLevel = LogLevel.INFO):
        super().__init__(name, level)
        self.loggers = loggers
    
    def write(self, message: str) -> None:
        for logger in self.loggers:
            logger.write(message)


# Usage
logger = CompositeLogger(
    name="ETL_Pipeline",
    loggers=[
        ConsoleLogger("ETL_Console"),
        FileLogger("ETL_File", "/logs/etl.log"),
        AzureLogAnalyticsLogger("ETL_Azure", "workspace-id", "key")
    ]
)

logger.info("Pipeline started")  # Goes to all 3 destinations!
```

---

## 📊 Data Reader Design

Another common LLD problem: design data readers for multiple sources.

```python
from abc import ABC, abstractmethod
from typing import Iterator, Dict, Any

class DataReader(ABC):
    """Abstract base for all data readers."""
    
    @abstractmethod
    def read(self) -> Iterator[Dict[str, Any]]:
        """Yield records one at a time."""
        pass
    
    @abstractmethod
    def close(self) -> None:
        """Clean up resources."""
        pass
    
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()
        return False


class CSVReader(DataReader):
    """Read from CSV files."""
    
    def __init__(self, file_path: str, delimiter: str = ','):
        self.file_path = file_path
        self.delimiter = delimiter
        self._file = None
    
    def read(self) -> Iterator[Dict[str, Any]]:
        import csv
        self._file = open(self.file_path, 'r')
        reader = csv.DictReader(self._file, delimiter=self.delimiter)
        for row in reader:
            yield row
    
    def close(self) -> None:
        if self._file:
            self._file.close()


class SQLReader(DataReader):
    """Read from SQL database."""
    
    def __init__(self, connection_string: str, query: str, batch_size: int = 1000):
        self.connection_string = connection_string
        self.query = query
        self.batch_size = batch_size
        self._connection = None
    
    def read(self) -> Iterator[Dict[str, Any]]:
        import pyodbc
        self._connection = pyodbc.connect(self.connection_string)
        cursor = self._connection.cursor()
        cursor.execute(self.query)
        
        columns = [column[0] for column in cursor.description]
        
        while True:
            rows = cursor.fetchmany(self.batch_size)
            if not rows:
                break
            for row in rows:
                yield dict(zip(columns, row))
    
    def close(self) -> None:
        if self._connection:
            self._connection.close()


class S3Reader(DataReader):
    """Read from S3 (Parquet files)."""
    
    def __init__(self, bucket: str, key: str):
        self.bucket = bucket
        self.key = key
    
    def read(self) -> Iterator[Dict[str, Any]]:
        import boto3
        import pyarrow.parquet as pq
        import io
        
        s3 = boto3.client('s3')
        response = s3.get_object(Bucket=self.bucket, Key=self.key)
        
        table = pq.read_table(io.BytesIO(response['Body'].read()))
        
        for row in table.to_pylist():
            yield row
    
    def close(self) -> None:
        pass  # No persistent connection
```

---

## 🎯 Interview Answer Framework

When designing a class hierarchy:

> **1. Identify the abstraction**
> *"First, I'll define an abstract base class or interface that captures the common behavior."*

> **2. Apply SOLID principles**
> *"Each implementation does one thing (SRP), and I can add new implementations without changing existing code (OCP)."*

> **3. Use composition over inheritance**
> *"For complex behavior, I prefer composition (CompositeLogger) over deep inheritance."*

> **4. Consider lifecycle**
> *"I'll implement context manager protocol (`__enter__`/`__exit__`) for proper resource cleanup."*

---

## 📖 Next Topic

Continue to [Singleton Pattern](./02-singleton-pattern.md) for config and connection management.
