# Webpack chunk打包时，一定要采用非覆盖式升级

最近使用webpack时，遇到了一个问题：版本更新后，如果没有刷新页面就去动态加载某个chunk文件，可能会返回404，即找不到这个文件。

###### 问题复现条件

1. 使用webpack 动态import实现按需加载业务模块
2. chunk文件名带有哈希：3种哈希（hash/chunkhash/contenthash）都可以
3. 静态资源部署时，采用覆盖的方式。即：将目录下的所有文件都清空，再把最新的静态文件放到目录下
4. 升级前，有部分chunk文件没有加载
5. 升级后，浏览器不刷新，不清缓存，就直接请求该chunk文件

###### 问题现象

页面加载失败，浏览器的network显示该文件的HTTP状态码是404

![image](/images/Webpack%20chunk打包时，一定要采用非覆盖式升级/404.png)

###### 定位过程

既然chunk文件找不到，那肯定是当前服务器的目录中没有相应的文件，原因是升级时采用覆盖式的方式，旧的文件都被清掉了。但是，既然文件都升级过了，浏览器为啥还会请求旧的文件呢？

这个和webpack动态import的实现有关：

webpack会构建出bundle和chunk，bundle是根据入口文件生成的产物，index.html中会引入；chunk是使用动态import的方式，单独构建出来的文件，是需要的时候才引入的（e.g. 路由切换到指定的页面，或者点击了展开按钮，等等），如下面的代码所示：

bundle:

```javascript
entry: {
    vendor: ["@babel/polyfill"],
    businessAll: ['./ddm_main'],
  },
```

chunk的引入（其中一种方式）:

```javascript
 {
        url: "/instance_redis",
        template: require("../../redis/view/instance_redis.html"),
        controller: "instanceRedisCtrl",
        resolve: {
          lazyLoad: function() {
           
            return import(/* webpackChunkName:"redis" */ "app/business/redis/redis");
          }
        }
      }
```

以上是bundle和chunk的区别，这是webpack的基本概念。

既然浏览器会使用旧的hash，请求chunk文件，那这个hash肯定在某个地方定义过。在哪里定义呢？

先看一下bundle文件的代码：

```javascript
script.src = function(chunkId) {
                    return __webpack_require__.p + "" + ({
                        0: "redis",
                        1: "database",
                        2: "i18n/default/en-US",
                        3: "i18n/default/zh-CN",
                        4: "i18n/ctc/en-US",
                        5: "i18n/ctc/zh-CN",
                        6: "i18n/hws/en-US",
                        7: "i18n/hws/zh-CN"
                    }[chunkId] || chunkId) + "." + {
                        0: "f5f831d236203eb3b955",
                        1: "53c1b6b617d374879104",
                        2: "fa2bbaf0ae884fe204fa",
                        3: "3f202ffb3b1feaa4d18b",
                        4: "28f78b320dbf4ad87314",
                        5: "3b319c17624ef3a97570",
                        6: "8ec02eb39d901d73c2e6",
                        7: "bf87531c34846408f797"
                    }[chunkId] + ".js"
                }(chunkId),
```

在bundle文件中，webpack构建时会生成这样的一段代码，记录每个chunk的id，chunk名称，和chunk的哈希。然后根据这3个属性值拼装成一个请求的url，通过在DOM结构上面增加script标签的方式，去动态引入对应的文件。所有逻辑都在__webpack_require__.e这个函数中：

