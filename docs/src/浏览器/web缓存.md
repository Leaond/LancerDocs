# Web 缓存

Web 缓存分为 HTTP 缓存 和 浏览器缓存，下面将进行分别梳理。

## HTTP 缓存

HTTP 缓存机制是 web 性能优化的重要手段。指的是当客户端向服务器请求资源时，会先抵达浏览器缓存，如果浏览器有要请求的资源的副本，就直接从浏览器缓存中提取，而不是从服务器中提取这个资源。

常见的 HTTP 缓存只能缓存 GET 请求响应的资源，对于其他类型的响应则无能为力。

HTTP 缓存都是从第二次请求开始的。第一次请求资源时，服务器返回资源，并在响应头中回传资源的缓存参数；第二次请求时，浏览器判断这些请求参数，命中`强缓存`就直接 200，否则就把请求参数加到请求头中传给服务器，看是否命中`协商缓存`，命中则返回 304，否则服务器会返回新的资源。
:::tip 为什么要使用 HTTP 缓存

- 减少了冗余的数据传输，节省了网费
- 缓解了服务器端的压力
- 加快了客户端加载网页的速度，大大提高了网站的性能
  :::

### HTTP 缓存的分类

根据是否需要重新向服务器发起请求，HTTP 缓存可分为 `强制缓存` 和 `协商缓存` 。强制缓存如果生效，不需要再和服务器发生交互；而协商缓存不管是否生效，都需要和服务器发生交互。

### 强缓存

强制缓存在缓存数据未失效的情况下，即 Cache-Control 的 max-age 没有过期或者 Expires 的缓存时间没有过期，那么就会直接使用浏览器的缓存数据，不会再向服务器发送任何请求。强制缓存生效时，HTTP 的状态码为 200，同时我们可以以通过开发者工具在浏览器网络面板的 size 列看到`from disk cache`的提示。

这种方式页面的加载速度时最快的，性能也是很好的，但是如果在这期间，服务器端的资源修改了，页面上是拿不到的，因为它不会再向服务器发请求了。

跟强制缓存相关的头属性有 Expires 和 Cache-Control，用来表示资源的缓存时间。

1. Expires：响应头的缓存字段。GMT 格式日期，代表资源过期时间，由服务器返回。如果时间没过期，不发起请求，直接使用本地缓存；如果时间过期，发起请求。是 HTTP1.0 的属性，在与 max-age 共存的情况下，优先级要低。
2. Cache-Control：请求头/响应头的缓存字段。

:::tip 属性值

- no-store：所有内容都不缓存；
- no-cache：缓存，但是浏览器使用缓存前，都会请求服务器判断缓存资源是否是最新。
- max-age：单位秒，请求资源后的 xx 秒内不再发起请求。属于 HTTP1.1 属性，与 Expires 类似，但优先级要比 Expires 高。
- s-maxage：单位秒，代理服务器请求源站资源后的 xx 秒内不再发起请求，只对 CDN 有效。
- public：客户端和代理服务器（CDN）都可缓存。
- private：只有客户端可以缓存。
  :::

其实这两者差别不大，区别就在于 Expires 是 http1.0 的产物，Cache-Control 是 http1.1 的产物，两者同时存在的话，Cache-Control 优先级高于 Expires；在某些不支持 HTTP1.1 的环境下，Expires 就会发挥用处。所以 Expires 其实是过时的产物，现阶段它的存在只是一种兼容性的写法.

### 协商缓存

当浏览器第一次向服务器发送请求时，会在响应头返回协商缓存的头属性：ETag 和 Last-Modified，其中 ETag 返回的是一个 hash 值，Last-Modified 返回的是 GMT 格式的时间，标识该资源的最新修改时间；然后浏览器发送第二次请求的时候，会在请求头中带上与 ETag 对应的 If-Not-Match，与 Last-Modified 对应的 If-Modified-Since；服务器在接收到这两个参数后会做比较，会优先验证 ETag，一致的情况下，才会继续比对 Last-Modified；如果返回的是 304 状态码，则说明请求的资源没有修改，浏览器可以直接在缓存中读取数据，否则，服务器直接返回数据。

