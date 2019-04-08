### 一、什么是 AJAX 跨域问题

[同源策略](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)规定，AJAX 请求（[`XMLHttpRequest`](http://www.ruanyifeng.com/blog/2012/09/xmlhttprequest_level_2.html)）只能发给同源的网址，否则就会出错。所谓的同源策略是指 3 个相同：协议相同、域名相同、端口相同。

比如 `http://www.example.com/index.html` 这个网址，协议是`http` ，域名是 `www.example.com`，端口是默认的 `80`。如果你在这个网站上使用 AJAX 发送 `http://www.example1.com/index.html` 请求，就会出错，因为域名不同。

AJAX 跨域的根本原因是浏览器不允许这么做（不是服务端的问题），浏览器限制 AJAX 跨域请求的目的是为了保证用户信息的安全，防止数据被恶意获取。

### 二、JSONP 解决跨域问题

**2.1 JSONP 如何解决跨域问题**

 - AJAX 请求非同源资源会发生跨域问题，但是有的 HTML 标签支持非同源请求，举例来说：`<img src="">` 与 `<script src="">` 标签
 - 使用 JSONP 解决跨域问题，有一种方法，前端以 `<script>` 同样的方式发送请求，把返回的 JSON  数据封装，然后以 JS 脚本形式返回
 - JSONP 允许客户端传递一个 `callback` 参数请求跨域服务端，服务端返回数据时用 `callback` 参数名作为返回的函数名来包裹住返回的 JSON 数据

**2.2 JSONP 示例**

这里我们以 MOOC 网为例，当访问 `https://www.imooc.com/` 时，会发送一个 `https://www.imooc.com/index/getstarlist`  AJAX 请求，下面是访问后返回的 JSON 数据。
![这里写图片描述](https://i.loli.net/2018/09/21/5ba4a796cd244.png)

我们以 JONP 的形式（`https://www.imooc.com/index/getstarlist?callback=test` ）访问。
![这里写图片描述](https://i.loli.net/2018/09/21/5ba4a796e15f1.png)

**2.3 JSONP 的弊端**

 - 需要在后端改动代码，比如在 Spring 后台代码中需要配置一个 `AbstractJsonpResponseBodyAdvice` 切面，不过现在这个类已经过时了，也就是说现在不推荐使用 JSONP 来解决跨域问题
 - JSONP 默认只支持 `GET` 方式的请求
 - 发送的不是 `XMLHttpRequest` 请求（`script`），所以不支持异步事件等，不过这也是为什么 JSONP 可以解决跨域问题的原因

### 三、CORS 解决跨域问题

CORS 跨资源共享（Cross-origin resource sharing）是一种机制，它通过添加 HTTP  头信息，解决 AJAX 跨域资源访问问题。

CORS 请求可以分为两类：简单请求与非简单请求，主要区别是非简单请求在进行访问之前，会发送一个预检请求。“预检”请求首先通过 OPTIONS 方法向另一域上的资源发送HTTP请求，以便确定实际请求是否安全发送。为了防止每次非简单请求之前都发送预检请求，可以在服务端设置预检请求的缓存的时间。

**3.1 简单请求**

请求方法为 `HEAD`、`GET`、`POST` 中的 1 种
 - 请求的 `header` 中没有自定义的请求头
 - `Content-Type` 为以下几种：`application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`

**3.2 非简单请求**

 - `header` 中包含自定义请求头 的 AJAX 请求
 - `PUT`、`DELETE` 形式的 AJAX 请求
 - `Content-Type` 字段的类型是 `application/json` 等

我自己在本地写了一个测试跨域的 demo（Spring Boot），通过端口 8082 发出请求（携带自定义的请求头）访问端口为 8081 的服务。下面是 Chrome 控制台有关 HTTP 的控制信息。
![这里写图片描述](https://i.loli.net/2018/09/21/5ba4a797227d1.png)<br>
通过上面的图可以看出 CORS 解决跨域问题的关键所在，添加必要的请求头与响应头信息，下面是有关的头信息介绍。

**3.3 HTTP 响应首部字段**

 - `Access-Control-Allow-Origin`：指定了允许访问该资源的外域 URI。对于不需要携带身份凭证的请求，服务器可以指定该字段的值为通配符，表示允许来自所有域的请求。
 - `Access-Control-Allow-Methods`：指明了实际请求所允许使用的 HTTP 方法
 - `Access-Control-Expose-Headers`：在跨域访问时，`XMLHttpRequest` 对象的`getResponseHeader()` 方法只能拿到一些最基本的响应头，如果要携带自定义的请求头，需要在该首部中进行允许的请求头放入白名单
 - `Access-Control-Max-Age`：非简单请求预检命令的最大缓存时间，在缓存时间内，对于非简单请求，不会再发送预检请求
 - `Access-Control-Allow-Credentials`：是否允许携带身份凭证（Cookie）的请求

**3.4 HTTP 请求首部字段**

 - `Origin`：表明预检请求或实际请求的源 URL
 - `Access-Control-Request-Method`：用于预检请求。其作用是，将实际请求所使用的 HTTP 方法告诉服务器
 - `Access-Control-Request-Headers`：用于预检请求。其作用是，将实际请求所携带的首部字段告诉服务器

**3.5 CORS 跨域相关代码**

``` java
public class MyCorsFilter extends OncePerRequestFilter {
    private static final String ORIGIN = "Origin";
    private static final String HEADERS = "Access-Control-Request-Headers";
    private static final String REQUEST_METHOD_FOR_CHECK = "OPTIONS";

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        // 获取请求头中的 'Origin' 信息
        String origin = request.getHeader(ORIGIN);
        // 获取请求头中的 'header' 信息
        String headers = request.getHeader(HEADERS);

        /**
         * 1.支持任何域名跨域访问
         * 当 'Access-Control-Allow-Origin' 设置为 '*' 时，不能解决带 Cookie 的跨域
         */
        if (!StringUtils.isEmpty(origin)) {
            response.setHeader("Access-Control-Allow-Origin", origin);
        }
        /**
         * 2.支持自定义请求头的跨域
         */
        if (!StringUtils.isEmpty(headers)) {
            response.setHeader("Access-Control-Allow-Headers", headers);
        }
        // 3.设置支持带 Cookie 的跨域请求
        response.setHeader("Access-Control-Allow-Credentials", "true");
        // 4.设置允许跨域请求的方法形式 'GET'、'DELETE' 等
        response.setHeader("Access-Control-Allow-Methods", "*");
        // 5.设置非简单请求的预检命令缓存时间，单位 's'
        response.setHeader("Access-Control-Max-Age", "1728000");

        if (REQUEST_METHOD_FOR_CHECK.equals(request.getMethod())) {
            response.setStatus(HttpServletResponse.SC_OK);
        } else {
            filterChain.doFilter(request, response);
        }

    }
}

```
### 四、other
能解决跨域的方式还有很多，比如禁止浏览器的跨域、Nginx 解决跨域等。这里只讲解了两种常见的跨域方式，因为 JSONP 存在一些弊端，因此推荐使用 CORS 等方式来解决跨域问题。

### 参考资料
[跨域资源共享 CORS 详解l](http://www.ruanyifeng.com/blog/2016/04/cors.html)<br>
[【原创】说说JSON和JSONP，也许你会豁然开朗，含jQuery用例](https://www.cnblogs.com/dowinning/archive/2012/04/19/json-jsonp-jquery.html)  <br>
[Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)<br>
[ajax跨域完全讲解 MOOC](https://www.imooc.com/learn/947)
