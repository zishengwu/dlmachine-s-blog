# 简介
Lancedb是一个用于人工智能的开源矢量数据库，旨在存储、管理、查询和检索大规模多模式数据的嵌入。Lancedb的核心是用Rust编写的，并构建在Lance之上，专为高性能 ML 工作负载和快速随机访问而设计。

# 快速开始
## 安装

```shell
pip install lancedb
```
目前0.6.8需要pyarrow-12.0.0及以上，亲测15.0会报错。

## 创建客户端
```python
import lancedb
import pandas as pd
import pyarrow as pa

uri = "data/sample-lancedb"
db = lancedb.connect(uri)   
# 异步客户端
#async_db = await lancedb.connect_async(uri)    
```
与Chroma不同，lancedb没有服务端-客户端模式。支持同步和异步客户端，看起来异步客户端更新较快，从官方文档来看没发现使用上的区别。
    
## 创建一张表
```python    
data = [
    {"vector": [3.1, 4.1], "item": "foo", "price": 10.0},
    {"vector": [5.9, 26.5], "item": "bar", "price": 20.0},
]

tbl = db.create_table("my_table", data=data) 
```
如果表名已经存在，则会报错。如果希望覆盖已经创建的同名表，可以添加mode='overwrite'参数。
```python
tbl = db.create_table("my_table", data=data, mode='overwrite') 
```    
如果不希望覆盖已经创建的同名表，而直接打开的话，可以添加exist_ok=True参数。
```python
tbl = db.create_table("my_table", data=data, exist_ok=True) 
```  
    
    
## 创建一张空表
```python
schema = pa.schema([pa.field("vector", pa.list_(pa.float32(), list_size=2))])
tbl = db.create_table("empty_table", schema=schema)
```  
类似SQL语法，先创建一张空表，插入数据可以放到后面进行。
 
## 添加数据
```python
# 直接添加数据
data = [
    {"vector": [1.3, 1.4], "item": "fizz", "price": 100.0},
    {"vector": [9.5, 56.2], "item": "buzz", "price": 200.0},
]
tbl.add(data)

# 添加df数据帧
df = pd.DataFrame(data)
tbl.add(data)
```
    
    
## 查找数据
```python
# Synchronous client
tbl.search([100, 100]).limit(2).to_pandas()

```
通过向量来查找相似的向量。默认情况下没有对向量创建索引，因此是全表暴力检索。官方推荐数据量超过50万以上才需要创建索引，否则全表暴力检索的延迟也在可以接受的范围之内。（明明就是没实现，还说的冠冕堂皇。。）
    
## 删除数据
```python
tbl.delete('item = "fizz"')
```
类似SQL语法中的WHERE声明，需要指定字段和对应的值。


## 修改数据
```python
table.update(where='item = "fizz"', values={"vector": [10, 10]})
```
类似SQL语法中的UPDATE声明，需要指定字段和对应的值。

    
    
## 删除表
```python
db.drop_table("my_table")
```

## 查看所有表
    
```python
print(db.table_names())
tbl = db.open_table("my_table")    
```
table_names可以返回该数据库中已经创建的所有表，使用open_table可以打开对应的表。

# 高级用法
## 数据类型    
### 多种数据类型
除了直接添加数据和添加df数据帧之外，lancedb还支持用pyarrow创建schema和添加数据。
```python
import pyarrow as pa
schema = pa.schema(
    [
        pa.field("vector", pa.list_(pa.float16(), 2)),
        pa.field("text", pa.string())
    ]
)   
```
lancedb直接float16数据类型，这就比chromadb有存储优势了。

### 自定义数据类型
```python 
from lancedb.pydantic import Vector, LanceModel

class Content(LanceModel):
    movie_id: int
    vector: Vector(128)
    genres: str
    title: str
    imdb_id: int

    @property
    def imdb_url(self) -> str:
        return f"https://www.imdb.com/title/tt{self.imdb_id}"   
```
LanceModel是pydantic.BaseModel的子类，主要就是实现了Vector数据类型的定义，避免手动创建schema中vector的定义，只需要指定维度即可。

