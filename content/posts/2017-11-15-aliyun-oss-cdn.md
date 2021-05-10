---
title: 使用阿里云OSS + CDN遇到的问题及解决方式
author: haohongfan
type: post
date: 2017-11-15T12:05:39+00:00
categories:
  - ops

---
### 一. OSS 配置 CDN加速 
CDN(Content Delivery Network)是将源站内容分发至最接近用户的节点，使用户可就近取得所需内容，提高用户访问的响应速度和成功率. CDN系统能够实时地根据网络流量和各节点的连接、负载状况以及到用户的距离和响应时间等综合信息将用户的请求重新导向离用户最近的服务节点上

优点: 解决网络拥挤. 提高用户下载速度; 保护源服务器. 无论渗透攻击,还是DDos攻击, 攻击的都是CDN节点.


1. 你需要一个已经备案的域名, 否则是设置不上的
2. 进入OSS 管理控制台 (需要使用主账号), 点击需要配置CDN加速的Bucket, 单击域名管理页签, 单击绑定用户域名(不在阿里云解析的域名, 不需要选择`自动添加 CNAME`)
3. `用户域名`填入你要绑定的域名; 选择`阿里云 CDN 加速`.  点击提交即可
![](http://images.haohongfan.com/oss-cdn.png?imageView2/2/w/800/h/500)
4. 然后选择`CDN管理控制台`, 点击域名管理, 点击`你刚才添加的域名`
5. 由于我们域名解析没有使用阿里云的, 所以需要额外一步, 将这个域名的`cname`到域名解析中.
![](http://images.haohongfan.com/cname.png?imageView2/2/w/800/h/500)

经过这样的配置之后, 你就可以通过CDN访问到Bucket内容上. 

如: 
私有Bucket的文件URL: 
```
http://dev.oss-cn-beijing.aliyuncs.com/111/composer.json?Expires=1510751768&OSSAccessKeyId=TMP.AQHdtru_W-NPJvuyWoWnMIzWzLrZq2gDOSbda_-Bc9HMFcmLMFWrSkeDUC8eMC4CFQCE-ERdOvcmBw1-zDtvU-ZrDrS0YQIVALnZ49WNhS84cNjogplHVKExsIpz&Signature=kYEs8RouE%2FKJTs2%2F1jMFaj8sYLc%3D
```

配置的域名为: `test.cn`

那么通过CDN访问这个文件的方式是:
```
http://test.cn/111/composer.json?Expires=1510751768&OSSAccessKeyId=TMP.AQHdtru_W-NPJvuyWoWnMIzWzLrZq2gDOSbda_-Bc9HMFcmLMFWrSkeDUC8eMC4CFQCE-ERdOvcmBw1-zDtvU-ZrDrS0YQIVALnZ49WNhS84cNjogplHVKExsIpz&Signature=kYEs8RouE%2FKJTs2%2F1jMFaj8sYLc%3D
```

详细配置请参考[官方文档](https://help.aliyun.com/document_detail/31902.html?spm=5176.8466035.bucket-domain-name.1.27656022shGzEb)

### 二. CDN 配置 HTTPS
1. 选择`CDN管理控制台`, 点击域名管理, 点击`点击你添加HTTPS的域名`
2. 选择`HTTPS设置`, 点击`修改配置`, 添加证书, 即可开启改`HTTPS`
3. 强制跳转: 默认: http->http, https->https. 
![](http://images.haohongfan.com/https.png?imageView2/2/w/800/h/500)

这里遇到一个问题: 
![](http://images.haohongfan.com/https_error.png?imageView2/2/w/800/h/500)
![](http://images.haohongfan.com/https_cname.png?imageView2/2/w/800/h/500)

如果没有把`CDN的域名`的`CNAME`添加到这个域名的`CNAME`解析上, 那么就会出现设置HTTPS不成功.就会出现上面的情况

### 三. CDN 配置 缓存
CDN缓存配置就很简单了, 对于静态资源缓存的时间尽量长些

需要注意的是: 
CDN的缓存不会自动刷新, 需要手动刷新. 假如你更新某个文件的资源, 但是文件的名字没有改变, 那么需要手动刷新CDN, 不然客户端是访问不到更新的资源的. 

![](http://images.haohongfan.com/cache.png?imageView2/2/w/700/h/500)

### 四. CDN 使用过程中遇到的问题
1.使用CDN访问OSS内容

经过`OSS 配置 CDN加速`后就可以访问OSS上的内容了. 但是有个问题
1. 对于公开Bucket来说, 其实没有什么影响.
2. 对于私有Bucket来说, 通过CDN访问OSS的文件仍然需要私有Bucket的签名.

如这样一个URL
```
http://ling-pro.oss-cn-beijing.aliyuncs.com/analysis/1.zip?Expires=1510753112&OSSAccessKeyId=TMP.AQHdtru_W-NPJvuyWoWnMIzWzLrZq2gDOSbda_-Bc9HMFcmLMFWrSkeDUC8eMC4CFQCE-ERdOvcmBw1-zDtvU-ZrDrS0YQIVALnZ49WNhS84cNjogplHVKExsIpz&Signature=vMHILFGRV%2BsmjyYn2mRRgUNJkeg%3D
```
我配置的CDN域名为:`test.cn`

我们不能通过`test.cn/lanalysis/1.zip`访问到这个资源, 仍然需要签名才可以.
```
http://test.cn/analysis/1.zip?Expires=1510753112&OSSAccessKeyId=TMP.AQHdtru_W-NPJvuyWoWnMIzWzLrZq2gDOSbda_-Bc9HMFcmLMFWrSkeDUC8eMC4CFQCE-ERdOvcmBw1-zDtvU-ZrDrS0YQIVALnZ49WNhS84cNjogplHVKExsIpz&Signature=vMHILFGRV%2BsmjyYn2mRRgUNJkeg%3D
```

如果你的程序是这么用CDN的, 那么基本上你的CDN只是把流量接进CDN了, 但是CDN完全没有起上效果. 
也就是说CDN在计费, 却没有产生效益.

还有一个问题需要引起注意:
使用URL带签名时, 是需要在每次请求的时候都去阿里云获取这样一个签名, 当网络中流量比较大的时候, 这件事情是值得优化的地方(虽说阿里云比较牛逼, 但是这不是我们设计这个功能的方式)

2.开启CDN鉴权

>URL鉴权功能旨在保护用户站点的内容资源不被非法站点下载盗用. 用户使用加密后的 URL 向加速节点发起请求，加速节点对加密 URL 中的权限信息进行验证以判断请求的合法性，对合法请求给予正常响应，拒绝非法请求，从而有效保护CDN客户站点资源。

上面这段是官方的说法, 看起来挺莫名其妙的. 简单来说, 开启CDN鉴权URL后, 通过CDN访问资源的时候, 需要通过CDN提供的三种鉴权方式(A, B, C)的其中一种算出一个带鉴权信息的URL, 来保证资源安全.

其中三种鉴权方式的详细信息: [详细文档](https://help.aliyun.com/document_detail/27135.html?spm=5176.doc27125.2.12.vzBC5A)

其中, `鉴权加密KEY`是需要保密的, 也是需要定期更换的. 这个`KEY`计算CDN鉴权URL需要的加密字段.

这里使用鉴权A的方式说下如何计算这个URL的:
1. 文件在位置: oss的dev bucket上(dev/111/composer.json)
```
http://dev.oss-cn-beijing.aliyuncs.com/111/composer.json?Expires=1510801016&OSSAccessKeyId=TMP.AQHgrFafuBeu4Jxpn7p8Zn6KbQR71DaxHxHHxZXnvVoKLHGE2IPPcLyvxDBfMC4CFQCGqYgJjFp8aIoqfbs4RNbtpGgugQIVAJmy4UIzI92bCOLbAfilBBkxPO4B&Signature=2KRA6asAm4fs5G8OTMaSVQJgF8A%3D
```

2. 鉴权A URL格式
`http://DomainName/Filename?auth_key=timestamp-rand-uid-md5hash`

3. DomianName
CDN的域名: `test.cn`

4. Filename
OSS Bucket中改文件的路径: `111/composer.json`

5. auth_key
公式:
```
sstring = "URI-Timestamp-rand-uid-PrivateKey" 
HashValue = md5sum(sstring)
```
| 字段         | 解释                            |
|------------|-------------------------------|
| URI        | 即Filename `111/composer.json` |
| Timestamp  | 过期时间, 如1800秒                  |
| rand       | 0                             |
| uid        | 0                             |
| PrivateKey | 即`鉴权加密KEY`                    |

**生成的URL**:
```
md5hash = md5("111/composer.json-1800-0-0-abcdefg") = `5095b8fa1f2c15e4bd1bd999b0834ace`
url = "http://test.cn/111/composer.json?auth_key=1800-0-0-5095b8fa1f2c15e4bd1bd999b0834ace"
```

![](http://images.haohongfan.com/auth.png?imageView2/2/w/700/h/500)

计算鉴权方式的三种方式的[示例代码](https://help.aliyun.com/document_detail/27277.html?spm=5176.doc27135.2.4.blEp4l)

3.CDN 开启私有Bucket回源

> 是否开启了CDN鉴权, 就可以不需要OSS的签名, 就可以通过CDN直接访问OSS的资源了呢?

答案是否定的
如果用`http://test.cn/111/composer.json?auth_key=1800-0-0-5095b8fa1f2c15e4bd1bd999b0834ace`访问`111/composer.json`, 会提示下面错误:
> 403 Forbidden
You don't have permission to access the URL on this server. 

这需要开启CDN 私有Bucket回源功能, 不然仍然需要使用签名来访问资源. [详细资料](https://help.aliyun.com/document_detail/57653.html?spm=5176.8232292.domaindetail.23.5dd5c833ftTkfO)

配置:
![](http://images.haohongfan.com/huiyuan1.png?imageView2/2/w/800/h/500)
![](http://images.haohongfan.com/huiyuan.png?imageView2/2/w/800/h/500)

### 五. 私有Bucket使用CDN加速后是否支持断点续传?
![](http://images.haohongfan.com/duandian.png?imageView2/2/w/800/h/500)

从测试结果来看, 是支持的

### 六. 私有Bucket静态资源缓存策略
① 将鉴权加密key, 下发给客户端, 客户端跟鉴权方式(A, B, C)计算URL
这么做好处是:
1. 客户端获取到的是一个不变URL, 有利于他们自己做缓存. 
2. 服务器端业务也不用关注URL的时效性, 业务简单

坏处:
1. 加密key需要设计如何下发
2. 加密key需要动态变化

结果: pass

② 不将鉴权加密key下发, 服务器端做缓存. 在一定时间段内, 客户端获取到是一样的URL
这么做好处:
1. 客户端业务简单
2. 加密key不下发

坏处:
服务端业务复杂

结果: pass

③ 终极方案:
服务端下发的URL如下:
```
http://dev.ling.cn/111/%E5%8A%AA%E5%8A%9B.png?auth_key=1510737594-0-0-03f44227da0a4b9b492e1ed652f523c3&timestamp=1234567
```
这个URL的组成: 
APP端采用URI+ timestamp 联合做KEY的方式, 缓存. 

