## 第02课：第一个Python程序

在上一课中，我们已经了解了Python语言并安装了运行Python程序所需的环境，相信大家已经迫不及待的想开始自己的Python编程之旅了。首先我们来看看应该在哪里编写我们的Python程序。

### 编写代码的工具

#### 交互式环境

我们打开Windows的“命令提示符”工具，输入命令`python`然后回车就可以进入到Python的交互式环境中。所谓交互式环境，就是我们输入一行代码回车，代码马上会被执行，如果代码有产出结果，那么结果会被显示在窗口中。例如：

```Bash
Python 3.7.6
Type "help", "copyright", "credits" or "license" for more information.
>>> 2 * 3
6
>>> 2 + 3
5
```

> **提示**：使用macOS系统的用户需要打开“终端”工具，输入`python3`进入交互式环境。

如果希望退出交互式环境，可以在交互式环境中输入`quit()`，如下所示。

```Bash
>>> quit()
```

#### 更好的交互式环境 - IPython

Python默认的交互式环境用户体验并不怎么好，我们可以用IPython来替换掉它，因为IPython提供了更为强大的编辑和交互功能。我们可以使用Python的包管理工具`pip`来安装IPython，如下所示。

```bash
pip install ipython
```

> **温馨提示**：在使用上面的命令安装IPython之前，可以先通过`pip config set global.index-url https://pypi.doubanio.com/simple`命令将`pip`的下载源修改为国内的豆瓣网，否则下载安装的过程可能会非常的缓慢。

可以使用下面的命令启动IPython，进入交互式环境。

```bash
ipython
```

#### 文本编辑器 - Visual Studio Code

