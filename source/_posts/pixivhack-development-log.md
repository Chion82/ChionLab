---
title: PixivHack：记P站爬图脚本开发
date: 2016-01-20 01:28:14
tags: [python, crawler]
---

由于7月学校考试，预习功课有点忙，最近又忙着做外包，一直都忘了维护博客，所以现在补上上个月本来应该写的技术博文。

# 这是啥

跟生活在2.5次元的老司机混久了便跟着入了ACG坑（看到本站首页LL大法时您应该意识到了，博主是个死宅_(:з」∠)_ ）。P站找图是每一个ACG爱好者的必备技能，然而像我这种刚入坑不久的，收藏的画师才几个，再者没有钱买Premium，不能按人气选图，每次找图都要手动一页一页翻(╯‵□′)╯︵┻━┻ 于是我想用Py写个自动爬图脚本。这个脚本可以按你输入的关键词搜索作品，并根据Rating（评分次数，以此来判断作品人气）的最小值来筛选并自动下载，也可以手动指定画师ID列表，也是按照设定最小Rating的方法下载每个画师的图。目前支持下载插画、漫画和大图。

# 你为什么还不用！  
[GitHub链接](https://github.com/Chion82/PixivHack.git)  
使用方法详见README.md

# 实现过程
* 要从P站搜图首先要登录。我原来的设想是通过抓包直接用Py模拟登录，但是不出所料，P站的登录API参数都是经过加密的（貌似是基于RSA的），在不知加密算法的情况下无法实现。（虽然在另一个开源项目中我已经能够分析登录部分的JS加密逻辑，但在那之前我还是懒得审查JS）  
于是目前的解决方案是要求用户在浏览器登录进P站一次，通过浏览器debugger获取Cookies中的PHPSESSID的值并输入到脚本中，脚本向P站发出的每次http请求都要带上该Cookie，目的是让P站服务器认为我们已经登录。  
下面贴一行每次请求带上Cookie的代码  
``` python
search_result = self.__session.get('http://www.pixiv.net/search.php?word=' + urllib.quote(self.__keyword) + '&p=' + str(page), cookies={'PHPSESSID': self.__session_id})
```
**更好的方法：在requests的session中通过headers.update()方法在Header中设置Cookie，该session的每次请求都能自动带上该header，这样就不需要每次都在请求中加上cookies参数**

* 从HTML源码中提取有用数据：虽然Python中可以使用HTMLParser更灵活地分析HTML，但是由于不想在这个小项目上浪费太多时间，我直接用正则从HTML中匹配。这里贴一行匹配作品搜索列表的代码：  
``` python
result_list = re.findall(r'<a href="(/member_illust\.php\?mode=.*?&amp;illust_id=.*?)">', search_result.text)
```

* 自动脚本的流程是：获取作品搜索结果页面，从每个搜索结果分别进入作品首页，判断Rating是否高于设定的最小值（若低于则跳过该作品），判断作品类型（插画/漫画/大图）并根据不同的流程进入作品二级页面并获取原图的URL

* 绕过P站的防Bot机制：P站的原图不可直接下载。用户在浏览器中访问原图链接时，浏览器会自动加上Refer这个HTTP头，P站图片服务器会验证Refer是否合法。所以，脚本在访问原图链接并下载时，也需要在header中带上Refer。经过测试，Refer的值为作品首页或者作品二级页的URL。总之，在下载原图时带上当前页面（也就是HTML中能找到原图src的页面）的URL作为Refer就不会出问题。
``` python
download_result = self.__session.get(url, cookies={'PHPSESSID': self.__session_id}, headers={'Referer':referer})
```

* 连接失败处理：经过测试，在长时间连续爬图时很有可能会有一两次requests请求超时（不排除是天朝某墙的TCP RST所致，也有可能是requests2.0在同域下长时间保持单TCP连接使P站服务器拒绝所致），因此，requests每次发出http请求时，都应该用try...except捕获超时异常并重试。
``` python
try:
	page_result = self.__session.get(url, cookies={'PHPSESSID': self.__session_id}, headers={'Referer':referer})
except Exception:
	print('Connection failure. Retrying...')
	self.__enter_manga_big_page(url, referer, directory)
	return
```

* 统计画师总评分数：按关键词爬完一波图后，需要统计所爬的每个画师的ID、Rating等值，方便我们之后能收藏人气高的画师。
``` python
def __increment_author_ratings(self, author_id, increment, pixiv_id):
	for author in self.__author_ratings:
		if (author['author_id'] == author_id):
			if (pixiv_id in author['illust_id']):
				return
			author['total_ratings'] = author['total_ratings'] + increment
			author['illust_id'].append(pixiv_id)
			return
	self.__author_ratings.append({'author_id':author_id, 'total_ratings':increment, 'illust_id':[pixiv_id]})
```

* 用户交互：用argparse实现CLI参数传入分析。在本脚本中，通过"-a"或"--authorlist"参数指定存储了画师ID列表的JSON文件。
``` python
parser = argparse.ArgumentParser()
parser.add_argument('-a', '--authorlist', help='Crawl illustrations by author IDs. A JSON file containg the list of Pixiv member IDs is required.')
args = parser.parse_args()
```
