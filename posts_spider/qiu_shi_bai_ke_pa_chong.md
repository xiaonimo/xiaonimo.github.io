# 糗事百科爬虫

**前排提示：**
<li>Python 3.5
<li>没有分布式队列，没有查重，没有Scrapy-Redis框架，没有效率

**参考资料（前排拜谢）;**
<li>[网友静觅]( http://cuiqingcai.com/)
<li>[CSDN专栏](http://blog.csdn.net/column/details/why-bug.html)
<li>[Jecvay Notes](https://jecvay.com/archive)
<li>[知乎大神，言简意赅](https://www.zhihu.com/question/20899988)

###**第一步：能爬就行**

```
import urllib
import urllib.request

url = "http://news.dbanotes.net"
html = urllib.request.urlopen(url)
data = html.read().decode('UTF-8')
print(data)
```

　　通过这样几行代码，我们就可以抓取一个界面了，返回的是html代码。但是这样的爬虫只能爬一个界面，如果想连续的爬应该怎么办呢？

###**第二步：连续爬行**
　　如果想连续爬行，那么就需要在爬的页面上找到链接到其他网页的URL，这里需要用到**正则表达式**来匹配。匹配到的URL需要存储在一个队列中，这样的话就可以持续存取。大型爬虫采用的是Scrapy-Redis框架。与此同时，还要维护一个队列，用来记录哪些URL已经被访问过，以免重复。这个简单的查重方法可以用set()容器实现。但是随着set元素增大，消耗会呈线性增长。大型爬虫采用的是Bloom Filter。

```
import re
from collections import deque

queue = deque()
queue.append('www.baidu.com')#爬虫的起点
visited = set()

while queue:
　　url = queue.popleft()
　　visited |= url

　　linkre = re.compile(r'href="(.+?)"')
　　for x in linkre.findall(data):
　　　　if (len(queue) &lt; 100 and x not in visited):
　　　　　　queue.append(x)
```
　　上述代码中的URL匹配规则很简陋，有可能匹配到的不是URL，所以这个正则表达式还需要写的更精确一些。上述代码中的data，是第一步得到的data。注意，这个data必须是decode处理过的，否则会出错。通过存储一个队列的URL爬虫就可以源源不断的获得URL，通过与set中的元素比较，就可以避免访问重复的URL。但是访问的URL无效或者服务器不响应，那该怎么办？

###**第三步：异常处理**
　　我们在使用浏览器时，可能会遇到页面无法访问的情况，最常见的就是404。404是http的状态码，意思是要访问的页面没有找到，这时我们从浏览器上关闭网页就是了。但是如果在python爬虫中出现这样的错误，那么整个爬虫就会崩溃，从而停止工作。下面给出错误处理方式。

```
try:
　　html = urllib.request.urlopen(url, timeout = 100)#访问超时
　　data = html.read().decode('UTF-8')
except:
　　continue
```

　　代码中的处理办法比较暴力，直接就把不能访问的URL略过，这样并不知道不能访问的原因。其中有一种不能访问的原因值得深入研究，那就是服务器拒绝响应。现在大部分网站都禁止爬虫来爬，所以当爬虫与服务器交互时，服务器发现你是爬虫，就会拒绝你的所有请求（比如京东、淘宝，直接爬是不行的）。这时，我们需要把爬虫伪装成浏览器。

###**第四步：伪装成浏览器**
　　伪装成浏览器简单来说，就是在你的请求中加入头部信息，最重要的是User-Agent参数。什么是头部信息？浏览器可以告诉你。打开浏览器调试界面（我是Chrome，直接按F12就出现），选中顶层的Networks，然后在浏览器地址栏出入一个网址，回车，你会发现调试器中出现了好多请求，js啊，图片啊，css啊等等，随便点击一个请求，在右侧选择headers，headers的最后一项就是User-Agent。复制，然后以字典的形式存入headers中。

```
url = deque.popleft()
request = urllib.request.Request(url, headers = {
 'User-Agent':'Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.36 (KHTML, like Gecko)\
             Chrome/50.0.2657.3 Safari/537.36'
})

html = urllib.request.urlopen(request)
```

　　通过以上的构造方式，服务器就会认为你是浏览器而不会拒绝你的请求。接下来你可以对静态的HTML代码进行正则抓取了。如果正则表达式运用的好，就能很轻松的得到你想要的东西。但是如果想抓取动态的东西，还需要其他的知识啦。

###**附**
上我的抓取糗事百科笑话并保存的爬虫。

```
#包含的库
import re
import urllib
import urllib.request

#自定义的保存函数，由于需要不断的打开文件、写入文件操作，效率很低。
def saveData(data, savePath):
    f = open(savePath, 'a')
    try:
        f.write(data)
    except:
        pass
    f.close()
    
cnt = 1 
#匹配笑话的正则表达式
rule_joke = r'<div class="content">((?:.|[\r\n])*?)</div>'
savePath = 'E:\qb_jokes.txt'
url_header = 'http://www.qiushibaike.com/8hr/page/'
url_tail = 1

while url_tail < 30:
  　#由于糗事百科网页特殊性，直接改变页号即可实现访问不同网页
    url = url_header + str(url_tail)
    request = urllib.request.Request(url, headers = {
            'User-Agent':'Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.36 (KHTML, like Gecko)\
             Chrome/50.0.2657.3 Safari/537.36'
    })
    #尝试打开网页，读取信息
    try:
        html = urllib.request.urlopen(request)
        data = html.read()
    except:
        print('Parser Html Error!')
        continue
    #尝试匹配网页中的笑话
    jokes = re.compile(rule_joke)
    try:
        joke = jokes.findall(data.decode("UTF-8"))
    except:
        print("Find jokes error!\n")
        continue
    #将得到的笑话的字符串处理并存储
    for j in joke:
        print(str(cnt) + j + "\n")
        saveData(str(cnt) + j + "\n", save_path)
        cnt += 1
    #表示读取下一页
    url_tail += 1
```
读到的笑话截图
![这里写图片描述](http://img.blog.csdn.net/20160508215930655)


