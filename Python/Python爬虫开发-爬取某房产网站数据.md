## **背景**
最近跟一个朋友聊天的过程中，朋友提到最近工作上被老板安排了一个任务，要她对某个城市的二手房数据进做一下分析，而这些房源的数据基本都在几大房产交易网站上，如果靠她个人一页一页的浏览下载太慢了，在如今工作如此内卷的情况下，等她下载完数据分析完，老板估计得立马让她卷铺盖走人了。

但如果用程序自动访问这些页面，解析页面数据，然后获取需要的数据到本地，再进行分析，就能快很多了。而这个过程用Python来完成正合适不过，于是我告诉她这个简单，我来帮她搞定，朋友听完安心地点了一杯咖啡继续内卷去了，于是，我开始负责搞定这个Python爬虫脚本。


## **Python爬虫开发步骤**
其实，无论是用Python还是用其他编程语言来开发一个爬虫爬取某个网站的数据，一般都会分为如下几个步骤：
1. 待爬取页面的访问url探索和获取；
2. 待爬取页面的页面元素的探查和分析；
3. 访问url获取网页数据；
4. 解析网页数据获取自己想要的结果；
5. 将结果数据保存到本地文件；

所以，接下来，我就分如上5个步骤分别讲解一下这个Pytho爬虫脚本如何开发。

### **待爬取页面的访问url探索和获取**
步骤一是对待爬取页面访问url的探索和获取

有人说，获取要爬取页面的访问url还不简单，网站上点一下链接看一下浏览器地址栏的链接地址不就行了。大家可以去试试，有的网站可能确实如此简单，但有的网站靠这样的方式是获取不到的。

比如，我们先拿一个简单的网站来说，比如有这样一个博客网站[《蚂蚁学Phthon》](http://www.crazyant.net/)，假如我想爬取这个网站上的博客文章列表，如下图红框所示：

![crazyant-page](https://github.com/xiaoyuge/Tech-Notes/blob/main/Python/resources/crazyant-page.png)

大家可以点一下红框的文章以及点一下翻页，然后看一下浏览器地址栏的链接地址，可以看到，每点一个文章或者翻页，浏览器地址栏的链接地址都会变化，反映的是当前你访问的url地址，具体是长如下这样的：
```Python
#每篇文章url的正则
pattern = r'^http://www.crazyant.net/\d+.html$'
#文章列表url的正则
pattern = r'^http://www.crazyant.net/page/\d+$'
```

所以，这个网站假如我们要爬取所有的文章列表数据，那就很简单了，因为从浏览器地址栏我们就可以很方便地获取到要爬取的每个文章列表页的url地址，然后根据特征用正则进行匹配。

但不是所有网站都是如此简单的，我们再来看一下另外一个博客网站[博客园](https://www.cnblogs.com/)，大家可以自己访问看一下。假如我同样希望爬取这个博客网站所有的文章列表数据，那如何获取到每个文章列表页面的url呢？

如果大家实际点过翻页就会发现，点完后浏览器地址栏的地址是这样的：

![cnblogs-page](https://github.com/xiaoyuge/Tech-Notes/blob/main/Python/resources/cnblogs-page.png)

也即浏览器地址是长这样子的：
```Python
#文章列表页的正则
pattern = r'^https://www.cnblogs.com/#p\d+$
```

我们会看到地址里面有一个#号，如果我们直接访问这个url，能获取到我们想要的文章列表数据吗？大家可以用下面的代码自己试一下：
```Python
import requests

r = requests.get('https://www.cnblogs.com/#p5',timeout=3)
if r.status_code == 200:
    resp = r.text
print(resp)
```
可以看到，直接访问浏览器地址的url是拿不到我们希望的文章列表页的返回数据的。因为这个网站是一个所谓的SPA应用（Single Page Application），地址栏的#号是一个Hash，改变Hash 不会重新加载页面。SPA应用的典型特征是，首次访问会加载整个页面框架，然后通过Ajax/Fetch异步获取数据更新页面内容。

所以，在这种情况下，我们只能借助于抓包工具比如chrome的开发者工具，来探索和获取真实的Ajax/Fetch的url，如下图所示：

![cnblogs-chrome-dev-tools](https://github.com/xiaoyuge/Tech-Notes/blob/main/Python/resources/cnblogs-page-chrome-dev-tools.png)

Chrome的开发者工具中的Network面板，可以很方便的获取到当前访问页面的各类请求资源，比如静态的Html/js/css/图片、动态的Fetch/XHR异步请求等，我们打开Network面板下的子面板Fetch/XHR，就可以看到所有的异步请求。点击某一个异步请求，我们就可以查看这个请求的request header、response header、cookies、请求提交的数据等信息。

我们通过探查，可以发现，有一个POST的异步请求：
```
https://www.cnblogs.com/AggSite/AggSitePostList
```
在他的Playload也即post提交的数据里，有一个"PageIndex:5"的字段，正好对应当前的页码，通过进一步探查response面板的内容，我们就可以进一步确认，这个异步请求才是我们真正要爬取的url。

所以，大家看到了，对于爬虫开发的第一步，并不都是如大家想象的那么简单，对于有的网站，还是需要利用类似Chrome开发者工具这样的抓包工具做一些探查工作的。

好了，利用上述的思路，我也对我们这次真正要爬取的某房产网站页面进行了一次探查，获取到了我要爬取的url，这次要爬取的网站的url获取还是比较简单的，属于第一种情况，在这里就不详述过程了。

### **待爬取页面的页面元素的探查和分析**
步骤二是对待爬取页面的页面元素进行探查和分析

对页面元素进行探查分析的工作，还是会用到我们之前提到的Chrome开发者工具，而要探查什么内容，则取决于我们要爬取的数据是什么。比如这次我们的需求是要爬取某房产网站的二手房房源新消息，那我们的探查就会类似下图所示：

![page-element-explore]()





