
最近项目中需要用到js库来渲染pdf文件，调研后发现无论是reach-pdf.js或者是svelte-pdf.js都是在pdf.js基础上做了些许精简，反而功能还不如原始的pdf.js来得全面。但是原始的库几乎没有像样的代码示例，而能搜索到的大多数代码不少都是十几年前的了，在这个过程中踩了不少坑，做个记录，希望对看到的人有所帮助。

使用npm安装pdfjs-dist库（也可以直接下载源码并引入）

```bash
npm install pdfjs-dist
```

导入库

```javascript
// 网上很多代码都是import xxx from 'pdfjs-dist';
// 而xxx一般都是过期或者不存在的，直接把所有导出为pdfjslib即可
import * as pdfjslib from 'pdfjs-dist';
// 注意需要设置这个参数
pdfjslib.GlobalWorkerOptions.workerSrc = 'node_modules/pdfjs-dist/build/pdf.worker.js';
```


单页渲染，多页渲染在下面代码基础上直接添加一个循环即可

```javascript
let src = 'xxx.pdf';
let pageNum = 1;
let scale_ratio = 1.5;

async function renderPage() {
    const pdf = await pdfjsLib.getDocument(src).promise;
    const page = await pdf.getPage(pageNum);
    const viewport = page.getViewport({ scale: scale_ratio });

    const canvas = document.createElement('canvas');
    const context = canvas.getContext('2d');

    canvas.height = viewport.height;
    canvas.width = viewport.width;

    const renderContext = {
      canvasContext: context,
      viewport: viewport
    };
    await page.render(renderContext);

  }
```

注意渲染完的pdf只有图片形式，使用开发者工具看网页的结构只有canvas组件，想要实现文字的选择和复制还需要在上面渲染一层文字层。

```javascript
// 需要引入样式文件，不然文字不会悬浮在cavas组件上
import 'pdfjs-dist/web/pdf_viewer.css';

async function renderFullPage(){
        const pdf = await pdfjsLib.getDocument(src).promise;

        const pdfContainer = document.createElement('div');
        pdfContainer.style.setProperty('--scale-factor', scale_ratio);

        for (let i=1; i<=pdf.numPages; i++){
            const pageNumber = i;
            const page = await pdf.getPage(pageNumber);

            const viewport = page.getViewport({scale: scale_ratio});

            const canvas = document.createElement('canvas');
            const context = canvas.getContext('2d');

            canvas.height = viewport.height ;
            canvas.width = viewport.width ;
            
            const renderContext = {
                canvasContext: context,
                viewport: viewport
            };
        
            await page.render(renderContext);

            // canvasWrapper 可加可不加
            const canvasWrapper = document.createElement('div');
            canvasWrapper.className = 'canvasWrapper';
            canvasWrapper.appendChild(canvas);

            const textContent = await page.getTextContent();
            const textLayerDiv = document.createElement('div');
            // 类名严格为：textLayer
            textLayerDiv.className = `textLayer`;

            pdfjsLib.renderTextLayer({
                textContentSource: textContent,
                container: textLayerDiv,
                viewport: viewport,
                textDivs: []
            });
            
            const pageDiv = document.createElement('div');
            pageDiv.className = 'page';
            // 需要设置 position: relative
            // 否则全部文字可能都挤在第一页
            pageDiv.style = "position: relative; margin-bottom:10px";

            pageDiv.appendChild(canvasWrapper);
            pageDiv.appendChild(textLayerDiv);
            
            pdfContainer.appendChild(pageDiv);
            
        }

    }
```

简单来说就是在渲染完canvas代码之后，再渲染出文字层。有几个注意点：
1. 需要在开头引入样式表，不然文字层会实际显示在页面中，不会悬浮不会透明；
2. 需要在外面的组件中设置参数--scale-factor，用于保证图片和文字的位置对应，否则调整了scale_ratio后图片尺寸改变，但是文字层的大小还是不变；
3. 文字层的类名需要严格设置为textLayer，从开头引入的样式表中可以看到；
4. 包含canvas和文字层的父组件需要设置style为position: relative，否则多页的文字都会渲染到第一页中。

