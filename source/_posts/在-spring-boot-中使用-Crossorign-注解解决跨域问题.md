---
title: 在 spring boot 中使用 @Crossorign 注解解决跨域问题
date: 2020-04-11 19:52:46
permalink: spring-boot-crossorign
categories: 
- spring boot
tags: 
- spring boot
- java
toc: true
---

最近在 spring boot 中写 rest api 时发现前端 ajax 调用的时候会产生跨域问题。于是乎想到用 jsonp 的方式进行解决，但是发现 `AbstractJsonpResponseBodyAdvice` 这个类在 spring boot 2.0 已经被废弃了，上官网发现有新姿势——[《Enabling Cross Origin Requests for a RESTful Web Service》](https://spring.io/guides/gs/rest-service-cors/)。

<!-- more -->

### 1. 浏览器的同源政策

**同源策略（SOP，Same origin policy）** 是由 Netscape 提出的一个著名的安全策略。所有支持 JavaScript 的浏览器都会使用这个策略。所谓同源是指，**域名，协议，端口相同**。即便两个不同的域名指向同一个 ip 地址，也非同源。

同源策略限制以下几种行为：

- 1、Cookie、LocalStorage 和 IndexDB 无法读取
- 2、Dom 和 Js 对象无法获得
- 3、Ajax 请求不能发送

其目的是为了保证用户信息的安全，防止恶意的网站窃取数据。举个例子：

- A 网站是一家银行，用户登录以后再去访问 B 网站。如果 B 网站可以读取 A 网站的 Cookie，那么 B 网站就可以利用这些信息为所欲为！（浏览器提交表单不受同源政策的限制）。

### 2. 跨域解决方案

- 1、jsonp（只支持 GET 请求）
- 2、document.domain + iframe
- 3、location.hash + iframe
- 4、window.name + iframe
- 5、postMessage
- 6、跨域资源共享（CORS）
- 7、nginx 代理
- 8、nodejs 中间件代理跨域
- 9、WebSocket 协议跨域

### 3. CORS

CORS 是一个 [W3C 标准](https://baike.baidu.com/item/W3C%E6%A0%87%E5%87%86)，全称是 **"跨域资源共享"（Cross-origin resource sharing）**，是一种允许当前域（domain）的资源（比如 html / js / web service）被其他域（domain）的脚本请求访问的机制（利用 [XMLHttpRequest](https://baike.baidu.com/item/XMLHTTPRequest) 请求）。由于同源策略，浏览器通常会禁止这种跨域请求。

CORS 请求分为两类：**简单请求**和**非简单请求**。

其中满足以下两大条件，就属于简单请求。

- 请求方法是以下三种方法之一：
  - HEAD
  - GET
  - POST
- HTTP 的头信息不超出以下几种字段：
  - AcceptAccept-Language
    - Content-Language
    - Last-Event-ID
    - Content-Type：只限于以下三个值
      - application/x-www-form-urlencoded
      - multipart/form-data
      - text/plain

不满足的为非简单请求。

浏览器对于两种请求的处理是不一样的：

#### 简单请求

- 浏览器会在简单请求的请求头中添加 `Origin` 字段，用来说明本次请求来自哪个源（协议 + 域名 + 端口）。服务器判断这个值决定是否通过这次请求
- 不通过时，返回头则没有 `Access-Control-Allow-Origin` 字段
- 通过时，则会多出以下几个字段

```html
Access-Control-Allow-Origin: http://api.baidu.com
Access-Control-Allow-Credentials: true Access-Control-Expose-Headers: FooBar
Content-Type: text/html; charset=utf-8
```

#### 非简单请求

- 在正式请求前会发送一个预检请求（OPTIONS）

```html
OPTIONS /cors HTTP/1.1 Origin: http://api.bob.comAccess-Control-Request-Method:
PUT Access-Control-Request-Headers: X-Custom-Header Host: api.baidu.com
Accept-Language: en-US Connection: keep-alive User-Agent: Mozilla/5.0...
```

- 一旦服务器通过了预检请求，以后每次浏览器正常的 CORS 请求，就都跟简单请求一样，返回头会多一个 `Origin` 字段以及 `Access-Control-Allow-Origin` 字段。

### 4.@Crossorigin 注解

#### 全局配置

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
@EnableWebMvc
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        //设置允许跨域的路径
        registry.addMapping("/**")
                //设置允许跨域请求的域名
                .allowedOrigins("*")
                //是否允许证书 不再默认开启
                .allowCredentials(true)
                //设置允许的方法
                .allowedMethods("*")
                //跨域允许时间
                .maxAge(3600);
    }
```

#### 局部配置

可以注解在整个 Controller 上。

```java
@CrossOrigin(origins="http://localhost:9000", maxAge=3600)
@RestController
public class RestController {}
```

也可以注解在单个方法上。

```java
@CrossOrigin(origins="http://localhost:9000")
@GetMapping("/hello")
public Greeting greeting() {
    return "world";
}
```

还可以通过 `Filter` 的方式指定接口。

```java
@Bean
public FilterRegistrationBean corsFilter() {
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowCredentials(true);
    config.addAllowedOrigin("http://localhost:9000");
    config.addAllowedOrigin("null");
    config.addAllowedHeader("*");
    config.addAllowedMethod("*");
    source.registerCorsConfiguration("/**", config); // CORS 配置对所有接口都有效
    FilterRegistrationBean bean = newFilterRegistrationBean(new CorsFilter(source));
    bean.setOrder(0);
    return bean;
}
```

### 5. 测试

新建另外一个 web 项目，此项目端口为 9000，即我们前面设置的 `origin` ，而 api 项目的端口为 8080。

其中主要代码如下：

`public/hello.js`

```js
$(document).ready(function () {
  $.ajax({
    url: "http://localhost:8080/hello",
  }).then(function (data, status, jqxhr) {
    $(".greeting-test").append(data);
    console.log(jqxhr);
  });
});
```

`public/index.html`

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Hello CORS</title>
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js"></script>
    <script src="hello.js"></script>
  </head>

  <body>
    <div>
      <p class="greeting-test">Hello</p>
    </div>
  </body>
</html>
```

请求成功时即可看到返回结果。

```html
Hello world
```

---

参考链接：

- [跨域资源共享 CORS 详解 —— 阮一峰](http://www.ruanyifeng.com/blog/2016/04/cors.html)
- [《Enabling Cross Origin Requests for a RESTful Web Service》](https://spring.io/guides/gs/rest-service-cors/)
