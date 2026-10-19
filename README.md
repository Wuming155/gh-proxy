# gh-proxy

基于 **Cloudflare Workers** 的 GitHub 文件与 API 加速代理。在原始 GitHub 链接前加上本项目的 Worker 地址，即可加速访问 `release` 资源、`archive` 源码包、仓库文件、`gist` 以及 `GitHub API`，并支持 `git clone`。

> 本项目源于 [hubporg/CF-GitHub-Proxy](https://github.com/hubporg/CF-GitHub-Proxy)，其核心代理逻辑派生自 [hunshcn/gh-proxy](https://github.com/hunshcn/gh-proxy)，前端页面参考自 [cmliu/CF-Workers-GitHub](https://github.com/cmliu/CF-Workers-GitHub)。

## 功能特性

- **多类型链接加速**：`release` 下载、`archive` 源码包、`blob` / `raw` 文件、`gist`、`tags`、`info` / `git-*` 等。
- **GitHub API 加速**：支持代理 `api.github.com` 请求。
- **签名直链与 CDN 分流**：支持 `release-assets.githubusercontent.com` 签名直链、`codeload.github.com` 源码归档、`media.githubusercontent.com`（Git LFS）等 CDN 主机。
- **`git clone` 可用**：可直接用作 `remote`，私有仓库支持携带 `TOKEN`。
- **可选 jsDelivr 镜像**：分支文件可切换走 `cdn.jsdelivr.net`。
- **CORS / 预检**：自动处理 `OPTIONS` 预检并返回 `Access-Control-Allow-Origin: *`。
- **内置加速链接生成页**：自带单文件静态页面，粘贴原始链接即自动生成加速链接。

## 项目文件

| 文件 | 说明 |
| --- | --- |
| `workers.js` | Cloudflare Worker 主体代码（代理逻辑）。 |
| `index.html` | 单文件静态页面，提供「粘贴 → 生成加速链接」的交互界面，由 Worker 回源提供。 |
| `LICENSE` | MIT 许可证。 |

## 快速开始

### 在线使用

在任意受支持的 GitHub 链接前拼接 Worker 地址即可，例如：

```
https://<你的Worker域名>/https://github.com/owner/repo/releases/download/v1.0.0/file.zip
```

也可以直接访问 Worker 首页，在输入框中粘贴原始链接，自动生成加速链接。

### 私有仓库 Clone

在地址中携带用户名与 Token：

```bash
git clone https://user:TOKEN@<你的Worker域名>/https://github.com/owner/private-repo.git
```

### 合法输入示例

| 类型 | 示例链接 |
| --- | --- |
| 分支源码 | `https://github.com/hunshcn/project/archive/master.zip` |
| release 源码 | `https://github.com/hunshcn/project/archive/v0.1.0.tar.gz` |
| release 文件 | `https://github.com/hunshcn/project/releases/download/v0.1.0/example.zip` |
| 分支 / commit 文件 | `https://github.com/hunshcn/project/blob/123/filename` |
| gist | `https://gist.githubusercontent.com/cielpy/123/raw/cmd.py` |
| api | `https://api.github.com/repos/Geekertao/CF-Workers-GitHub-Proxy` |
| release 签名直链 | `https://release-assets.githubusercontent.com/github-production-release-asset/xxx?sig=xxx` |
| 源码归档分流 | `https://codeload.github.com/hunshcn/project/tar.gz/refs/tags/v0.1.0` |
| Git LFS 媒体文件 | `https://media.githubusercontent.com/media/hunshcn/project/master/filename` |

> 暂不支持文件夹（目录）下载。`blob` / `raw` 分支文件在开启 jsDelivr 镜像时会跳转至 `cdn.jsdelivr.net`。

## 部署

### 1. 部署 Worker

1. 在 [Cloudflare Dashboard](https://dash.cloudflare.com/) 中新建一个 **Worker**。
2. 将 `workers.js` 的内容粘贴进 Worker 编辑器并保存部署。
3. （可选）为 Worker 绑定自定义域名。

### 2. 部署静态页面（`index.html`）

`workers.js` 中未匹配到代理规则的路径会回源到 `ASSET_URL` 指向的静态资源。因此：

1. 将 `index.html` 部署到任意静态托管（GitHub Pages / Cloudflare Pages 等）。
2. 将 `workers.js` 中的 `ASSET_URL` 改为该静态站点地址，**必须以 `/` 结尾**，然后重新部署 Worker。

> 若页面与 Worker 不同域，还需在 `index.html` 中把 `PROXY_ORIGIN` 填成 Worker 地址。

## 配置说明

### `workers.js`

| 常量 | 默认值 | 说明 |
| --- | --- | --- |
| `ASSET_URL` | `https://geekertao.github.io/gh-proxy/` | 未匹配路径的回源地址（静态页面所在位置），必须以 `/` 结尾。建议改为你自己的静态页地址。 |
| `PREFIX` | `/` | 路由前缀。若自定义路由为 `example.com/gh/*`，需改为 `/gh/`（少一个杠都会出错）。 |
| `Config.jsdelivr` | `0` | 分支文件使用 jsDelivr 镜像的开关，`0` 关闭、`1` 开启。 |
| `Config.rewriteAssetRedirect` | `0` | 签名 CDN 跳转的重写开关。`0`：Worker 内部跟随跳转并直接流式返回；`1`：返回 `302` 由客户端再请求一次代理。 |
| `whiteList` | `[]` | 路径白名单，路径包含其中任一子串才放行；为空表示不限制。 |

### `index.html`

| 常量 | 默认值 | 说明 |
| --- | --- | --- |
| `PREFIX` | `/` | 需与 `workers.js` 的 `PREFIX` 保持一致。 |
| `PROXY_ORIGIN` | `''` | 留空时使用当前页面域名；若页面托管在静态站点、与 Worker 不同域，则填写 Worker 地址，如 `https://your-worker.workers.dev`。 |

## 安全说明

- **主机白名单**：入口链接严格限定在 `github.com`、`raw.githubusercontent.com`、`gist`、`codeload.github.com`、`api.github.com` 及若干 GitHub 资源 CDN 主机，避免主机绕过。
- **错误信息不外泄**：异常堆栈仅记录到服务端日志，对外统一返回固定文案（`cfworker error`）。
- **防索引**：页面 `index.html` 已包含 `<meta name="robots" content="noindex">`，避免被搜索引擎收录。
- **注意**：响应统一返回 `Access-Control-Allow-Origin: *` 且默认无限流，公开部署时存在被滥用消耗配额的风险，建议自行加限流或访问控制。

## 致谢

- [hunshcn/gh-proxy](https://github.com/hunshcn/gh-proxy) —— 核心代理逻辑来源。
- [hubporg/CF-GitHub-Proxy](https://github.com/hubporg/CF-GitHub-Proxy) —— 本项目直接上游。
- [cmliu/CF-Workers-GitHub](https://github.com/cmliu/CF-Workers-GitHub) —— 前端页面参考。

## 许可证

本项目采用 [MIT](LICENSE) 许可证。