### 复合数据类型
```python 
class Document(BaseModel):
    content: str
    source: str
    
class NestedSchema(LanceModel):
    id: str
    vector: Vector(1536)
    document: Document

tbl = db.create_table("nested_table", schema=NestedSchema, mode="overwrite")
```
## 索引    
### 创建IVF_PQ索引
```python 
tbl.create_index(num_partitions=256, num_sub_vectors=96)
```
lancedb支持创建倒排索引的乘积量化。num_partitions是索引中的分区数，默认值是行数的平方根。num_sub_vectors是子向量的数量，默认值是向量的维度除以16。
    
### 使用GPU创建
```python 
accelerator="cuda"
# accelerator="mps"
```
支持CUDA的GPU或者Apple的MPS加速
 
### 使用索引加速近似查找 
```python 
tbl.search(np.random.random((1536))) \
.limit(2) \
.nprobes(20) \
.refine_factor(10) \
.to_pandas()
```
nprobes是探针数量，默认为20，增加探针数量则会提高查找的精度并相应增加计算耗时。refine_factor是一个粗召的数量，用于读取额外元素并重新排列，以此来提高召回。
    
    
## 向量化模型

### 内置向量模型
```python 
import lancedb
from lancedb.pydantic import LanceModel, Vector
from lancedb.embeddings import get_registry

model = get_registry().get("sentence-transformers").create(name="BAAI/bge-small-en-v1.5", device="cpu")

class Words(LanceModel):
    text: str = model.SourceField() # 指定这个字段为需要模型进行向量化的字段
    vector: Vector(model.ndims()) = model.VectorField() # 指定这个字段为模型向量化的结果

table = db.create_table("words", schema=Words)
table.add(
    [
        {"text": "hello world"},
        {"text": "goodbye world"}
    ]
)

query = "greetings"
actual = table.search(query).limit(1).to_pydantic(Words)[0]
print(actual.text)
``` 

官方支持了多种sentence-transformers的向量化模型。用上述方法调用内置模型需要指定模型的SourceField和VectorField。
    
### 自定义向量模型
```python 
from lancedb.embeddings import register
from lancedb.util import attempt_import_or_raise

@register("sentence-transformers")
class SentenceTransformerEmbeddings(TextEmbeddingFunction):
    name: str = "all-MiniLM-L6-v2"
    # set more default instance vars like device, etc.

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self._ndims = None

    def generate_embeddings(self, texts):
        return self._embedding_model().encode(list(texts), ...).tolist()

    def ndims(self):
        if self._ndims is None:
            self._ndims = len(self.generate_embeddings("foo")[0])
        return self._ndims

    @cached(cache={}) 
    def _embedding_model(self):
        return sentence_transformers.SentenceTransformer(name)
``` 
  
```python 
from lancedb.pydantic import LanceModel, Vector

registry = EmbeddingFunctionRegistry.get_instance()
stransformer = registry.get("sentence-transformers").create()

class TextModelSchema(LanceModel):
    vector: Vector(stransformer.ndims) = stransformer.VectorField()
    text: str = stransformer.SourceField()

tbl = db.create_table("table", schema=TextModelSchema)

tbl.add(pd.DataFrame({"text": ["halo", "world"]}))
result = tbl.search("world").limit(5)
```
官方提供了模板用于自定义模型，但是我觉得直接调用模型进行向量化表示更直接吧，这样感觉有点追求格式化的统一了。

# 总结
与Chromadb对比，没有服务端模式，全部在客户端完成，虽然官方声称有云原生的版本，但感觉大部分场景下可能都不需要放在云上，感觉这一款产品会更加轻量化。
此外，创建表的时候没有默认的向量化模型，感觉对开发者可能更加灵活一些，相比之下Chromadb默认会从HuggingFace下载模型，对于内网环境不太友好。
    
