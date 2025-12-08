# Factory Pattern

> **Interview Frequency:** ⭐⭐⭐⭐ (Common Pattern)

## The Core Question

*"Design a factory that creates appropriate data readers based on source type"*

---

## 🤔 When to Use Factory

| Scenario | Why Factory |
|----------|-------------|
| **Multiple data sources** | Create reader based on source type |
| **Config-driven pipeline** | Instantiate components from config |
| **Hide complexity** | Client code doesn't know implementation |
| **Extensibility** | Add new types without modifying factory |

---

## 🏗️ Simple Factory

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, Iterator


class DataReader(ABC):
    @abstractmethod
    def read(self) -> Iterator[Dict[str, Any]]:
        pass


class CSVReader(DataReader):
    def __init__(self, file_path: str):
        self.file_path = file_path
    
    def read(self) -> Iterator[Dict[str, Any]]:
        import csv
        with open(self.file_path, 'r') as f:
            reader = csv.DictReader(f)
            yield from reader


class JSONReader(DataReader):
    def __init__(self, file_path: str):
        self.file_path = file_path
    
    def read(self) -> Iterator[Dict[str, Any]]:
        import json
        with open(self.file_path, 'r') as f:
            data = json.load(f)
            if isinstance(data, list):
                yield from data
            else:
                yield data


class ParquetReader(DataReader):
    def __init__(self, file_path: str):
        self.file_path = file_path
    
    def read(self) -> Iterator[Dict[str, Any]]:
        import pyarrow.parquet as pq
        table = pq.read_table(self.file_path)
        yield from table.to_pylist()


class DataReaderFactory:
    """Factory for creating data readers."""
    
    @staticmethod
    def create(source_type: str, config: Dict[str, Any]) -> DataReader:
        """
        Create a reader based on source type.
        
        Args:
            source_type: 'csv', 'json', 'parquet', 'sql', etc.
            config: Configuration for the reader
        
        Returns:
            Appropriate DataReader implementation
        """
        readers = {
            'csv': CSVReader,
            'json': JSONReader,
            'parquet': ParquetReader,
        }
        
        reader_class = readers.get(source_type.lower())
        if not reader_class:
            raise ValueError(f"Unsupported source type: {source_type}")
        
        return reader_class(**config)


# Usage
reader = DataReaderFactory.create('csv', {'file_path': '/data/orders.csv'})
for record in reader.read():
    print(record)
```

---

## 📝 Registry-Based Factory

More extensible - allows runtime registration of new types.

```python
from typing import Callable, Type

class DataReaderRegistry:
    """Registry-based factory with dynamic registration."""
    
    _registry: Dict[str, Type[DataReader]] = {}
    
    @classmethod
    def register(cls, source_type: str) -> Callable:
        """Decorator to register a reader class."""
        def decorator(reader_class: Type[DataReader]):
            cls._registry[source_type.lower()] = reader_class
            return reader_class
        return decorator
    
    @classmethod
    def create(cls, source_type: str, **kwargs) -> DataReader:
        """Create a reader from registry."""
        reader_class = cls._registry.get(source_type.lower())
        if not reader_class:
            raise ValueError(f"Unknown source type: {source_type}")
        return reader_class(**kwargs)
    
    @classmethod
    def available_types(cls) -> list:
        """List all registered source types."""
        return list(cls._registry.keys())


# Register readers using decorator
@DataReaderRegistry.register('csv')
class CSVReader(DataReader):
    def __init__(self, file_path: str, delimiter: str = ','):
        self.file_path = file_path
        self.delimiter = delimiter
    
    def read(self) -> Iterator[Dict[str, Any]]:
        import csv
        with open(self.file_path, 'r') as f:
            reader = csv.DictReader(f, delimiter=self.delimiter)
            yield from reader


@DataReaderRegistry.register('sql')
class SQLReader(DataReader):
    def __init__(self, connection_string: str, query: str):
        self.connection_string = connection_string
        self.query = query
    
    def read(self) -> Iterator[Dict[str, Any]]:
        # SQL reading logic
        pass


# Usage
print(DataReaderRegistry.available_types())  # ['csv', 'sql']
reader = DataReaderRegistry.create('csv', file_path='/data/orders.csv')
```

---

## 🔧 Config-Driven Factory

Create entire pipelines from configuration.

```python
from typing import List
import yaml


class PipelineFactory:
    """Create pipeline components from YAML config."""
    
    def create_reader(self, config: Dict) -> DataReader:
        return DataReaderRegistry.create(
            source_type=config['type'],
            **config.get('params', {})
        )
    
    def create_writers(self, configs: List[Dict]) -> List['DataWriter']:
        return [self.create_writer(c) for c in configs]
    
    def create_writer(self, config: Dict) -> 'DataWriter':
        return DataWriterRegistry.create(
            target_type=config['type'],
            **config.get('params', {})
        )
    
    @classmethod
    def from_yaml(cls, yaml_path: str) -> 'Pipeline':
        """Build complete pipeline from YAML configuration."""
        with open(yaml_path, 'r') as f:
            config = yaml.safe_load(f)
        
        factory = cls()
        
        reader = factory.create_reader(config['source'])
        writers = factory.create_writers(config.get('targets', []))
        
        return Pipeline(reader=reader, writers=writers)


# config.yaml:
# source:
#   type: sql
#   params:
#     connection_string: "..."
#     query: "SELECT * FROM orders"
# targets:
#   - type: parquet
#     params:
#       path: /output/orders.parquet
#   - type: delta
#     params:
#       path: /delta/orders

pipeline = PipelineFactory.from_yaml('config.yaml')
pipeline.run()
```

---

## 🎯 Interview Answer

> *"For data source abstraction, I'd use a Factory pattern:*
>
> 1. *Define a `DataReader` interface with common methods*
> 2. *Implement concrete readers (CSV, SQL, S3, etc.)*
> 3. *Factory creates appropriate reader based on config*
> 4. *Registry pattern allows adding new sources at runtime*
>
> *Benefits:*
> - *Client code doesn't know which reader it's using*
> - *Easy to add new source types*
> - *Configuration-driven for flexibility*"

---

## 📖 Next Topic

Continue to [Custom Exceptions](./04-custom-exceptions.md) for error handling patterns.
