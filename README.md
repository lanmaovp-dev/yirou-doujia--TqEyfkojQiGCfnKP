
有个新需求，当点击【下载】按钮时，直接将当前 html页面下载为 PDF。通过 html2canvas \+ jsPDF 可实现PDF单页下载，甚至是多页下载，记录分享一下\~ 最后有源码，可自取🫡


[![](https://img2024.cnblogs.com/blog/2180164/202411/2180164-20241125175934462-1567954946.gif)](https://img2024.cnblogs.com/blog/2180164/202411/2180164-20241125175934462-1567954946.gif)


# html2canvas


html2canvas官网在这：[html2canvas \- Screenshots with JavaScript](https://github.com)


html2canvas 是一个 HTML 渲染器。该脚本允许你直接在用户浏览器截取页面或部分网页的“屏幕截屏”，通过读取 DOM 以及应用于元素的不同样式，将当前页面呈现为 `canvas` 图像


安装 html2canvas



```
npm install --save html2canvas
```

截取页面生成`canvas`，并将其插入页面中



```
html2canvas(document.body}).then(function(canvas) {
    document.body.appendChild(canvas);
});
```

注意：受限于浏览器的实现，HTML 的 canvas 元素也有高度限制⚠ 可参考 [canvas 最大画布尺寸 \- MDN](https://github.com)


* Chrome 和 Firefox 等现代浏览器，canvas 的最大尺寸通常限制在 32767 像素，这也是 WebGL 和 2D canvas 的共同限制。超过这个值会导致 canvas 生成失败，抛出错误，或者显示空白内容。
* 老版本的 IE 对 canvas 尺寸限制较为严格，一般在 8192 像素上下。现代版本的 Edge 则与 Chrome 的限制类似


# jsPDF


jsPDF文档可以看这：[GitHub \- parallax/jsPDF: Client\-side JavaScript PDF generation for everyone.](https://github.com)


安装 jspdf



```
npm install jspdf --save
```

API也很简单，下面是个生成文本和图片的PDF样例



```
// jsPDF 下载文本图片PDF
const downLoadPdf = () => {
  // 三个参数，第一个方向，第二个单位，第三个尺寸 'a4' = [595.28,841.89]
  const doc = new jsPDF('', 'pt', [500, 1000])

  // 字体大小
  doc.setFontSize(50)

  // 文本，左边距，上边距
  doc.text('Hello world!', 10, 50)

  // base64，图片类型，左边距，上边距，宽度，高度
  doc.addImage(base64, 'PNG', 10, 60, 400, 200)

  doc.save('a4.pdf')
}
```

注意： jsPDF 生成的 PDF 默认以 pt (point) 为单位，单页的最大高度通常限制在 14400 pt。超过这个高度可能导致生成的 PDF 无法正确渲染或浏览器崩溃⚠


# html2canvas \+ jsPDF 实现页面下载


## 单页下载（自适应纸）


PDF页面的宽高 采用 canvas的宽高


* 若`canvas.height >= canvas.width`，采用 `portrait` 纵向页面
* 若`canvas.width > canvas.height`，采用 `landscape` 横向页面


如果页面很长的话，单页下载就会生成一张长长的PDF。注意！超过限制就会显示空白页面， jsPDF 生成的 PDF单页最大高度为 14400pt⚠



> canvas也有最大高度限制 32767像素，如果页面过长的话，通过 html2canvas 生成 canvas会失败



```
const downLoadPdfAutoSingle = () => {
  html2canvas(document.body, {
    scale: window.devicePixelRatio * 2, // 使用设备的像素比 * 2
  }).then(canvas => {
    // 返回图片dataURL，参数：图片格式和清晰度(0-1)
    const pageData = canvas.toDataURL('image/jpeg', 1.0)

    const pageWidth = canvas.width
    const pageHeight =  canvas.height
    const orientation = canvas.height >= canvas.width ? 'portrait' : 'landscape'  // portrait 表示纵向，landscape表示横向
    const pdf = new jsPDF(orientation, 'pt', [pageWidth, pageHeight])
    
    // addImage后两个参数控制添加图片的尺寸，此处将页面高度按照a4纸宽高比列进行压缩
    pdf.addImage(
      pageData,
      'JPEG',
      0,
      0,
      pageWidth,
      pageHeight
    )
    pdf.save('下载一页PDF（自适应纸）.pdf')
  })
}
```

## 单页下载（A4纸）


已知：A4纸的宽度 和 canvas的宽度高度。可得 canvas在A4纸上占用的总高度（A4纸尺寸为宽 595\.28pt，高 841\.89pt）


如果页面很长的话，单页下载就会生成一张长长的PDF。注意！超过限制就会显示空白页面， jsPDF 生成的 PDF单页最大高度为 14400pt⚠



> canvas也有最大高度限制 32767像素，如果页面过长的话，通过 html2canvas 生成 canvas会失败



```
const downLoadPdfA4Single = () => {
  html2canvas(document.body).then(canvas => {
    // 返回图片dataURL，参数：图片格式和清晰度(0-1)
    const pageData = canvas.toDataURL('image/jpeg', 1.0)

    // 方向纵向，尺寸ponits，纸张格式 a4 即 [595.28, 841.89]
    const A4Width = 595.28
    const A4Height = 841.89 // A4纸宽
    const pageHeight = A4Height >= A4Width * canvas.height / canvas.width ? A4Height :  A4Width * canvas.height / canvas.width
    const pdf = new jsPDF('portrait', 'pt', [A4Width, pageHeight])
    
    // addImage后两个参数控制添加图片的尺寸，此处将页面高度按照a4纸宽高比列进行压缩
    pdf.addImage(
      pageData,
      'JPEG',
      0,
      0,
      A4Width,
      A4Width * canvas.height / canvas.width,
    )
    pdf.save('下载一页PDF（A4纸）.pdf')
  })
}
```

## 多页下载（自适应纸）


由于 jsPDF 单页最大高度的限制 又或是 需求层面，我们需要实现自动分页下载


我们设置一页PDF页面宽度为`canvas.width`，高度为`canvas.width * 1.3`


**分页思路：每个PDF页面都显示一张 canvas 图，只不过是计算偏移量，每个PDF页面显示的是 canvas 的不同位置**


问题来了，如何创建一个新的PDF页面呢？可以使用 jsPDF 的`pdf.addPage()`



```
const downLoadPdfAutoMultiple = () => {
  const ele = document.body
  html2canvas(ele, {
    scale: window.devicePixelRatio * 2, // 使用设备的像素比 * 2
  }).then(canvas => {
    let position = 0 //页面偏移
    const autoWidth = canvas.width // 一页纸宽度
    const autoHeight = canvas.width * 1.3 // 一页纸高度

    // 一页PDF可显示的canvas高度
    const pageHeight = (canvas.width * autoHeight) / autoWidth
    // 未分配到PDF的canvas高度
    let unallottedHeight = canvas.height

    // canvas对应的PDF宽高
    const imgWidth = canvas.width
    const imgHeight = canvas.height

    const pageData = canvas.toDataURL('image/jpeg', 1.0)
    const pdf = new jsPDF('', 'pt', [autoWidth, autoHeight])

    // 当canvas高度 未超过 一页PDF可显示的canvas高度，无需分页
    if (unallottedHeight <= pageHeight) {
      pdf.addImage(pageData, 'JPEG', 0, 0, imgWidth, imgHeight)
      pdf.save('下载多页PDF（自适应纸）.pdf')
      return
    }

    while (unallottedHeight > 0) {
      pdf.addImage(pageData, 'JPEG', 0, position, imgWidth, imgHeight)
      unallottedHeight -= pageHeight
      position -= autoHeight
      if (unallottedHeight > 0) {
        pdf.addPage()
      }
    }
    pdf.save('html2canvas+jsPDF下载PDF.pdf')
  })
}
```

## 多页下载（A4纸）


由于 jsPDF 单页最大高度的限制 又或是 需求层面，我们需要实现自动分页下载


我们设置一页PDF页面的宽高为A4纸尺寸，即宽 595\.28pt，高 841\.89pt


**分页思路：每个PDF页面都显示一张 canvas 图，只不过是计算偏移量，每个PDF页面显示的是 canvas 的不同位置**


问题来了，如何创建一个新的PDF页面呢？可以使用 jsPDF 的`pdf.addPage()`



```
const downLoadPdfA4Multiple = () => {
  const ele = document.body
  html2canvas(ele, {
    scale: 2, // 使用设备的像素比
  }).then(canvas => {
    let position = 0 //页面偏移
    const A4Width = 595.28 // A4纸宽度
    const A4Height = 841.89 // A4纸宽

    // 一页PDF可显示的canvas高度
    const pageHeight = (canvas.width * A4Height) / A4Width
    // 未分配到PDF的canvas高度
    let unallottedHeight = canvas.height

    // canvas对应的PDF宽高
    const imgWidth = A4Width
    const imgHeight = (A4Width * canvas.height) / canvas.width

    const pageData = canvas.toDataURL('image/jpeg', 1.0)
    const pdf = new jsPDF('', 'pt', [A4Width, A4Height])

    // 当canvas高度 未超过 一页PDF可显示的canvas高度，无需分页
    if (unallottedHeight <= pageHeight) {
      pdf.addImage(pageData, 'JPEG', 0, 0, imgWidth, imgHeight)
      pdf.save('下载多页PDF（A4纸）.pdf')
      return
    }

    while (unallottedHeight > 0) {
      pdf.addImage(pageData, 'JPEG', 0, position, imgWidth, imgHeight)
      unallottedHeight -= pageHeight
      position -= A4Height
      if (unallottedHeight > 0) {
        pdf.addPage()
      }
    }
    pdf.save('下载多页PDF（A4纸）.pdf')
  })
}
```

# 源码


[GitHub \- burc\-li/vue\-pdf: HTML 转 PDF下载（html2canvas 和 jsPDF实现）](https://github.com)


  * [html2canvas](#html2canvas)
* [jsPDF](#jspdf)
* [html2canvas \+ jsPDF 实现页面下载](#html2canvas--jspdf-%E5%AE%9E%E7%8E%B0%E9%A1%B5%E9%9D%A2%E4%B8%8B%E8%BD%BD)
* [单页下载（自适应纸）](#%E5%8D%95%E9%A1%B5%E4%B8%8B%E8%BD%BD%E8%87%AA%E9%80%82%E5%BA%94%E7%BA%B8):[cmespeed楚门加速器](https://77yingba.com)
* [单页下载（A4纸）](#%E5%8D%95%E9%A1%B5%E4%B8%8B%E8%BD%BDa4%E7%BA%B8)
* [多页下载（自适应纸）](#%E5%A4%9A%E9%A1%B5%E4%B8%8B%E8%BD%BD%E8%87%AA%E9%80%82%E5%BA%94%E7%BA%B8)
* [多页下载（A4纸）](#%E5%A4%9A%E9%A1%B5%E4%B8%8B%E8%BD%BDa4%E7%BA%B8)
* [源码](#%E6%BA%90%E7%A0%81)

   \_\_EOF\_\_

   ![](https://github.com/burc)柏成  - **本文链接：** [https://github.com/burc/p/18568288](https://github.com)
 - **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
