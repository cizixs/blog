---
layout: post
title: "HTTP 断点下载功能实现"
excerpt: "在网络不好的情况下，从网站上下载大文件，经常会出现下载到一半就断掉的情况。利用断点继续下载可以有效节约网络和效率，这篇文章我们就会自己写一个简单的脚本来做这个。"
categories: blog
tags: [http, stream]
comments: true
share: true
---

## 缘起

最近在家要下载一个比较大的镜像文件，因为网络太差，每次都是下载到中间就停了，文件就下载失败。我用的是 chrome 自带的下载功能，就这样重试了4-5 次，每次都以失败告终。可恨的是，有时候下载到 80% - 90%，失败了还要从头重来。

后来我就想到 wget 命令行，上网搜了一下，发现 wget 是自带断点下载功能的。也就是说，如果中间连接断开，可以直接从已经下载好的地方继续开始。然后，直接使用 wget 搞定了任务。

如果之前已经有了一个下载部分的文件，也可以使用 wget 的 `-c` 命令，来继续下载。 可以参考 [nixCraft 上一篇文章 Wget: Resume Broken Download](http://www.cyberciti.biz/tips/wget-resume-broken-download.html)

## 琢磨
按说完事之后，我也就该放心啦。文件也下载完了，你还有啥可惦记的呢？可是我心里一直有个疙瘩：HTTP 协议不是无状态的吗？重发的请求，是怎么告诉服务器端，我只要某一段数据的呢？应用程序有事怎么把不同的数据接起来的呢？

然后，我就继续搜索，发现 HTTP 断点续传秘密所在：`Range` 头部字段。

## 解释
断点下载，通俗一点说，就是客户端告诉服务器端：我已经下载了前面长度为 n 的内容， 请从 n+1 个数据给我传过来，不用从 0 开始重新传。

在 `HTTP 1.1` 版本中，有个对应的实体头部做这件事情，那就是 `Range`。`Range` 指定要请求实体（entity）的范围，和多数的程序规范一样，它也是从 0 开始计数的。比如前面已经下载了 500 bytes 的内容，要请求 500 bytes 以后的内容，只要在 HTTP 请求的头部加上 `Range: bytes=500-` 就可以啦。

`Range` 有几种不同的方式来限定范围：

1. `500-900`：指定开始到结束这一段的长度，记住 `Range` 是从 0 计数 的，所以这个是要求服务器从 501 字节开始传，一直到 901 字节结束。这个一般用来特别大的文件**分片传输**，比如视频。
2. `500-`：从 501 bytes 之后开始传，一直传到最后。这个就比较适合用于断点续传，客户端已经下载了 500 字节的内容，麻烦把后面的传过来
3. `-500`：这个需要注意啦，**如果范围没有指定开始位置，就是要服务器端传递倒数 500 字节的内容。而不是从 0 开始的 500 字节。**。
4. `500-900, 1000-2000`：也可以同时指定多个范围的内容，这种情况不是很常见

客户端发出这样的请求，也要服务器端响应和支持这个功能。前面也提到过，这个实体头部是 HTTP 1.1 版本才添加的，所以有些 HTTP 服务端使用比较老的版本可能不支持。

如果服务端支持的话，那么这个响应里也要带一个 `content-range` 的字段。比如客户端使用的是 `Range: bytes=1024-`，那么对应的服务端返回头部会包括类似 `content-range: bytes 1024-439714/439715` 的内容。这个意思就是说，我知道啦，我会从只传递 1024 - 439714 范围的内容，而整个文件的大小是 439715 字节。而且，这个时候响应的 status code 一定是 `206`，关于 `206` 的解释可以查看 [w3 官网上的说明。
](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)。
如果真是这样也就皆大欢喜啦，但是可能出现两种异常情况：

### 不支持

服务端返回的响应头部没有 `content-range` 字段，表明服务端忽略了请求里 `range` 字段。可能是因为服务端不支持这个功能，那么也没办法，只能从头开始下载。 注：目前大多数 HTTP 都支持这个功能，所以这个情况也很少见

### 支持是支持，但还是给我从头传

服务端返回的响应头部包含了 `content-range`，但还是从 0 开始重新下载的。这个就是服务端实现比较严格，考虑到了这个文件可能会被修改的情况。

什么意思？考虑一下这个情况：昨天我下载某个文件，下了一半，今天使用 `wget -c` 接着下。但问题是：中间某个时间点，这个文件被更新了！URL 没变，但是文件内容变啦。如果我接着往后下，然后把之前的内容和新下的内容和在一起，很可能就不是一个正常的文件啦，没有办法打开。这个情况比从头开始下载还要糟糕：辛辛苦苦下载完，结果是不能用的。

如果服务端要下载的文件，或者其他资源是可变的。就要有一个办法来标识文件或者资源的唯一性，就是说我之前下载的到底是哪个版本的，和现在版本是不是一致的。HTTP 协议里有两个办法可以来做这件事，当然也就是两个头部字段：`ETag` 和 `Last-Modified`，都定义在 RFC2616。

+ `ETag`： 你可以把 ETag 比作文件的 MD5，用来唯一标识一个文件，它是一串字符
+ `Last-Modified`：望名思意，就是这个文件上次修改的时间

如果服务端比较严格，会检查你这次要下载的文件和上次下载的有没有变化，那么在第一次下载的时候，它一定会提供 `ETag` 或者 `Last-Modified` 两个字段中的至少一个。那么断点下载的请求，只要把任意一个放到 `If-Range` 字段里传过去就可以啦(`If-Range` 只能和 `Range` 一起使用，否则会被服务端忽略)。

如果两次下载的是同一个文件，就会返回 206，从后开始续传；否则就会返回 200，表示文件变了，要重头开始下载。

如果客户端发送的 range 范围有错误，会返回 416，并且 Content-Range 字段形如 `bytes */439715`，表示提供的范围有误，文件总大小是 `439715`。

## 实现

知道上面的原理，实现一个自己的断点下载程序其实很简单。

这段代码实现的是 `wget -c` 的功能，可以实现的效果是：

1. 第一次下载文件会创建新的文件并开始下载
2. 第二次下载，如果发现已经有文件存在，就会发送 `Range` 请求，从已经下载的重新开始
3. 如果服务器端支持 `Content-Range`，就接着下载
4. 如果服务器端不支持，或者又重头发送（我们没有发送 `If-Range` 头部），就删除原来的文件，重新开始

如果要使用 `If-Range` 就要把第一次请求的数据存起来，为了简单起见，我们不会实现这个功能。

    import os
    import sys
    
    import requests
    
    
    def file_size(filename):
        return os.stat(filename).st_size

    
    def download(url, chunk_size=65535):
        downloaded = 0  # How many data already downloaded.
        filename = url.split('/')[-1]  # Use the last part of url as filename
    
        if os.path.isfile(filename):
            downloaded = file_size(filename)
            print("File already exists. Send resume request after {} bytes".format(
                downloaded))
    
        # Update request header to add `Range`
        headers = {}
        if downloaded:
            headers['Range'] = 'bytes={}-'.format(downloaded)
    
        res = requests.get(url, headers=headers, stream=True, timeout=15)
       
        mode = 'w+'
        content_len = int(res.headers.get('content-length'))
        print("{} bytes to download.".format(content_len))
        # Check if server supports range feature, and works as expected.
        if res.status_code == 206:
            # Content range is in format `bytes 327675-43968289/43968290`, check
            # if it starts from where we requested.
            content_range = res.headers.get('content-range')
            # If file is already downloaded, it will reutrn `bytes */43968290`.
                
            if content_range and \
                    int(content_range.split(' ')[-1].split('-')[0]) == downloaded:
                mode = 'a+'
        if res.status_code == 416:
                print("File download already complete.")
                return

        with open(filename, mode) as fd:
            for chunk in res.iter_content(chunk_size):
                fd.write(chunk)
                downloaded += len(chunk)
                print("{} bytes downloaded.".format(downloaded))
    
        print("Download complete.")
    
    
    if __name__ == '__main__':
        url = 'http://dldir1.qq.com/qqfile/QQforMac/QQ_V4.0.4.dmg'
        url = sys.argv[1] if len(sys.argv) == 2 else url
        download(url)

我们使用了 `requests` 库来实现 HTTP 请求，代码实现起来也比较简单，需要说明的地方都已经添加了注释。

稍微修改一下，就能封装成一个还不错的下载工具。

## 参考资料

+ [`how to resume an interrupted download` on StackOverflow](http://stackoverflow.com/questions/3428102/how-to-resume-an-interrupted-download-part-2)