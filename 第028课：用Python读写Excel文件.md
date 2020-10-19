## 第028课：用Python读写Excel文件

### Excel简介

Excel是Microsoft（微软）为使用Windows和macOS操作系统开发的一款电子表格软件。Excel凭借其直观的界面、出色的计算功能和图表工具，再加上成功的市场营销，一直以来都是最为流行的个人计算机数据处理软件。当然，Excel也有很多竞品，例如Google Sheets、LibreOffice Calc、Numbers等，这些竞品基本上也能够兼容Excel，至少能够读写较新版本的Excel文件，当然这些不是我们讨论的重点。掌握用Python程序操作Excel文件，可以让日常办公自动化的工作更加轻松愉快，而且在很多商业项目中，导入导出Excel文件都是特别常见的功能。

Python操作Excel需要三方库的支持，如果要兼容Excel 2007以前的版本，也就是`xls`格式的Excel文件，可以使用三方库`xlrd`和`xlwt`，前者用于读Excel文件，后者用于写Excel文件。如果使用较新版本的Excel，即操作`xlsx`格式的Excel文件，也可以使用`openpyxl`库，当然这个库不仅仅可以操作Excel，还可以操作其他基于Office Open XML的电子表格文件。

下面我们以`xlwt`和`xlrd`为例讲解如何读写Excel文件，大家可以先使用下面的命令安装这两个三方库文件。

```Bash
pip install xlwt xlrd -i https://pypi.doubanio.com/simple
```

### 使用xlwt和xlrd

#### 读Excel文件

例如在当前文件夹下有一个名为“阿里巴巴2017年股票数据.xlsx”的Excel文件，如果想读取并显示该文件的内容，可以通过如下所示的代码来完成。

```Python
import xlrd

# 使用xlrd模块的open_workbook函数打开指定Excel文件并获得Book对象（工作簿）
wb = xlrd.open_workbook('阿里巴巴2017年股票数据.xlsx')
# 通过Book对象的sheet_names方法可以获取所有表单名称
sheetname = wb.sheet_names()[0]
# 通过指定的表单名称获取Sheet对象（工作表）
sheet = wb.sheet_by_name(sheetname)
# 通过Sheet对象的nrows和ncols属性获取表单的行数和列数
print(sheet.nrows, sheet.ncols)
for row in range(sheet.nrows):
    for col in range(sheet.ncols):
        # 通过Sheet对象的cell方法获取指定Cell对象（单元格）
        # 通过Cell对象的value属性获取单元格中的值
        value = sheet.cell(row, col).value
        # 对除首行外的其他行进行数据格式化处理
        if row > 0:
            # 第1列的xldate类型先转成元组再格式化为“年月日”的格式
            if col == 0:
                # xldate_as_tuple函数的第二个参数只有0和1两个取值
                # 其中0代表以1900-01-01为基准的日期，1代表以1904-01-01为基准的日期
                value = xlrd.xldate_as_tuple(value, 0)
                value = f'{value[0]}年{value[1]:>02d}月{value[2]:>02d}日'
            # 其他列的number类型处理成小数点后保留两位有效数字的浮点数
            else:
                value = f'{value:.2f}'
        print(value, end='\t')
    print()
# 获取最后一个单元格的数据类型
# 0 - 空值，1 - 字符串，2 - 数字，3 - 日期，4 - 布尔，5 - 错误
last_cell_type = sheet.cell_type(sheet.nrows - 1, sheet.ncols - 1)
print(last_cell_type)
# 获取第一行的值（列表）
print(sheet.row_values(0))
# 获取指定行指定列范围的数据（列表）
# 第一个参数代表行索引，第二个和第三个参数代表列的开始（含）和结束（不含）索引
print(sheet.row_slice(3, 0, 5))
```

相信通过上面的代码，大家已经了解到了如何读取一个Excel文件，如果想知道更多关于`xlrd`模块的知识，可以阅读它的[官方文档](https://xlrd.readthedocs.io/en/latest/)。

#### 写Excel文件

写入Excel文件可以通过`xlwt` 模块的`Workbook`类创建工作簿对象，通过工作簿对象的`add_sheet`方法可以添加工作表，通过工作表对象的`write`方法可以向指定单元格中写入数据，最后通过工作簿对象的`save`方法将工作簿写入到指定的文件或内存中。下面的代码实现了将5个学生3门课程的考试成绩写入Excel文件的操作。

```Python
import random

import xlwt

student_names = ['关羽', '张飞', '赵云', '马超', '黄忠']
scores = [[random.randint(40, 100) for _ in range(3)] for _ in range(5)]
# 创建工作簿对象（Workbook）
wb = xlwt.Workbook()
# 创建工作表对象（Worksheet）
sheet = wb.add_sheet('一年级二班')
# 添加表头数据
titles = ('姓名', '语文', '数学', '英语')
for index, title in enumerate(titles):
    sheet.write(0, index, title)
# 将学生姓名和考试成绩写入单元格
for row in range(len(scores)):
    sheet.write(row + 1, 0, student_names[row])
    for col in range(len(scores[row])):
        sheet.write(row + 1, col + 1, scores[row][col])
# 保存Excel工作簿
wb.save('考试成绩表.xlsx')
```

#### 单元格样式



#### 公式计算



###  简单的总结

其他操作Excel文件的三方库（如`openpyxl`）大家有兴趣可以自行了解。掌握了Python程序操作Excel的方法，可以解决日常办公中很多繁琐的处理Excel电子表格工作，最常见的就是将多个数据格式相同的Excel文件合并到一个文件以及从多个Excel文件或表单中提取指定的数据。本章例子中使用的Excel文件，大家可以从我的百度云盘链接中进行下载，地址：<https://pan.baidu.com/s/1rQujl5RQn9R7PadB2Z5g_g>，提取码：e7b4。

> **温馨提示**：学习中如果遇到困难，可以加**QQ交流群**询问。
>
> 付费群：**789050736**，群一直保留，供大家学习交流讨论问题。
>
> 免费群：**151669801**，仅供入门新手提问，定期清理群成员。