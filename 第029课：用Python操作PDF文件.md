## 第029课：用Python操作PDF文件

PDF是Portable Document Format的缩写，这类文件通常使用`.pdf`作为其扩展名。在日常开发工作中，最容易遇到的就是从PDF中读取文本内容以及用已有的内容生成PDF文档这两个任务。

### 从PDF中提取文本

在Python中，可以使用名为`PyPDF2`的三方库来读取PDF文件，可以使用下面的命令来安装它。

```Bash
pip install PyPDF2 -i https://pypi.doubanio.com/simple
```

`PyPDF2`没有办法从PDF文档中提取图像、图表或其他媒体，但它可以提取文本，并将其返回为Python字符串。

```Python
import PyPDF2

reader = PyPDF2.PdfFileReader('test.pdf')
page = reader.getPage(0)
print(page.extractText())
```

当然，`PyPDF2`并不是什么样的PDF文档都能提取出文字来，这个问题就我所知并没有什么特别好的解决方法，尤其是在提取中文的时候。之前给成都一汽大众做企业内训的时候，就有学员提出了一个从报关信息的PDF文档中提取中文文本内容的需求，但是我尝试了多种方法都失败了。网上也有很多讲解从PDF中提取文字的文章，推荐大家自行阅读[《三大神器助力Python提取pdf文档信息》](https://cloud.tencent.com/developer/article/1395339)一文进行了解。

要从PDF文件中提取文本也可以使用一个三方的命令行工具，具体的做法如下所示。

```Bash
pip install pdfminer.six
pdf2text.py test.pdf
```

### 旋转和叠加页面

上面的代码中通过创建`PdfFileReader`对象的方式来读取PDF文档，该对象的`getPage`方法可以获得PDF文档的指定页并得到一个`Page`对象，利用`Page`对象的`rotateClockwise`和`rotateCounterClockwise`方法可以实现页面的顺时针和逆时针方向旋转，代码如下所示。

```Python
import PyPDF2

reader = PyPDF2.PdfFileReader('test.pdf')
page = reader.getPage(0)
page.rotateClockwise(90)
writer = PyPDF2.PdfFileWriter()
writer.addPage(page)
with open('test-rotated.pdf', 'wb') as file:
    writer.write(file)
```

`Page`对象还有一个名为`mergePage`的方法，可以将一个页面和另一个页面进行叠加，对于叠加后的页面，我们还是使用`PdfFileWriter`对象的`addPage`将其添加到一个新的`PDF`文档中，有兴趣的读者可以自行尝试。

### 加密PDF文件

使用`PyPDF2`中的`PdfFileWrite`对象可以为PDF文档加密，如果需要给一系列的PDF文档设置统一的访问口令，使用Python程序来处理就会非常的方便。

```Python
import PyPDF2

with open('test.pdf', 'rb') as file:
   	# 通过PdfReader读取未加密的PDF文档
    reader = PyPDF2.PdfFileReader(file)
    writer = PyPDF2.PdfFileWriter()
    for page_num in range(reader.numPages):
        # 通过PdfReader的getPage方法获取指定页码的页
        # 通过PdfWriter方法的addPage将添加读取到的页
        writer.addPage(reader.getPage(page_num))
    # 通过PdfWriter的encrypt方法加密PDF文档
    writer.encrypt('foobared')
    # 将加密后的PDF文档写入指定的文件中
    with open('test-encrypted.pdf', 'wb') as file2:
        writer.write(file2)
```

> **提示**：按照上面的方法也可以读取多个PDF文件的多个页面并将其合并到一个PDF文件中。

### 创建PDF文件

创建PDF文档需要三方库`reportlab`的支持，该库的使用方法可以参考它的[官方文档](https://www.reportlab.com/docs/reportlab-userguide.pdf)，安装的方法如下所示。

```Bash
pip install reportlab
```

下面通过一个例子为大家展示`reportlab`的用法。

```Python
import random

from reportlab.lib import colors
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas
from reportlab.platypus import Table, TableStyle

# 创建Canvas对象（PDF文档对象）
doc = canvas.Canvas('demo.pdf', pagesize=A4)
# 获取A4纸的尺寸
width, height = A4
# 读取图像
image = canvas.ImageReader('guido.jpg')
# 通过PDF文档对象的drawImage绘制图像内容
doc.drawImage(image, (width - 250) // 2, height - 475, 250, 375)
# 设置字体和颜色
doc.setFont('Helvetica', 32)
doc.setFillColorRGB(0.8, 0.4, 0.2)
# 通过PDF文档对象的drawString输出字符串内容
doc.drawString(10, height - 50, "Life is short, I use Python!")
# 保存当前页创建新页面
doc.showPage()
# 准备表格需要的数据
scores = [[random.randint(60, 100) for _ in range(3)] for _ in range(5)]
names = ('Alice', 'Bob', 'Jack', 'Lily', 'Tom')
for row, name in enumerate(names):
    scores[row].insert(0, name)
scores.insert(0, ['Name', 'Verbal', 'Math', 'Logic'])
# 创建一个Table对象（第一个参数是数据，第二个和第三个参数是列宽和行高）
table = Table(scores, 50, 20)
# 设置表格样式（对齐方式和内外边框粗细颜色）
table.setStyle(TableStyle([
    ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
    ('INNERGRID', (0, 0), (-1, -1), 0.25, colors.red),
    ('BOX', (0, 0), (-1, -1), 0.25, colors.black)
]))
table.split(0, 0)
# 通过Table对象的drawOn在PDF文档上绘制表格
table.drawOn(doc, (width - 200) // 2, height - 150)
# 保存当前页创建新页面
doc.showPage()
# 保存PDF文档
doc.save()
```

> **说明**：上面的代码使用了很多字面常量来指定位置和尺寸，在商业项目开发中应该避免这样的硬代码（hard code），因为这样的代码不仅可读性差，维护起来也是一场噩梦。如果项目中需要动态生成PDF文档且PDF文档的格式是相对固定的，可以将上面的字面常量处理成符号常量。记住：**符号常量优于字面常量**。Python语言没有内置定义常量的语法，但是可以约定变量名使用全大写字母的变量就是常量。

###  简单的总结

在学习完上面的内容之后，相信大家已经知道像合并多个PDF文件这样的工作应该如何用Python代码来处理了，赶紧自己动手试一试吧。

> **温馨提示**：学习中如果遇到困难，可以加**QQ交流群**询问。
>
> 付费群：**789050736**，群一直保留，供大家学习交流讨论问题。
>
> 免费群：**151669801**，仅供入门新手提问，定期清理群成员。