Visual Studio Code（通常简称为VSCode）是一个由微软开发能够在Windows、 Linux和macOS等操作系统上运行的代码编辑神器。它支持语法高亮、自动补全、多点编辑、运行调试等一系列便捷功能，而且能够支持多种编程语言。如果大家要选择一款高级文本编辑工具，强烈建议使用VSCode。关于VSCode的[下载](https://code.visualstudio.com/)、安装和使用，推荐大家阅读一篇名为[《VScode安装使用》](<https://zhuanlan.zhihu.com/p/106357123>)的文章。

#### 集成开发环境 - PyCharm

如果用Python开发商业项目，我们推荐大家使用更为专业的工具PyCharm。PyCharm是由捷克一家名为[JetBrains](https://www.jetbrains.com/)的公司开发的用于Python项目开发的集成开发环境（IDE)。所谓集成开发环境，通常是指工具中提供了编写代码、运行代码、调试代码、分析代码、版本控制等一系列功能，因此特别适合商业项目的开发。在JetBrains的官方网站上提供了PyCharm的[下载链接](<https://www.jetbrains.com/pycharm/download>)，其中社区版（Community）是免费的但功能相对弱小（其实已经足够强大了），专业版（Professional）功能非常强大，但需要按年或月付费使用，新用户可以试用30天时间。

运行PyCharm，可以看到如下图所示的欢迎界面，可以选择“New Project”来创建一个新的项目。

<img src="https://gitee.com/jackfrued/mypic/raw/master/20210720102203.png" width="80%">

创建项目的时候需要指定项目的路径并创建运行项目的”虚拟环境“，如下图所示。

<img src="https://gitee.com/jackfrued/mypic/raw/master/20210720102822.png" width="80%">

项目创建好以后会出现如下图所示的画面，我们可以通过在项目文件夹上点击鼠标右键，选择“New”菜单下的“Python File”来创建一个Python文件，创建好的Python文件会自动打开进入可编辑的状态。

![image-20210720133621079](https://gitee.com/jackfrued/mypic/raw/master/20210720133621.png)

写好代码后，可以在编辑代码的窗口点击鼠标右键，选择“Run”菜单项来运行代码，下面的“Run”窗口会显示代码的执行结果，如下图所示。

![image-20210720134039848](https://gitee.com/jackfrued/mypic/raw/master/20210720134039.png)

PyCharm常用的快捷键如下表所示，我们也可以在“File”菜单的“Settings”中定制PyCharm的快捷键（macOS系统是在“PyCharm”菜单的“Preferences”中对快捷键进行设置）。

表1. PyCharm常用快捷键。

| 快捷键                                  | 作用                                   |
| --------------------------------------- | -------------------------------------- |
| `ctrl + j`                              | 显示可用的代码模板                     |
| `ctrl + b`                              | 查看函数、类、方法的定义               |
| `ctrl + alt + l`                        | 格式化代码                             |
| `alt + enter`                           | 万能代码修复快捷键                     |
| `ctrl + /`                              | 注释/反注释代码                        |
| `shift + shift`                         | 万能搜索快捷键                         |
| `ctrl + d` / `ctrl + y`                 | 复制/删除一行代码                      |
| `ctrl + shift + -` / `ctrl + shift + +` | 折叠/展开所有代码                      |
| `F2`                                    | 快速定位到错误代码                     |
| `ctrl + alt + F7`                       | 查看哪些地方用到了指定的函数、类、方法 |

> **说明**：使用macOS系统，可以将上面的`ctrl`键换成`command`键，在macOS系统上，可以使用`ctrl + space`组合键来获得万能提示，在Windows系统上不能使用该快捷键，因为它跟Windows默认的切换输入法的快捷键是冲突的，需要重新设置。

### hello, world

按照行业惯例，我们学习任何一门编程语言写的第一个程序都是输出`hello, world`，因为这段代码是伟大的丹尼斯·里奇（C语言之父，和肯·汤普森一起开发了Unix操作系统）和布莱恩·柯尼汉（awk语言的发明者）在他们的不朽著作*The C Programming Language*中写的第一段代码。

```Python
print('hello, world')
```

### 运行程序

如果不使用PyCharm这样的集成开发环境，我们可以将上面的代码命名为`hello.py`，对于Windows操作系统，可以在你保存代码的目录下先按住键盘上的`shift`键再点击鼠标右键，这时候鼠标右键菜单中会出现“命令提示符”选项，点击该选项就可以打开“命令提示符”工具，我们输入下面的命令。

```Shell
python hello.py
```

> **提醒**：我们也可以在任意位置打开“命令提示符”，然后将需要执行的Python代码通过拖拽的方式拖入到“命令提示符”中，这样相当于指定了文件的绝对路径来运行该文件中的Python代码。再次提醒，macOS系统要通过`python3`命令来运行该程序。

你可以尝试将上面程序单引号中的`hello, world`换成其他内容；你也可以尝试着多写几个这样的语句，看看会运行出怎样的结果。需要提醒大家，上面代码中的`print('hello, world')`就是一条完整的语句，我们用Python写程序，最好每一行代码中只有一条语句。虽然使用`;`分隔符可以将多个语句写在一行代码中，但是最好不要这样做，因为代码会变得非常难看。

### 注释你的代码

注释是编程语言的一个重要组成部分，用于在源代码中解释代码的作用从而增强程序的可读性。当然，我们也可以将源代码中暂时不需要运行的代码段通过注释来去掉，这样当你需要重新使用这些代码的时候，去掉注释符号就可以了。简单的说，**注释会让代码更容易看懂但不会影响程序的执行结果**。

Python中有两种形式的注释：

1. 单行注释：以`#`和空格开头，可以注释掉从`#`开始后面一整行的内容。
2. 多行注释：三个引号开头，三个引号结尾，通常用于添加多行说明性内容。

```Python
"""
第一个Python程序 - hello, world

Version: 0.1
Author: 骆昊
"""
# print('hello, world')
print("你好，世界！")
```

### 总结

到这里，我们已经把第一个Python程序运行起来了，是不是很有成就感？只要你坚持学习下去，再过一段时间，我们就可以用Python制作小游戏、编写爬虫程序、完成办公自动化操作等。**写程序本身就是一件很酷的事情**，在未来编程就像英语一样，**对很多人来说或都是必须要掌握的技能**。