跟协商缓存相关的头属性有 Last-Modified/If-Modified-Since、ETag/If-Not-Match，请求头和响应头需要成对出现。

- Last-Modefied： 响应头的缓存字段。资源最新修改时间，由服务器告诉浏览器。
- if-Modified-Since：请求头的缓存字段。资源最新修改时间，由浏览器告诉服务器，和 Last-Modefied 是一对，它两会进行对比。（其实就是上次服务器给的 Last-Modified，请求又还给服务器对比）
- Etag：响应头的缓存字段。资源标识，由服务器告诉浏览器。
- if-None-Match：请求头的缓存字段。资源标识，由浏览器告诉服务器，和 Etag 是一对，它两会进行对比。（其实就是上次服务器给的 Etag，请求又还给服务器对比）

HTTP1.1 中 Etag 的出现主要是为了解决几个 Last-Modified 比较难解决的问题：

- Last-Modified 标注的最新修改时间只能精确到秒，如果某些文件在 1 秒以内被多次修改的话，它将不能准确标注文件的修改时间；
- 如果某些文件是被定期生成的话，内容没有任何改变，但 Last-Modified 却变了，导致文件无法再使用缓存；
- 有可能存在服务器没有准确获取文件修改时间，或者与代理服务器时间不一致等情形；
- ETag 是服务器自动生成或者由开发者生成的对应资源在服务器端的唯一标识，ETag 可以保证每一个资源是唯一的，资源变化都会导致 ETag 变化，ETag 值的变更则说明资源状态已经被修改，ETag 能够更加准确地控制缓存。Last-Modified 是可以与 ETag 一起使用的，服务器会优先验证 ETag，一致的情况下，才会继续比.Last-Modified，最后才决定是否返回 304。

:::tip
强缓存 和 协商缓存 规则可以同时存在，强制缓存优先级高于协商缓存，也就是说，当执行强制缓存的规则时，如果缓存生效，直接使用缓存，不再执行协商缓存规则。
:::

### 哪些请求不能被缓存？

- HTTP 信息头中包含 Cache-Control:no-cache，pragma:no-cache，或 Cache-Control:max-age=0 等告诉浏览器不用缓存的请求
- 需要根据 Cookie，认证信息等决定输入内容的动态请求是不能被缓存的
- 经过 HTTPS 安全加密的请求（有人也经过测试发现，ie 其实在头部加入 Cache-Control：max-age 信息，firefox 在头部加入 Cache-Control:Public 之后，能够对 HTTPS 的资源进行缓存。
- POST 请求无法被缓存
- HTTP 响应头中不包含 Last-Modified/Etag，也不包含 Cache-Control/Expires 的请求无法被缓存

### HTTP 缓存的应用

- 同一个资源保证 URL 的稳定性。URL 是浏览器缓存机制的基础，所以如果一个资源需要在多个地方被引用，尽量保证 URL 是固定的。同时，比较推荐使用公共类库，比如 Google Ajax Library 等，有利于最大限度使用缓存
- 给 css、js、图片等资源增加 HTTP 缓存头，并强制入口 html 不被缓存。对于不经常修改的静态资源，比如 Css，js，图片等，可以设置一个较长的过期的时间，或者至少加上 Last-Modified/Etag，而对于 html 页面这种入口文件，不建议设置缓存。这样既能保证在静态资源不变了情况下，可以不重发请求或直接通过 304 避免重复下载，又能保证在资源有更新的，只要通过给资源增加时间戳或者更换路径，就能让用户访问最新的资源
- 减少对 Cookie 的依赖。过多的使用 Cookie 会大大增加 HTTP 请求的负担，每次 GET 或 POST 请求，都会把 Cookie 都带上，增加网络传输流量，导致增长交互时间；同时 Cache 是很难被缓存的，应该尽量少使用，或者这在动态页面上使用。
- 减少对 HTTPS 加密协议的使用。通过 HTTPS 请求的资源，默认是不会被缓存的，必须通过特殊的配置，才能让资源得到缓存。建议只对涉及敏感信息的请求使用 HTTPS 传输，其他类似 Css，Js，图片这些静态资源，尽量避免使用。
- 多用 Get 方式请求动态 Cgi。虽然 POST 的请求方式比 Get 更安全，可以避免类似密码这种敏感信息在网络传输，被代理或其他人截获，但是 Get 请求方式更快，效率更高，而且能被缓存，建议对于那些不涉及敏感信息提交的请求尽量使用 Get 方式请求。

