+++
title = "Tornado Httpclient简单异步爬虫"
date = 2018-02-21T10:26:49+08:00
tags = ["tornado", "爬虫"]
categories = ['web']
keywords = ['tornado','爬虫', 'asyncio']
draft = false
markup = "mmark"
+++
最近要写篇论文，需要给abstract画个图。在同学那里看到他画的一张图感觉很不错，一问原来是用3Ds marks
画的。他还给推荐了一个微信公众号，专为科研3D绘图发布教程。为了方便随时随地能够看教程，就生出了
爬下来的想法。
<!--more-->

也用 python 几年时间了， python 做爬虫似乎比较流行，实际就 python 的特性来看，python 对于爬虫方面
的应用确实适用：丰富的库可以减少大量重复工作，可以用较少的代码实现功能，网络爬虫的性能瓶颈往往
在于网络IO而非 python代码的执行速度。

用浏览器打开该公众号的网址，发现可以直接查看，说明该公众号内容并未做验证，不需要cookie，不需要
通过微信，直接用tornado的httpclient就可以爬取，是一个最基础的爬取过程。

爬虫爬取内容的过程：
获取网页内容->理解网页内容->获取需要的内容

### 1. 获取网页内容
就是通过url获取入口页面的html内容。

### 2. 理解网页内容
这里就各显神通了，对于内容简单的 html 内容，可以直接手工处理。对于稍微复杂一点，不涉及太多或者复杂
 js 运行的 html 网页，可以用能够理解 html 各标签的库处理，比如本文使用的 beautifulsoup4。对于
更复杂的网页，可以使用 selenium 或者类似的手段，直接通过 headless 的浏览器对网页内容进行处理。
然后就可以通过 API 调用高效的进行后续的处理。

### 3. 获取所需的内容
根据`2`中的处理后的对象，可以轻易获取想要的内容。比如`img`标签的`src`，`a`标签的`href`。然后
获取相应内容即可。

## 公众号教程页面爬取
### 1. 公众号内容组织
该公众号的内容组织十分简单，一个总览页，罗列了绝大部分的（最新的可能没有）教程，每个教程用一个代表
最终成果的图片，点击图片可以跳转到相应的教程页面。

### 2. 需要的处理
爬这个教程需要将首页及图片爬过来，然后将每个图片中的`a`标签的对应的详细教程页面爬取过来。将`a`
标签的`href`替换为本地的网址，以保存文件夹作为根目录。将所有的图片保存到本地的`images`文件夹
中，将所有对应的`img`标签的`src`替换程本地以根目录作为网站根的地址。对于教程详细页面的文件名，
需要将`titile`提取出来作为文件名保存，以`.html`作为后缀。所有的图片也进行重命名，将`url`的
地址进行`base32`编码。

### 3. tornado-httpclient + asyncio异步获取内容
tornado中包含有一个AsyncHTTPClient实现，可以使用它非常方面的实现内容的异步非阻塞获取，加快内
容获取。tornado 包含对asyncio的支持，在所有的tornado相关代码运行之前，需要将asyncio的loop
设置为tornado的loop，并且编写代码时，需要注意到tornado的`future`和asyncio的`future`不同，
可以使用`tornado.platform.asyncio`中的`to_asyncio_future`和`to_tornado_future`相互
转换。下面是一个简单示例：

    from tornado.platform.asyncio import AsyncIOMainLoop, to_asyncio_future

    AsyncIOMainLoop().install()  # 使用asyncio的loop作为tornaod的loop
    to_asyncio_future(TORNADO_FUTURE)  # 将tornado的future转化为asyncio的future

可以用asyncio.gather将多个tornado的`AsyncHTTPClient.fetch`的`future`对象集合到一起，
并行获取，如果采用`循环 + await`的方式，则是序列化运行的，无法起到加速的目的。

