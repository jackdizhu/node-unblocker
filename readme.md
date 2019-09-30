
# unblocker

Unblocker 是用于Web代理,用于代理和重写远程网页的通用库。类似于CGIproxy / PHProxy / Glype

### 神奇的部分

无需修改即可处理相对路径的链接。(E.g. `<a href="path/to/file2.html"></a>`)

无需修改即可处理相对于根路径的链接。 (E.g. `<a href="/path/to/file2.html"></a>`)

通过调整Cookie的路径来代理Cookie，并做了一些额外的工作来确保它们在切换协议或子域时保持完整。

### 局限性

尽管该代理对于标准登录表单甚至大多数AJAX内容都适用，但OAuth登录表单以及任何使用postMessage的东西（Google，Facebook等）都不太可能立即可用 。这不是一个无法解决的问题，但我也不希望在短期内解决这个问题。欢迎使用修补程序，包括用于主库的通用修补程序和用于示例文件夹的特定于站点的修补程序。


## unblocker作为库使用

```
npm i unblocker --s
```
```js
const express = require('express')
const Unblocker = require('unblocker');
const app = express();
const http = require('http')

// this must be one of the first app.use() calls and must not be on a subdirectory to work properly
app.use(new Unblocker({ prefix: '/proxy/' }));

app.get('/', function (req, res) {
  res.send('init');
});


/**
 * Listen on provided port, on all network interfaces.
 */

const server = http.createServer(app);
server.listen(3001);
```

### 配置选项

Unblocker支持以下配置选项，显示默认值:

```js
{
    prefix: '/proxy/',  // 代理URL的起始路径
    host: null, // 重定向中使用的主机（例如`example.com`或`localhost：8080`）。默认行为是根据请求标头确定此行为
    requestMiddleware: [], // 在将客户端请求发送到远程服务器之前对它们进行额外处理的函数数组。API详细说明如下。
    responseMiddleware: [], // 在将远程响应发送回客户端之前对远程响应执行额外处理的函数数组。API详细说明如下。
    standardMiddleware: true, // 如果需要执行请求或响应的高级自定义，则可以禁用所有内置的中间件。
    processContentTypes: [ // 所有用于修改响应内容的内置中间件都将自身限制为这些内容类型。
        'text/html',
        'application/xml+xhtml',
        'application/xhtml+xml',
        'text/css'
    ],
    httpAgent: null, // 重写代理，用于从服务器请求HTTP响应 请参阅https://nodejs.org/api/http.html#http_class_http_agent
    httpsAgent: null // 覆盖用于从服务器请求https响应的代理。请参阅https://nodejs.org/api/https.html#https_class_https_agent
}
```

#### 定制中间件

Unblocker `middleware`是一个小的方法库，可让您检查和修改请求和响应。Unblocker的大多数内部逻辑都隐含为中间件，并且可以编写自定义中间件来扩充或替换内置的中间件。

自定义中间件应该是一个接受单个data参数并同步运行的函数。

要处理请求和响应数据，请创建一个转换流以大块形式执行处理，并通过该流进行管道传输。（下面的示例。）

要直接响应请求，请在其上添加一个函数来config.requestMiddleware处理clientResponse（直接使用时为标准http.ServerResponse，或者与Express一起使用时为Express Response。发送响应后，将不再对该请求执行任何中间件。（下面的示例。）

##### requestMiddleware

Data example:
```js
{
    url: 'http://example.com/',
    clientRequest: {request},
    clientResponse: {response},
    headers: {
        //...
    },
    stream: {ReadableStream of data for PUT/POST requests, empty stream for other types}
}
```

requestMiddleware可以检查标头，URL等。它可以修改标头，通过转换流传递PUT / POST数据或直接响应请求。如果使用Express，则请求和响应对象将具有所有通常的Express好东西。例如：

```js
function validateRequest(data) {
    if (!data.url.match(/^https?:\/\/en.wikipedia.org\//) {
        data.clientResponse.status(403).send('Wikipedia only.');
    }
}
var config = {
    requestMiddleware: [
        validateRequest
    ]
}
```

##### responseMiddleware

responseMiddleware接收与datarequestMiddleware 相同的对象，但是将headers和stream字段替换为远程服务器的响应对象，并为远程请求和响应添加了几个新字段：

Data example:
```js
{
    url: 'http://example.com/',
    clientRequest: {request},
    clientResponse: {response},
    remoteRequest {request},
    remoteResponse: {response},
    contentType: 'text/html',
    headers: {
        //...
    },
    stream: {ReadableStream of response data}
}
```

要修改内容，请创建一个新的流，然后通过管道 `data.stream` 将其传输并替换 `data.stream` 为:

