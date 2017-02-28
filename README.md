# dev-serv

开发时使用的专属服务器

* 安装 `dev-serv` 模块

  `npm install dev-serv --save-dev`
* 增加服务器的配置文件
  
  从 `dev-serv/example` 中复制 `_dev-serv.json` 到项目根目录

  配置项说明请参考 [src/dev-serv.js#DEV_SERV_CONFIG](https://github.com/ufologist/dev-serv/blob/master/src/dev-serv.js), 一般仅需要配置 `backend` 参数即可
* 配置项目的 `package.json` 脚本

  ```json
  "scripts": {
      "dev": "dev-serv"
  }
  ```
* 启动服务器

  `npm run dev`

## 开发时使用的服务器需要具备哪些功能

* 静态文件服务器
* 静态文件修改后浏览器自动刷新
* Mock API Server
* 代理后端真实接口(最好 Mock API Server 能够灵活切换到真实接口或者使用 Mock 接口)

即需要集 static server + mock server + proxy server + live-reload 于一身

## 实现方案

* 使用 expres 来做静态文件服务器                      -> 正式环境为 nginx
* 使用 puer-mock 来做 API Server(代理后端的真实接口)  -> 方便切换 mock 接口或者真实接口
* 使用 http-proxy-middleware 来代理 API Server       -> 正式环境为 nginx 反向代理(或者跨域调用)
* 使用 puer 中间件来做静态文件修改后浏览器自动刷新      -> 方便开发

即形成下面的结构

```
【静态服务器】                                                    【Mock API Server】
http://localhost:8000                                            http://localhost:8001
┏━━━━━━━━━━━━━━━━━━┓                            ┏━━━━━━━━━━━━━━━━━━┓ 
┃                                 ┃                            ┃                                 ┃
┃ http://localhost:8000/a.html    ┃                            ┃ Mock API Route Config           ┃
┃ http://localhost:8000/a.css     ┃                            ┃ http://localhost:8001/user/list ┃
┃                                 ┃                            ┃                                 ┃
┃ http://localhost:8000/api       ┃ --http-proxy-middleware--> ┃      Mock API Server(puer-mock) ┃
┃                                 ┃                            ┗━━━━━━━━━━━━━━━━━━┛
┃          Static Server(express) ┃                                           
┃          + Auto Reload          ┃                                           ┃
┃       (puer-connect-middleware) ┃                                          proxy
┗━━━━━━━━━━━━━━━━━━┛                                           ┃
                                                                                v

                                                                 【后端真实接口服务器】
                                                                 http://localhost:18520
                                                                 ┏━━━━━━━━━━━━━━━━━━┓
                                                                 ┃                                 ┃
                                                                 ┃http://localhost:18520/user/list ┃
                                                                 ┃                                 ┃
                                                                 ┃             API Server(Backend) ┃
                                                                 ┗━━━━━━━━━━━━━━━━━━┛
```

* 通过 `:8000` 端口服务静态文件
* 通过 `:8000/api` 代理在 `:8001` 端口的后端接口
  * puer-mock 在 `:8001` 端口启动 mock 接口服务
  * puer-mock 同时代理 `:18520` 端口的后端真实接口, 方便开发时可以随时切换成真实接口

因此页面上面所有接口的根路径应该为 `:8000/api`

例如:

```javascript
// 通过代理调用后端接口, 会去请求 puer-mock 的服务
// 因此实际上请求会发给 http://localhost:8001/user/list
//
// 如果 puer-mock 上禁用了这个 mock 接口, 由于 puer-mock 同时代理了后端真实接口
// 因此实际上请求会发给 http://localhost:18520/user/list
$.get('http://localhost:8000/api/user/list');
```