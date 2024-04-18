
# 前言
文档分割是一项具有挑战性的任务，它是任何知识库问答系统的基础。高质量的文档分割结果对于显著提升问答效果至关重要，但是目前大多数开源库的处理能力有限。
这些开源的库或者方法缺点大致可以罗列如下：
 - 只能处理文本，无法提取表格中的内容
 - 缺乏有效的分割策略，要么是一整个文档全部提取，要么是词粒度的获取

对于第一点，一般是把表格中的内容识别成文本，这样喂给大模型的时候就会出现一连串数字或者字母，这无疑会增大模型的理解难度；对于第二点，则是需要按照指定的长度对文档进行切分，或者把词按照一定的规则拼接到一块，这同样会损失到文本自身的上下文信息。

而本文接下来介绍的**Open-parse**这个库可以直接从文本中提取出多个节点，每个节点就是一个chunk，已经分好了，因此无需再按照长度进行split，这样同时也比单独提取一个词再进行合并又简化了不少操作；同时还支持同时提取表格和文字，无需分开提取。
# 快速开始

## 安装

```shell
pip install openparse
```
使用`pip`进行安装，同时这个库依赖`Pymupdf`、`pdfminer`等其他库，也会同时安装。
## 识别文字
```python
pdf = "c:\\人口.pdf"
parser = openparse.DocumentParser()
parsed_basic_doc = parser.parse(pdf)
for node in parsed_basic_doc.nodes:
    node
    print('\n--------------------\n')
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/direct/6811895f45434353a895e540c9deaa3b.jpeg#pic_center)
可以看到每一页的pdf被分成多个chunk，且还能保留原始文本中的**加粗**、*斜体*等信息。

```bash
print(parsed_basic_doc.nodes[0])

```

> elements=(TextElement(text='**Aging Research**老龄化研究**, 2022, 9(3), 26-34**\nPublished Online September 2022 in Hans. http://www.hanspub.org/journal/ar \nhttps://doi.org/10.12677/ar.2022.93004 ', lines=(LineElement(bbox=(56.64, 739.57, 232.44, 750.01), spans=(TextSpan(text='Aging Research ', is_bold=True, is_italic=False, size=9.0), TextSpan(text='老龄化研究', is_bold=False, is_italic=False, size=9.0), TextSpan(text=', 2022, 9(3), 26-34 ', is_bold=True, is_italic=False, size=9.0)), style=None, text='**Aging Research**老龄化研究**, 2022, 9(3), 26-34**'), LineElement(bbox=(56.65, 728.28, 348.95, 737.28), spans=(TextSpan(text='Published Online September 2022 in Hans. http://www.hanspub.org/journal/ar ', is_bold=False, is_italic=False, size=9.0),), style=None, text='Published Online September 2022 in Hans. http://www.hanspub.org/journal/ar '), LineElement(bbox=(56.64, 717.36, 225.23, 726.36), spans=(TextSpan(text='https://doi.org/10.12677/ar.2022.93004 ', is_bold=False, is_italic=False, size=9.0),), style=None, text='https://doi.org/10.12677/ar.2022.93004 ')), bbox=Bbox(page=0, page_height=807.96, page_width=595.32, x0=56.64, y0=717.36, x1=348.95, y1=750.01), variant=<NodeVariant.TEXT: 'text'>, embed_text='**Aging Research**老龄化研究**, 2022, 9(3), 26-34**\nPublished Online September 2022 in Hans. http://www.hanspub.org/journal/ar \nhttps://doi.org/10.12677/ar.2022.93004 '),) variant={'text'} tokens=66 bbox=[Bbox(page=0, page_height=807.96, page_width=595.32, x0=56.64, y0=717.36, x1=348.95, y1=750.01)] text='**Aging Research**老龄化研究**, 2022, 9(3), 26-34**\nPublished Online September 2022 in Hans. http://www.hanspub.org/journal/ar \nhttps://doi.org/10.12677/ar.2022.93004 '


通过打印出node，可以看出这种结构包含了原始文本中的元信息，包含文本的坐标、大小、是否加粗、是否斜体等。

## 识别表格内容

 - Pymupdf
 - Unitable
 - Table Transformer
 
 
`openparse`提供了三个方法来识别和提取表格中的内容，方法1是直接使用`Pymupdf`这个库的表格识别模块，因此准确率最差，但对硬件要求不高；其他的2个都是100mb左右的模型，如果用cpu来推理会比较耗时。

```python
# defining the parser (table_args is a dict)
parser = openparse.DocumentParser(
    table_args={
        "parsing_algorithm": "table-transformers", # 或者其他两个方法
        "table_output_format": "html" # 以html格式返回表格内容，也可以选择md
    }
)

```
与前面直接识别文本类似，只需要加入`table_args`参数即可。
![在这里插入图片描述](https://img-blog.csdnimg.cn/direct/95d01d7f991b4786a6c05488bdab7ca7.png#pic_center)
可以看到表格中的内容被很好的还原了

> 使用表格提取除了返回表格内容外，还会把正常的文本返回，这与`Pymupdf`等库只能选择返回文本还是只返回已有的表格不同。因此在不确定文本中含有什么内容时用这个方法会更加保险一点，对硬件的计算要求也不高。

### 语义相似

```python
from openparse import processing, DocumentParser

semantic_pipeline = processing.SemanticIngestionPipeline(
    openai_api_key=OPEN_AI_KEY,
    model="text-embedding-3-large",
    min_tokens=64,
    max_tokens=1024,
)

parser = DocumentParser(
    processing_pipeline=semantic_pipeline,
)
```
`openparse`还支持端到端的方式对node数据进行向量化并聚类，只需要指定`processing_pipeline`为相应的embedding模型即可。但是目前仅支持OpenAI的模型，需要OPEN_AI_KEY才可以使用。虽然后续会更新其他模型，但目前想用的话需要自己修改这段代码的实现。

```python
combine_parser = DocumentParser(
    processing_pipeline=semantic_pipeline,
    table_args={
        "parsing_algorithm": "table-transformers",
        "table_output_format": "html"
    }
    
)
```
同时，还能把语义相似和表格内容提取组合到一起使用，实现对表格内容提取的同时还能融合相似的片段。

# 总结
`openparse`这个库算是目前开源社区中比较优秀的文档分割处理库了，功能虽然全面，还是还有不少可以优化的地方，后续也会支持其他向量化模型，并且可以跟`Llamaindex`、`Langchain`等框架无缝衔接，应该值得持续关注。
