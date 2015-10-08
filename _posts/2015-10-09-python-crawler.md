---
layout: post
title: Python爬虫
category: other
---

# Python爬取微博数据

## 背景

以前用python搞爬虫，各种urllib，urllib2库搞得头都大了。今天用requests库来实现了一遍，发现思路比以前清晰了许多。

## 模拟登录

玩爬虫，有手机版的果断爬手机版（大家都懂的）。这次爬的是[微博手机版](http://login.weibo.cn)。

审核元素可以发现，该页面的Form有两部分数据需要post，一部分是静态的，一部分是动态的。（可以打开网页看看）

* 静态数据：mobile（用户名），remember，backURL，backTitle，tryCount以及submit
* 动态数据：登录url中有一个动态生成的数，input password的名称是动态生成的，以及post参数vk是动态生成的。

因此在传递参数时，静态数据可以直接传，动态数据通过先访问一次登录网页，从返回的html中解析出动态生成的数据即可，代码如下:
	
    def get_rand(self, url):
        r = requests.get(self.login_url)
        soup = BeautifulSoup(r.text, 'html.parser')
        rand = soup.form['action'][6: 15]
        passwd = soup.find_all('input', attrs = {"type": "password"})[0]['name']
        vk = soup.find_all('input', attrs = {'name': 'vk'})[0]['value']
        return rand, passwd, vk
        
接下来就要开始登录啦，登录无非是发送一个post请求，把各个参数传过去，获取cookie，以便之后爬去网页时可以不用再登录。同时注意设置一下http请求头，让爬虫行为更像浏览器。代码如下：
	

    def login(self):
        rand, passwd, vk = self.get_rand(self.login_url)
        
        self.headers = {
                'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; WOW64; rv:37.0) Gecko/20100101 Firefox/37.0',
                'Referer':'',
                'content-type': 'application/x-www-form-urlencoded'
                }
                
        self.datas = {
                'mobile': self.username,
                passwd: self.password,
                'remember':'on',
                'backURL': 'http%3A%2F%2Fweibo.cn%2F%3Ffrom%3Dhome',
                'backTitle': '%E6%89%8B%E6%9C%BA%E6%96%B0%E6%B5%AA%E7%BD%91',
                'tryCount': '',
                'vk': vk,
                'submit': '%E7%99%BB%E5%BD%95',
                }
        url = 'http://login.weibo.cn/login/?' + rand + '&backURL=http%3A%2F%2Fweibo.cn%2F&backTitle=%E5%BE%AE%E5%8D%9A&vt=4&revalid=2&ns=1'

        session = requests.Session()
        r = session.post(url, data = urllib.urlencode(self.datas), headers = self.headers)
        print 'login success...'
        return session
       
上面代码中，headers是http请求头，datas是需要post的数据。这里session对象可以跨请求保持某些参数，并且在同一个Session实例发出的所有请求间保持cookies，再也不用自己来设置cookies啦。

这里有一个问题，我把http请求头的`content-type`设为`application/json`，post data写成`json.dumps(self.datas)`，请求返回状态码200，但是cookies不正常，不知道什么原因。

## 爬取微博

登录完以后想怎么玩就怎么玩啦，不过要注意爬取间隔最好设长一点，免得遭来一些不必要的麻烦，比如帐号被封了，登录需要验证码了等等。

爬取到的页面用Beautifulsoup解析成各种你想要的东西即可。

下面是我爬取微博评论用户的例子：

	    def craw_comment_users(self, url, fileout):
        usernames = {}
        page = 1
        while True:
            crawl_url = '%s&page=%d' % (url, page)
            r = self.session.get(crawl_url)
            print 'crawling at page %d, url is: %s' % (page, crawl_url)
            soup = BeautifulSoup(r.text, 'html.parser')
            tag = soup.find_all('a', href = re.compile("/u/"))
            if len(tag) == 0:
                break
            for it in tag:
                usernames[it.get_text().encode('utf-8')] = 1
            page += 1
            time.sleep(10)

        fout = open(fileout, 'w')
        for k, v in usernames.iteritems():
            fout.write(k + '\n')

## Ending

GL&HF     
       
{% include references.md %}