```js
var Transform = require('stream').Transform;

function injectScript(data) {
    if (data.contentType == 'text/html') {

        // https://nodejs.org/api/stream.html#stream_transform
        var myStream = new Transform({
            decodeStrings: false,
            function(chunk, encoding, next) {
                chunk = chunk.toString.replace('</body>', '<script src="/my/script.js"></script></body>');
                this.push(chunk);
                next();
        });

        data.stream = data.stream.pipe(myStream);
    }
}

var config = {
    responseMiddleware: [
        injectScript
    ]
}
```

##### 内置中间件

代理的大多数内部功能也实现为中间件:

* **host**: 更正传出响应头

* **referer**: 更正传出请求头

* **cookies**:

修复设置cookie头的`Path`属性，将cookie限制为其在代理上的`Path`(e.g. `Path=/proxy/http://example.com/`)。

还插入重定向以从给定域上的协议和子域之间复制cookie。

* **hsts**：删除[Strict-Transport-Security](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security)，因为它们可能泄漏到其他站点并可能破坏代理。

* **hpkp**：删除[Public-Key-Pinning](https://en.wikipedia.org/wiki/HTTP_Public_Key_Pinning)，因为它们可能泄漏到其他站点并可能破坏代理。

* **CSP**：删除[Content-Security-Policy](https://en.wikipedia.org/wiki/Content_Security_Policy)，因为它们可能泄漏到其他站点并可能破坏代理。

* **redirects**：重写3xx重定向中的url以确保它们通过代理

* **decompress**：解压`Content-Encoding: gzip|deflate`响应，并调整请求头以请求仅使用gzip或根本不使用压缩。（它将尝试解压缩`deflate`内容，但存在一些问题，因此它不公布对`deflate`的支持。）

* **charsets**：将响应的字符集转换为UTF-8，以便在node.js中进行安全的字符串处理。从头或元标记确定字符集，并重写传出响应中的所有头和元标记。

* **urlpeffixer**：重写`links/images/css/etc`，以确保它们通过代理。

* **metaRobots**：注入ROBOTS: NOINDEX, NOFOLLOW meta tag，以防止搜索引擎通过代理爬行整个web。

* **contentLength**：如果修改了正文，则删除`content-length`响应头。

将`standardMiddleware`配置选项设置为`false`将禁用所有内置中间件，允许您有选择地启用、配置和重新排序内置中间件。

此配置将模拟默认设置：

```js

var Unblocker = require('unblocker');

var config = {
    prefix: '/proxy/',
    host: null,
    requestMiddleware: [],
    responseMiddleware: [],
    standardMiddleware: false,  // disables all built-in middleware
    processContentTypes: [
        'text/html',
        'application/xml+xhtml',
        'application/xhtml+xml'
    ]
}

var host = Unblocker.host(config);
var referer = Unblocker.referer(config);
var cookies = Unblocker.cookies(config);
var hsts = Unblocker.hsts(config);
var hpkp = Unblocker.hpkp(config);
var csp = Unblocker.csp(config);
var redirects = Unblocker.redirects(config);
var decompress = Unblocker.decompress(config);
var charsets = Unblocker.charsets(config);
var urlPrefixer = Unblocker.urlPrefixer(config);
var metaRobots = Unblocker.metaRobots(config);
var contentLength = Unblocker.contentLength(config);

config.requestMiddleware = [
    host,
    referer,
    decompress.handleRequest,
    cookies.handleRequest
    // custom requestMiddleware here
];

config.responseMiddleware = [
    hsts,
    hpkp,
    csp,
    redirects,
    decompress.handleResponse,
    charsets,
    urlPrefixer,
    cookies.handleResponse,
    metaRobots,
    // custom responseMiddleware here
    contentLength
];

app.use(new Unblocker(config));
```

## Debugging

Unblocker 通过环境变量启用调试:

    DEBUG=unblocker:* node mycoolapp.js

还有一个中间件调试器，可在每个现有中间件功能之前和之后添加额外的调试中间件以报告更改。它包含在默认的DEBUG激活中，也可以有选择地启用：

    DEBUG=unblocker:middleware node mycoolapp.js

... 或 禁用:

    DEBUG=*,-unblocker:middleware node mycoolapp.js

## 故障排除

如果您将Nginx用作反向代理，则可能需要禁用该功能`merge_slashes`以避免无止境的重定向和/或其他问题

    merge_slashes off;


## Todo

* Consider adding compress middleware to compress text-like responses
* Un-prefix urls in GET / POST data
* Inject js to proxy postMessage data and fix origins
* More examples
* Even more tests

## AGPL-3.0 License
This project is released under the terms of the [GNU Affero General Public License version 3](https://www.gnu.org/licenses/agpl-3.0.html).

All source code is copyright [Nathan Friedly](http://nfriedly.com/).

Commercial licensing and support are also available, contact Nathan Friedly (nathan@nfriedly.com) for details.

## Contributors
* [Nathan Friedly](http://nfriedly.com)
* [Arturo Filastò](https://github.com/hellais)
* [tfMen](https://github.com/tfMen)
* [Emil Hemdal](https://github.com/emilhem)