该公众号的`img`标签和`a`标签的`src`或者`href`都是藏在其他属性当中，通过页面加载过程中的`js`
代码将其设置的，因此代码中需要作相应的处理。这是非常简单的`获取 + 设置`属性的过程，直接用beautifulsoup4
载入html内容并修改就是，无需使用浏览器加载或`js`代码运行器。

具体代码：
```python
    import re
    from hashlib import md5


    pattern = re.compile('\w+')
    md5generator = md5()


    def validFileName(string):
        pattern = re.compile('[\w]', flags=re.UNICODE)

        def validator(s):
            if(pattern.match(s) is not None):
                return True
            return False
        result = ""
        newfilter = filter(validator, string)
        for i in newfilter:
            result += i
        return result


    async def getUrls():
        client = httpclient.HTTPClient()
        response = client.fetch(
                "https://mp.weixin.qq.com/s?__biz=MzIyNjM1MzQ1OA==&mid=100001567&"
                "idx=1&sn=eb1c1bf42e819e3bd0cfa8ff820e868b&pass_ticket="
                "7IZjYPWAyGSWFIcxM"
                "c3oTa1zQ56eGL%2F0hiuQkQUZncSHpTZxfQV5tAg0%2Bk8NERI0")
        soup = BeautifulSoup(response.body, "html5lib")
        sections = soup.find_all('section', attrs={'data-tools': '135编辑器'})
        page_urls = list()
        img_urls = list()
        atags = list()
        for section in sections:
            for atag in section.find_all('a'):
                atags.append(atag)
                page_urls.append(atag['href'])
                img_urls.append(atag.img['data-src'])

        coroutines = [getPage(url) for url in page_urls]
        pages = await asyncio.gather(*coroutines)

        coroutines = [getPicsAndChangeSrcSavePage(page)
                    for page in pages]
        page_names = list()
        for page_coroutine in coroutines:
            page_names.append(await page_coroutine)

        img_pathes = await fetchImages(img_urls)

        for index in range(len(atags)):
            atags[index]['href'] = page_names[index]
            atags[index].img['src'] = img_pathes[index]

        # write index.html file
        with open('index.html', 'w') as index_page:
            index_page.write(str(soup))


    async def getPage(url):
        client = httpclient.AsyncHTTPClient()
        print(url)
        response = to_asyncio_future(client.fetch(url))
        result = await response
        return result.body


    async def getPicsAndChangeSrcSavePage(htmlpage):
        soup = BeautifulSoup(htmlpage, "html5lib")
        page_name = soup.head.title.get_text()
        page_name = validFileName(page_name) + '.html'
        imgs = soup.find_all('img', attrs={'data-src': pattern})
        img_urls = list()
        for img in imgs:
            url = img['data-src']
            img_urls.append(url)
        file_pathes = await fetchImages(img_urls)
        # replace all img tag's src
        for index in range(len(imgs)):
            imgs[index]['src'] = file_pathes[index]
        # save page to file
        with open(page_name, 'w') as page_file:
            page_file.write(str(soup))
        return page_name


    async def fetchImages(urls):
        client = httpclient.AsyncHTTPClient()
        responses = await asyncio.gather(
                *[to_asyncio_future(client.fetch(url)) for url in urls])
        if(not os.path.exists('images')):
            os.mkdir('images')

        file_pathes = list()
        for response in responses:
            name = response.effective_url
            md5generator.update(bytes(name, 'utf-8'))
            encoded_name = md5generator.hexdigest()
            # get extension type in headers such as image/jpeg
            ext = response.headers['Content-Type'].split('/')[1]
            file_name = encoded_name + '.' + ext
            file_path = 'images' + os.sep + file_name
            with open(file_path, 'wb') as output:
                output.write(response.body)
            file_pathes.append(file_path)
        return file_pathes


    if __name__ == '__main__':
        AsyncIOMainLoop().install()
        loop = asyncio.get_event_loop()
        loop.run_until_complete(getUrls())
```

更新记录：   
    2018-02-23： 添加代码
