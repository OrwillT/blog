---
title: "利用 Backblaze 和 Cloudflare 构建私有内容分发网络"
subtitle: "Private CDN build with Backblaze and Cloudflare"
date: 2020-07-13T19:35:48+08:00
draft: false
author: ""
authorLink: ""
description: "博客重生三部曲之一，利用 Backblaze 和 Cloudflare 构建免费的私有内容分发网络。"

tags:
  - S3
  - Backblaze
  - Cloudflare
categories:
  - 脑机

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: "https://assets.anislet.dev/images/blog/posts/private-cdn-build/cover.png"
featuredImagePreview: "https://assets.anislet.dev/images/blog/posts/private-cdn-build/cover.png"

toc:
  enable: true
math:
  enable: false
lightgallery: true
license: ""
---

<!--more-->

书接[上回](/posts/hello-world/)！

## 名词解析

本着[对新人友好]^(多水点字数)的原则，我们先来简单了解一下以下两家公司。

### Backblaze

{{< image src="https://assets.anislet.dev/images/blog/posts/private-cdn-build/backblaze-site.png" alt="Backblaze Website" caption="**Backblaze**" >}}

[Backblaze](https://www.backblaze.com/) 是世界上最大的储存服务商之一，每个季度都会有的硬盘可靠性报告就是他家出品的。截止2020年第一季度，Backblaze 一共有13万余块硬盘在役。

Backblaze 原本只有备份这一个业务，但在2015年的时候推出了 B2 Could Storage，使用自有的 B2 API 且 10 GB 存储空间免费。2018年 Backblaze 加入了 Cloudflare 主导的[带宽联盟]^(Bandwidth Alliance)，从此 Backblaze 从 Cloudflare 的 CDN 走的流量和 API 也全都是免费的了。

2020年5月4日，Backblaze 发布了 B2 Cloud Storage 的 [S3 Compatible API](https://www.backblaze.com/blog/backblaze-b2-s3-compatible-api/)。真的是赶得早不如赶得巧，刚好赶上了我对服务器架构大动干戈的时候。于是 Backblaze 就替代了 Ceph 服务器所提供的 Objective Storage 功能。

而且人家还很便宜。

### Cloudflare

{{< image src="https://assets.anislet.dev/images/blog/posts/private-cdn-build/cloudflare-site.png" alt="Cloudflare Website" caption="**Cloudflare**" >}}

[Cloudflare](https://www.cloudflare.com) 是世界上最大的互联网公司之一，主营基于反向代理的内容分发网络和分布式域名解析服务。他虽然采用的是免费+增值付费的模式，但免费内容就已经充实到令人颤抖的地步了。

就真没啥可说的，就很牛逼，互联网企业中真正的慈善组织，拿各种奖拿到手抽筋，近几年各种创新功能还基本都有免费的方案。没用过的话那简直就是人生的一大憾事。

那何乐而不为呢？

## 部署

注册账号这种事情大家都会我就不具体表述了。直接上代码吧。

### index.js

``` javascript
import { AwsClient } from 'aws4fetch'

const HTML = "Some custom HTML code"

const aws = new AwsClient({
    "accessKeyId": AWS_ACCESS_KEY_ID,
    "secretAccessKey": AWS_ACCESS_KEY_SECRET,
    "region": AWS_S3_REGION
});

addEventListener('fetch', function(event) {
    event.respondWith(handleRequest(event.request))
});

async function handleRequest(request) {
    var url = new URL(request.url);    
    url.hostname = AWS_S3_BUCKET + '.' + AWS_S3_ENDPOINT;

    var signedRequest = await aws.sign(url);

    if (url.pathname === '/') {
        return new Response(HTML, { headers: { 'Content-Type': 'text/html; charset=utf-8' }});
    } else {
        return await fetch(signedRequest, { "cf": { "cacheEverything": true } });
    }
}
```

Worker 的代码基于这篇[文章](https://blog.cloudflare.com/backblaze-b2-and-the-s3-compatible-api-on-cloudflare/)提供的[示例](https://github.com/obezuk/worker-signed-s3-template)修改，使用 [aws4fetch](https://github.com/mhart/aws4fetch) 库作为 S3 客户端，使用 [Wrangler](https://developers.cloudflare.com/workers/quickstart#installing-the-cli) 进行部署管理。

最终构建出的 Worker 脚本仅有 4 KiB，加上 HTML 部分也才 5 KiB 而已。

### Variables

Parameter | Default | Examples
---|:---:|:---:
AWS_ACCESS_KEY_ID | `nil` |
AWS_ACCESS_KEY_SECRET | `nil` |
AWS_S3_REGION | `nil` | us-west-1 
AWS_S3_BUCKET | `nil` | worker-cdn-bucket 
AWS_S3_ENDPOINT | `nil` | s3.us-west-1.amazonaws.com 

鉴于安全性考虑，实际部署于生产环境时应将 `AWS_ACESS_KEY_SECRET` 做为 [Secret Environment Variables](https://developers.cloudflare.com/workers/tooling/wrangler/secrets/) 处理。

### index.html

默认情况下访问 `/` 时会直接返回包含桶名和桶内所有文件地址的 XML 文件，为了美观也为了防止暴露桶名导致恶意 API 访问，我截获了所有发向 `/` 的请求并返回了一段自定义的 HTML 代码。

```html
<!DOCTYPE html>
<html>
  
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <title>Anislet&#39;s Assets CDN</title>
    <style>
      * {margin:0; padding:0;}
      .main {width:80%; margin:50px auto; box-sizing:border-box; padding:50px;text-align: center;}
      .main p {display:block; font-size:24px; font-weight:300; color:#000000}
      .main a {color:#000000;}
      .footer p {font-size:12px;}
    </style>
  </head>
  
  <body>
    <div class="main">
      <svg t="1593929074047" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="2730" width="128" height="128" xmlns:xlink="http://www.w3.org/1999/xlink"><defs><style type="text/css"></style></defs><path d="M523.2 105.7c-218.4 0-395.5 177.1-395.5 395.5s177.1 395.5 395.5 395.5 395.5-177.1 395.5-395.5c0-218.5-177.1-395.5-395.5-395.5z m-87.9 45.1c-36.2 39.7-65.4 103.3-83 180.6-57.2-9.5-103.8-21.4-130.6-29 49-74.3 124.8-129.4 213.6-151.6z m-80.7 669.8C240.1 760 162.1 639.7 162.1 501.1c0-61.3 15.3-119 42.2-169.5 19.1 5.9 71.8 21.1 141.3 33-4.8 27-8.1 55.5-10 84.9H370c1.7-27.7 4.7-54.3 9-79.6 44.2 6.3 93.5 10.8 144.2 10.8 29.4 0 58.3-1.5 86-4v-34.4c-27.7 2.5-56.6 4-86 4-48.3 0-95.2-4.1-137.8-9.9C411.1 219.8 463.1 140 523.2 140c49.3 0 93.2 53.8 121.5 137.6h34.5c-17.4-53-40.8-96.7-68.2-126.8 88.8 22.2 164.6 77.3 213.7 151.6-21.1 6-54.2 14.6-95.2 22.6v34.4c55.2-10.8 96.2-22.7 112.6-27.8 1.4 2.6 2.7 5.2 4.1 7.9-27.7 16.4-78.7 47.9-136.9 90.1-0.9-10.7-2.1-21.3-3.4-31.7h-34.3c2.3 18.1 4 36.8 5.1 56-38.3 29.3-78.3 62.6-115.8 98.8-67.9 65.6-119.8 133.1-154.8 184.5-15.8-42.5-27.3-93.7-33.2-150.1h-34.3c7.4 69.7 23.6 132.4 46 182.9-15.7 24.7-25.7 42.7-30 50.6z m32.5 15.1c4.7-7.7 10.3-17 17-27.4 9.7 16.5 20.2 31 31.3 43.2-16.6-4.2-32.8-9.5-48.3-15.8z m136.1 26.5c-37.8 0-72.5-31.7-99.4-84.3 38.9-58 98-137.1 171.6-208.2 27.9-26.9 55.8-51.4 82.5-73.3v4.7c0 199.5-69.2 361.1-154.7 361.1z m87.8-10.7c60.2-66.1 101.3-198.2 101.3-350.3 0-10.6-0.2-21.2-0.6-31.6 68.3-52.7 125-86.8 147.8-99.9 15.9 40.8 24.7 85.1 24.7 131.5 0.1 169.1-116.2 311-273.2 350.3z" fill="" p-id="2731"></path><path d="M402.8 466.8H299.7V570h103.2V466.8z m-34.4 68.7H334v-34.4h34.4v34.4zM712.3 294.8h-86v86h86v-86z m-34.4 51.6h-17.2v-17.2h17.2v17.2z" fill="" p-id="2732"></path><path d="M523.2 71.3c-237.4 0-429.9 192.5-429.9 429.9C93.3 738.6 285.8 931 523.2 931s429.9-192.5 429.9-429.9c0-237.4-192.5-429.8-429.9-429.8z m0 842.5c-227.9 0-412.7-184.8-412.7-412.7 0-227.9 184.8-412.7 412.7-412.7 227.9 0 412.7 184.8 412.7 412.7 0 228-184.8 412.7-412.7 412.7z" fill="" p-id="2733"></path></svg>
      <h1>Anislet&#39;s Assets CDN</h1>
      <br>
      <hr>
      <br>
      <p>This is Anislet Studio&#39;s private Content Distribution Network</p>
      <p>Powered by Backblaze and Cloudflare</p>
      <p>If you have any problem, mail to
        <a href="mailto:orwill@anislet.dev">Orwill Q. Song</a></p>
      <br>
      <hr>
      <br>
      <div class="footer">
        <p>© 2020 Anislet Studio. All rights reserved.</p>
      </div>
    </div>
  </body>

</html>
```

仅供参考。没用什么高级的东西都是很基本的，还把一个 SVG 给写死了进去，实际效果还算是赏心悦目。后面可能会考虑给他做一些其他的加强。

放在这儿的代码为了好读给格式化了，实际在应用中直接删减成一行用了。不得不感叹这些个标记语言是真方便，改成啥都能正常渲染。

## 问题

1. 没有办法正确处理中文文件名。但实际上我也没打算往里面塞中文文件名，所以我也没打算修。

2. Cloudflare 没有办法缓存任何内容，据 Cloudflare 社区反馈这是由于 Worker 在缓存之后处理所导致的，尝试了多种方案但均以失败告终，似乎不是从用户方面能够解决的问题。

3. 不支持 Path style，也不打算支持。

4. 免费的 Worker 请求次数每天是有限的，没有 CORS 的话很难阻止恶意访问。这个是真正需要解决的问题[^1]。

[^1]: 但说句不好听的，哪怕解决了多刷 Blog 页面也能达到同样的效果，但对个人来说终究是问题不大也够用了。