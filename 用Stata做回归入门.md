# 用Stata做回归入门

本文给出了一些使用`Stata`做回归的资料链接，包括`Stata`的一些入门资料。这里假设只需要用`Stata`做回归以及回归结果的列报，数据的整理和清洗已经做好。

将数据整理好后，就可以用`Stata`跑回归，并且将回归结果列成表格，就像paper中的格式。这一部分工作最好用`Stata`来做，其他语言对部分的计量分析支持的不太好。并且`Stata`在回归报告方面有独特的优势，输出格式最好只需要微调就可以放在paper中。

`Stata`比较好的入门材料有官方的[User's Guild](http://www.stata.com/bookstore/users-guide/) 。其他流行的资料还有UCLA的网站
http://www.ats.ucla.edu/stat/stata/


比较常用的**回归**的命令，列在
https://github.com/aijinzhengu/undergrad_2016/blob/master/stata_reg.md

`Stata`官方给出的`Commands everyone should know`请见http://www.stata.com/manuals14/u27.pdf。这更侧重于一般功能，特别是数据整理。


需要做哪种回归的时候可以再查看具体帮助文件，如需要做面板回归，可以在命令窗口输入：

```stata
help areg
help xtreg
```

在follow已有文献的方法时，`Stata`几乎提供了所有的回归分析功能，细节的地方需要看帮助文件对应命令的`option`。比如做固定效应的面板回归就要用到命令`xtreg`的选项`fe`，对某些变量做cluster需要用到选项`vce(c var)`。

`Stata`是高度模板化的语言，照着模板填就行，基本不需要编程。

有些复杂的任务还可以搜索一些专题性资料，[Stata Journal](http://www.stata.com/bookstore/stata-journal/) 是很好的来源。`Stata`有活跃的社区，有很多第三方包/插件`ado`文件可以使用。

在报告回归结果的时候就要用到`esttab`就是来自第三方的`ado`文件，使用前需要从互联网下载安装：

```stata
ssc install estout
help esttab    

```





