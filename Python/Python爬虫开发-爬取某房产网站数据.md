## **背景**
最近跟一个朋友聊天，他工作上被安排了一个任务，需要对某个城市的二手房数据进做一下分析，而这些房源的数据基本都在几大房产交易网站上，如果靠他个人一页一页的浏览下载太慢了，在如今工作如此内卷的情况下，等他下载完数据分析完，老板估计得立马让他走人了。

但如果用程序自动访问这些页面，解析页面数据，然后获取需要的数据到本地，再进行分析，就能快很多了。而这个过程用Python来完成正合适不过，于是我告诉他这个简单，我来帮他搞定，朋友听完安心地泡了一杯茶继续内卷去了，于是，我开始负责搞定这个Python爬虫脚本。


## **Python爬虫开发步骤**
其实，无论是用Python还是用其他编程语言来开发一个爬虫爬取某个网站的数据，一般都会分为如下几个步骤：
1. 待爬取页面的访问url探索和获取；
2. 待爬取页面的页面元素的探查和分析；
3. 访问url获取网页数据；
4. 解析网页数据获取自己想要的结果；
5. 将结果数据保存到本地文件；

所以，接下来，我就分如上5个步骤分别讲解一下这个Pytho爬虫脚本如何开发。

### **待爬取页面的访问url探索和获取**
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