## 浏览器缓存

浏览器缓存分为 cookie 和 webstorage，而 webstorage 分为 sessionstorage 和 localstorage。

### localstorage

localStorage 的生命周期是永久，除非手动去清除，否则永远都存在，他的储存大小是 5MB，仅在客户端浏览器上储存，不参与服务器的通信。
语法

```js
//设置localStorage保存到本地，第一个为变量名，第二个是值
localStorage.setItem("user", "XXX");
// 获取数据
localStorage.getItem("user");
// 删除指定保存的数据
localStorage.removeItem("user");
// 清除所有保存的数据
localStorage.clear();
```

### sessionstorage

sessionStorage 在当前会话下有效，引入了一个 浏览器窗口的概念,sessionStorage 是在同源的同窗口中，始终存在的数据，只要浏览器不关闭，即使是刷新或者进入同源的另一个页面，数据仍在。同时打开“独立”的窗口，即使是同一个页面，sessionStorage 的对象也是不同的。关闭窗口后 sessionStorage 就会被销毁。

```js
//设置localStorage保存到本地，第一个为变量名，第二个是值
sessionstorage.setItem("user", "XXX");
// 获取数据
sessionstorage.getItem("user");
// 删除指定保存的数据
sessionstorage.removeItem("user");
// 清除所有保存的数据
sessionstorage.clear();

```
### cookie

cookie 是一个非常具体的东西，指的就是浏览器里面永久存储的一种数据，仅仅是浏览器实现的一种数据存储模式。

cookie 是服务器生成的，发送给浏览器，浏览器把 cookie 以 key=value 的形式保存到某个目录下面的文件夹里面，下一次请求同一个网站的时候会把该 cookie 发送给服务器。由于 cookie 是存在客户端的，所以浏览器加了一些限制确保 cookie 不会被恶意使用，同时不会占据太多磁盘空间，每个域的 cookie 数量是有限的的，一般浏览器都限制 50 个 cookie。cookie一般情况下是有大小限制的，4KB左右。

cookie 一般与特定的 web 网站关联在一起，保存了该客户端访问这个 web 网站的信息。由于 cookie 可以保存在客户端，所以我们可以用来实现记录用户个人信息的功能，例如记住密码，自动登录等。

缺点：不安全，能够被所有人看到信息，用户还可以自己去更改

原生使用 cookie

```js
// 设置 cookie
let oDate = new Date(); //获取系统当前时间
oDate.setDate(oDate.getDate() + 30); // 设置距离当前时间多少天后 cookit 过期
// 设置 cookie 及过期时间
document.cookie = "userName=hello;expires=" + oDate;
document.cookie = "password=123456;expires=" + oDate;
alert(document.cookie);
//获取 cookie
let oCookie = document.cookie.split("; ");
for (let i = 0; i < oCookie.length; i++) {
  let temp = oCookie[i].split("=");
  if (i == 1) {
    document.getElementById("userName").value = temp[1];
  }
  if (i == 0) {
    document.getElementById("password").value = temp[1];
  }
}
````