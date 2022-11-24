## **背景**
在上一篇文章[《用Python开发爬虫爬取某房产网站数据》](https://github.com/xiaoyuge/Tech-Notes/blob/main/Python/%E7%94%A8Python%E5%BC%80%E5%8F%91%E7%88%AC%E8%99%AB%E7%88%AC%E5%8F%96%E6%9F%90%E6%88%BF%E4%BA%A7%E7%BD%91%E7%AB%99%E6%95%B0%E6%8D%AE.md)的末尾，我们讲到，虽然用Python开发爬虫脚本，顺利把某房产网站的数据给爬取下来了，但是我朋友老板安排的工作并没有完成，我们还需要对这份数据进行分析并生成分析报告，所以，这篇文章就接着上篇文章，讲解一下，如果用Python做一份好看又好用的数据分析报告。

## **Python库的选择**
话说，工欲善其事，必先利其器，虽然我们已经选择Python来完成剩余的工作，但是我们需要考虑具体选择使用Pytho的哪些利器来帮助我们更快更好地完成剩余的工作。

我们可以看一下，在这个任务中，主要涉及到四类工作要完成：
1. csv文件的读取；
2. 对读取的数据，按照我们要分析的指标进行数据处理和指标计算；
3. 根据数据分析的结果，生成可视化的数据图表；
4. 通过web页面展示数据分析结果报告；

我们下面就根据这四类工作，来看看我们分别选择Python的哪些库来帮助我们完成工作。

### **1.数据处理和分析库**

对类似csv、excel等格式文件的读取和处理，其实就是对一维和二维数据的处理，对此类数据的处理，Python中常用的库是Pandas，其提供的数据结构中的Series对应一维数据，DataFrame对应二维数据，同时Pandas也提供了大量的高效内置函数和操作来实现对内存中一维和二维数据的处理。

而对于更高维度数据比如矩阵的计算，Python中则需要用Nunpy库来完成。numpy是以矩阵为基础的数学计算模块，提供高性能的矩阵运算，数组结构为ndarray，可以把它看作是多维数组（ndarray）的容器，可以对数组执行元素级计算以及直接对数组执行数学运算的函数。

Pandas是基于Numpy数组构建的，但二者最大的不同是pandas是专门为处理表格和混杂数据设计的，比较契合统计分析中的表结构，而numpy更适合处理统一的数值数组数据。

所以，第1步和第2步的工作，我们基本依靠Pandas库就能完成，不过，这次的数据分析报告中，我也用到了Numpy库的直方图计算的功能，后面会详细讲到。

### **2.数据可视化库**

而第3步的工作，其实是一个数据可视化的任务，在Python中可以用于进行数据可视化的库，常用的主要有三个：
- Matplotlib
- Seanborn
- Pyecharts

####  **Matplotlib**
Matplotlib可以说是Python数据可视化库的鼻祖了，他是Python编程语言及其数值计算包NumPy的可视化操作界面，其中pyplot是matplotlib的一个模块，提供了类似MATLAB的接口。其可以和Numpy、Pandas无缝结合，但一些图标的样式不够美观，而且原生不支持生成动态可交互的图表，虽然可以通过改变使用的后端来实现，但相对还是比较麻烦一些，而且如果想要在一个web页面中实现一个动态可交互的图表，目前没有什么特别好的办法，最近matplotlib在更面向web交互方面有了很多进展，比如新的HTML5/Canvas后端，可以从如下地址了解一下：
```HTML
http://code.google.com/p/mplh5canvas/
```
但还没有完全完成。

####  **Seanborn**
Seaborn跟matplotlib最大的区别就是它的默认绘图风格和色彩搭配都具有现代美感，其实他是在matplotlib的基础上进行了更高级的API封装，让你能用更少的代码去调用matplotlib的方法，从而使得作图更加容易。但matplotlib存在的动态交互性的问题他同样存在。

#### **Pyecharts**
说到Pyecharts则不得不提到ECharts，这个可是在前端数据可视化领域非常知名的库了，毕竟他出自我的老东家百度的前端工程师之手，最开始在百度内部孵化，我在百度工作期间，还和后来参与到ECharts开发的核心工程师有过其他项目合作。后来2018年捐赠给Apache基金会，成为ASF孵化级项目，并于2021年正式毕业，成为Apache顶级项目。

而Pyecharts则是基于ECharts实现的python版本，支持大量丰富的可视化图表类型，而且相比前两个库最大的优势在于，能够非常方便地生成支持交互性（如鼠标点选、拖拽、缩放等）的图片，且可动态地展示在web页面上

基于以上的对比分析，鉴于这次我希望给我朋友生成一个动态可交互的web数据分析报告页面，在这一点上，Pyecharts无疑更有优势，于是这次我们就用Pyecharts库来进行我们的数据可视化展现。

### **3.Web应用库**
在这个领域Python的选择主要有两个：
- Django
- Flask

Django是用 Python 开发的一个免费开源的 Web 框架，提供了许多网站后台开发经常用到的模块，本身自带了相当多的功能，这些功能是由官方和社区共同维护的，因而是个大而全的较重的框架，所以耦合度相比flask会高一些，做二次修改难度更高。

相比之下，Flask是一个免费的开放源代码的轻量型的Web框架，Flask不包含例如上载处理，ORM（对象关系映射器），数据库抽象层，身份验证，表单验证等web应用常用功能模块（这些Django提供了），但是可以使用预先存在的外部库来集成这些功能，因此是一个更灵活、扩展性更好的Web框架。

而我们这次的场景，仅仅只需要提供一个静态的web页面用于展示数据可视化结果，并不涉及其他复杂的web应用功能，因此，Flask是我们的不二之选。

## **开始我们的数据可视化分析之旅**

好了，选择好了我们的工具之后，我们就要正式开始我们的数据可视化分析之旅了。我们先来看一下我们要分析的这一份数据，如下图所示：

![anjuke-data](https://github.com/xiaoyuge/Tech-Notes/blob/main/Python/resources/anjuke_data.png)

数据读取到内存的过程使用Pandas来完成很简单，这里就不赘述了。接下来重点讲一下数据分析和可视化图表的生成。根据要分析的数据指标，这次我们主要用到了Pyecharts的5类图表组件，分别是Bar（柱状图）、Pie（饼图）
、Scatter（散点图）、Map（地图）和WordCloud（词云图），接下来就分别介绍一下。

- Bar（柱状图）

这次的数据分析报告中，在分析按房屋面积区间的房屋单价、按户型的房屋单价以及小区房价Top10这三个数据报告中，我们使用了柱状图，因为柱状图非常适合按照Category来展示相应的数值以及对比关系。我们就以小区房价Top10为例，来看一下如何生成柱状图。

其实主要过程包括两个步骤：
- 数据计算处理
- 数据可视化处理

我们先来看第一步的数据计算处理。因为要找到这个城市小区房价的Top10，所以我们主要完成如下几个计算步骤：
1. 根据原始数据表中的”小区名称“字段进行group by；
2. 对每个分组，对”均价“字段求平均值；
3. 对上述结果的”均价“字段按降序进行排序；
4. 对排序结果取前10项结果；

完成上述四个计算步骤的代码如下所示：

```Python
def unit_price_analysis_by_estate(df,isembed):
    #获取要分析的数据列
    analysis_df = df.loc[:,['小区名称','均价']]
    analysis_df.loc[:,'小区名称'] = analysis_df.loc[:,'小区名称'].astype('str')
    #对小区名称分组，然后按照分组计算单价均价
    group = analysis_df.groupby('小区名称',as_index=False)
    group_df = group.mean()
    group_df.loc[:,'均价'] = group_df.loc[:,'均价'].astype('int')
    #按照均价列降序排序
    group_df.sort_values('均价',ascending=False, inplace=True)
    #取Top10
    top10_df = group_df.head(10)
    #为了横向柱状图展示，再从低到高排序一下
    top10_df.sort_values('均价',ascending=True,inplace=True)
    ......
```
其实如果是生成常规的纵向柱状图的话，上面的代码里最后一步是不需要的。但因为要生成横向柱状图，需要对纵向柱状图进行一个reverse()操作，在reverse()操作后如果要保持从上至下降序的顺序，我们的对Top10的排序结果也需要倒置一下。

接下来就是柱状图的数据可视化图表生成部分了，这部分代码如下：
```Python
 bar = (
        Bar(init_opts=opts.InitOpts(width="1500px"))
        .add_xaxis(top10_df['小区名称'].tolist())
        .add_yaxis("房价单价",top10_df['均价'].tolist(),itemstyle_opts=opts.ItemStyleOpts(color=JsCode(top10_color_function)))
        .reversal_axis()
        .set_series_opts(label_opts=opts.LabelOpts(position="right"))
        .set_global_opts(title_opts=opts.TitleOpts(title="苏州各小区二手房房价TOP10"),
                         xaxis_opts=opts.AxisOpts(axislabel_opts={'interval':'0'}),
                         legend_opts=opts.LegendOpts(is_show=False))
    )
```
关于代码里详细的参数设置我就不一一解释了，大家可以去Pyecharts的官网查看到每个图表非常详细的参数解释和demo代码。

在这里唯一额外提一下的，就是关于如何给柱状图不同的柱子设置不同的颜色，需要我们提供一个自定义的js函数来实现，Pyecharts提供了这样的机制，可以让我们嵌入这样的js函数来完成部分自定义的功能，比如我是这样来实现的：
```Javascript
top10_color_function = """
        function (params) {
            if (params.value > 58000 && params.value < 59000) {
                return 'red';
            } else if (params.value > 59000 && params.value < 60000) {
                return 'blue';
            }else if (params.value > 60000 && params.value < 61000){
                return 'green'
            }else if (params.value > 61000 && params.value < 61800){
                return 'purple'
            }else if (params.value > 61800 && params.value < 70000){
                return 'brown'
            }else if (params.value > 70000 && params.value < 73000){
                return 'gray'
            }else if (params.value > 73000 && params.value < 79000){
                return 'orange'
            }else if (params.value > 79000 && params.value < 85000){
                return 'pink'
            }else if (params.value > 85000 && params.value < 100000){
                return 'navy'
            }
            return 'gold';
        }
        """
```
完成上述两个步骤后，我们的横向柱状图就生成了，如下图所示：

![](bar-top10)

