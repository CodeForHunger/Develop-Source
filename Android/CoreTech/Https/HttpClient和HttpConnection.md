<h1 align="center">HttpClient和HttpConnection</h1>

[TOC]

## HttpClient和HttpConnection的区别？

Http Client适用于web浏览器，拥有大量灵活的API，实现起来比较稳定，且其功能比较丰富，提供了很多工具，封装了http的请求头，参数，内容体，响应，还有一些高级功能，代理、COOKIE、鉴权、压缩、连接池的处理。但是，正因此，在不破坏兼容性的前提下，其庞大的API也使人难以改进，因此Android团队对于修改优化Apache Http Client并不积极。(并在Android 6.0中抛弃了Http Client，替换成OkHttp)

HttpURLConnection对于大部分功能都进行了包装，Http Client的高级功能代码会较复杂，另外，HttpURLConnection在Android 2.3中增加了一些Https方面的改进(包括Http Client，两者都支持https)。且在Android 4.0中增加了response cache。当缓存被安装后(调用HttpResponseCache的install()方法)，所有的HTTP请求都会满足以下三种情况：

- 所有的缓存响应都由本地存储来提供。因为没有必要去发起任务的网络连接请求，所有的响应都可以立刻获取到。
- 视情况而定的缓存响应必须要有服务器来进行更新检查。比如说客户端发起了一条类似于 “如果/foo.png这张图片发生了改变，就将它发送给我” 这样的请求，服务器需要将更新后的数据进行返回，或者返回一个304 Not Modified状态。如果请求的内容没有发生，客户端就不会下载任何数据。
- 没有缓存的响应都是由服务器直接提供的。这部分响应会在稍后存储到响应缓存中。

在Android 2.2版本之前，HttpClient拥有较少的bug，因此使用它是最好的选择。 而在Android 2.3版本及以后，HttpURLConnection则是最佳的选择。它的API简单，体积较小，因而非常适用于Android项目。压缩和缓存机制可以有效地减少网络访问的流量，在提升速度和省电方面也起到了较大的作用。对于新的应用程序应该更加偏向于使用HttpURLConnection，因为在以后的工作当中Android官方也会将更多的时间放在优化HttpURLConnection上面。