```javascript
__webpack_require__.e = function(chunkId) {
        var promises = []
          , installedChunkData = installedChunks[chunkId];
        if (0 !== installedChunkData)
            if (installedChunkData)
                promises.push(installedChunkData[2]);
            else {
                var promise = new Promise(function(resolve, reject) {
                    installedChunkData = installedChunks[chunkId] = [resolve, reject]
                }
                );
                promises.push(installedChunkData[2] = promise);
                var onScriptComplete, head = document.getElementsByTagName("head")[0], script = document.createElement("script");
                script.charset = "utf-8",
                script.timeout = 120,
                __webpack_require__.nc && script.setAttribute("nonce", __webpack_require__.nc),
                script.src = function(chunkId) {
                    return __webpack_require__.p + "" + ({
                        0: "redis",
                        1: "database",
                        2: "i18n/default/en-US",
                        3: "i18n/default/zh-CN",
                        4: "i18n/ctc/en-US",
                        5: "i18n/ctc/zh-CN",
                        6: "i18n/hws/en-US",
                        7: "i18n/hws/zh-CN"
                    }[chunkId] || chunkId) + "." + {
                        0: "f5f831d236203eb3b955",
                        1: "53c1b6b617d374879104",
                        2: "fa2bbaf0ae884fe204fa",
                        3: "3f202ffb3b1feaa4d18b",
                        4: "28f78b320dbf4ad87314",
                        5: "3b319c17624ef3a97570",
                        6: "8ec02eb39d901d73c2e6",
                        7: "bf87531c34846408f797"
                    }[chunkId] + ".js"
                }(chunkId),
                onScriptComplete = function(event) {
                    script.onerror = script.onload = null,
                    clearTimeout(timeout);
                    var chunk = installedChunks[chunkId];
                    if (0 !== chunk) {
                        if (chunk) {
                            var errorType = event && ("load" === event.type ? "missing" : event.type)
                              , realSrc = event && event.target && event.target.src
                              , error = new Error("Loading chunk " + chunkId + " failed.\n(" + errorType + ": " + realSrc + ")");
                            error.type = errorType,
                            error.request = realSrc,
                            chunk[1](error)
                        }
                        installedChunks[chunkId] = void 0
                    }
                }
                ;
                var timeout = setTimeout(function() {
                    onScriptComplete({
                        type: "timeout",
                        target: script
                    })
                }, 12e4);
                script.onerror = script.onload = onScriptComplete,
                head.appendChild(script)
            }
        return Promise.all(promises)
    }
```

由此可见，每次构建后，bundle文件中会保存本次构建的所有的chunk的id，名称，和hash，只要浏览器中的bundle文件没有更新，则当需要引入chunk模块时，就会根据bundle中记录的chunk信息去引入。如果此时服务器目录中恰好没有该文件，则服务器会返回404，所以对应的模块就加载失败。

举个例子：

第一次构建：

![image](/images/Webpack%20chunk打包时，一定要采用非覆盖式升级/第一次构建.png)

此时浏览器请求回来的文件如下，可见目前还没有请求过redis.xxx.js

![image](/images/Webpack%20chunk打包时，一定要采用非覆盖式升级/第一次构建后的资源.png)

打开entry的文件，查看assets:

![image](/images/Webpack%20chunk打包时，一定要采用非覆盖式升级/assets.png)

第二次构建

![image](/images/Webpack%20chunk打包时，一定要采用非覆盖式升级/第二次构建.png)

此时目录中的文件全部替换了

![image](/images/Webpack%20chunk打包时，一定要采用非覆盖式升级/第二次构建产物.png)

如果第二次构建完后，浏览器没有全页面刷新，且没有清除缓存，就会根据第一次的chunk hash请求旧的chunk文件，如果采用覆盖式更新，则上次构建的产物都被清除，就会导致文件找不到

![image](/images/Webpack%20chunk打包时，一定要采用非覆盖式升级/404.png)

###### 解决办法

既然知道了根因——前端静态文件采用覆盖式升级，导致旧的文件找不到。那么，我们只要不采用覆盖式的方式升级，这个问题即可解决：

非覆盖式的方式有两种：

1. 每次升级时，旧的文件不清除，直接把最新的文件部署到目录中。这样最简单，不过也有个缺点：随着版本的升级，这个目录中的文件会越集越多。目录下的文件数量会很大。
2. 每个版本对应服务器中的一个子目录，每次构建的产物根据版本号部署到对应的目录中，然后index.html更新资源目录，从新的版本号对应的目录中引入文件。如果浏览器没刷新，则会根据bundle中记录的chunk路径去旧的版本对应的目录中请求相应的文件；如果浏览器刷新了，则bundle，chunk都会从最新的目录下请求。这样新旧版本可以共存，也避免了1中的缺点，因为每个版本对应一个目录，不同版本下的文件肯定在不同的目录下存放，便于归档。

###### 备注：

如果chunk文件不带hash，是不是就没有这个问题了呢？

并不是，即时没有带hash，chunk文件依然可能加载失败。原因是：每个chunk都有唯一的ID，这个chunk id是构建时webpack根据扫描文件的顺序，递增得来的，不一定是一成不变的。

https://github.com/webpack/webpack/issues/4837

如果修改了原有业务模块中的代码，或者新增了某个按需加载的业务模块，webpack扫描文件的顺序就可能改变，则chunk id也会变化，变化后，原来的id可能就对应另外一个chunk了，如果升级时把旧的chunk文件删除了，而新生成的文件中又没有这个chunk，则文件加载失败。如下图所示：

![image](/images/Webpack%20chunk打包时，一定要采用非覆盖式升级/第二次构建hash.png)

![image](/images/Webpack%20chunk打包时，一定要采用非覆盖式升级/404.png)

如果升级后，这个chunk id仍然对应着一个文件，里面的逻辑也可能不是想要的模块，而是另外一个模块。