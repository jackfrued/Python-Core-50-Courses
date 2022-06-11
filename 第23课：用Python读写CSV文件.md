## 第23课：用Python读写CSV文件

### CSV文件介绍

CSV（Comma Separated Values）全称逗号分隔值文件是一种简单、通用的文件格式，被广泛的应用于应用程序（数据库、电子表格等）数据的导入和导出以及异构系统之间的数据交换。因为CSV是纯文本文件，不管是什么操作系统和编程语言都是可以处理纯文本的，而且很多编程语言中都提供了对读写CSV文件的支持，因此CSV格式在数据处理和数据科学中被广泛应用。

CSV文件有以下特点：

1. 纯文本，使用某种字符集（如[ASCII](https://zh.wikipedia.org/wiki/ASCII)、[Unicode](https://zh.wikipedia.org/wiki/Unicode)、[GB2312](https://zh.wikipedia.org/wiki/GB2312)）等）；
2. 由一条条的记录组成（典型的是每行一条记录）；
3. 每条记录被分隔符（如逗号、分号、制表符等）分隔为字段（列）；
4. 每条记录都有同样的字段序列。

CSV文件可以使用文本编辑器或类似于Excel电子表格这类工具打开和编辑，当使用Excel这类电子表格打开CSV文件时，你甚至感觉不到CSV和Excel文件的区别。很多数据库系统都支持将数据导出到CSV文件中，当然也支持从CSV文件中读入数据保存到数据库中，这些内容并不是现在要讨论的重点。

### 将数据写入CSV文件

现有五个学生三门课程的考试成绩需要保存到一个CSV文件中，要达成这个目标，可以使用Python标准库中的`csv`模块，该模块的`writer`函数会返回一个`csvwriter`对象，通过该对象的`writerow`或`writerows`方法就可以将数据写入到CSV文件中，具体的代码如下所示。

```Python
import csv
import random

with open('scores.csv', 'w') as file:
    writer = csv.writer(file)
    writer.writerow(['姓名', '语文', '数学', '英语'])
    names = ['关羽', '张飞', '赵云', '马超', '黄忠']
    for name in names:
        scores = [random.randrange(50, 101) for _ in range(3)]
        scores.insert(0, name)
        writer.writerow(scores)
```

生成的CSV文件的内容。

```
姓名,语文,数学,英语
关羽,98,86,61
张飞,86,58,80
赵云,95,73,70
马超,83,97,55
黄忠,61,54,87
```

需要说明的是上面的`writer`函数，除了传入要写入数据的文件对象外，还可以`dialect`参数，它表示CSV文件的方言，默认值是`excel`。除此之外，还可以通过`delimiter`、`quotechar`、`quoting`参数来指定分隔符（默认是逗号）、包围值的字符（默认是双引号）以及包围的方式。其中，包围值的字符主要用于当字段中有特殊符号时，通过添加包围值的字符可以避免二义性。大家可以尝试将上面第5行代码修改为下面的代码，然后查看生成的CSV文件。

```Python
writer = csv.writer(file, delimiter='|', quoting=csv.QUOTE_ALL)
```

生成的CSV文件的内容。

```
"姓名"|"语文"|"数学"|"英语"
"关羽"|"88"|"64"|"65"
"张飞"|"76"|"93"|"79"
"赵云"|"78"|"55"|"76"
"马超"|"72"|"77"|"68"
"黄忠"|"70"|"72"|"51"
```

### 从CSV文件读取数据

如果要读取刚才创建的CSV文件，可以使用下面的代码，通过`csv`模块的`reader`函数可以创建出`csvreader`对象，该对象是一个迭代器，可以通过`next`函数或`for-in`循环读取到文件中的数据。

```Python
import csv

with open('scores.csv', 'r') as file:
    reader = csv.reader(file, delimiter='|')
    for data_list in reader:
        print(reader.line_num, end='\t')
        for elem in data_list:
            print(elem, end='\t')
        print()
```

> **注意**：上面的代码对`csvreader`对象做`for`循环时，每次会取出一个列表对象，该列表对象包含了一行中所有的字段。

###  简单的总结

将来如果大家使用Python做数据分析，很有可能会用到名为`pandas`的三方库，它是Python数据分析的神器之一。`pandas`中封装了名为`read_csv`和`to_csv`的函数用来读写CSV文件，其中`read_CSV`会将读取到的数据变成一个`DataFrame`对象，而`DataFrame`就是`pandas`库中最重要的类型，它封装了一系列用于数据处理的方法（清洗、转换、聚合等）；而`to_csv`会将`DataFrame`对象中的数据写入CSV文件，完成数据的持久化。`read_csv`函数和`to_csv`函数远远比原生的`csvreader`和`csvwriter`强大。
