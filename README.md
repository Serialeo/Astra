# Astra — 发布分支

这条分支只放构建产物，不含源码。

职责分开，两个远端各只放一样东西：

| | 放什么 | 在哪 |
|---|---|---|
| 源码 | `main` | `origin` —— git.iphyxel.com/zwang/Astra |
| 产物 | `release` | `github` —— github.com/Serialeo/Astra |

- `astra.js` —— 打包好的 ESM 模块，自包含（不依赖宿主提供任何模块）。
- `version.json` —— 本次产物对应的源码提交与构建时间。

`pnpm release` 只做一件事：把 `release` 分支**强制重写**并推到 `github`。origin 上不留产物副本，GitHub 上也不放源码。

## 在酒馆助手里使用

```js
await import('https://cdn.jsdelivr.net/gh/Serialeo/Astra@<release 提交 SHA>/astra.js');
```

模块自己会在 DOM 就绪时挂载控制台面板（宿主页右下角的星形按钮）。

**地址里那串 SHA 就是版本号，每次发布都会变，`pnpm release` 结束时会打印当前的那条。**

## 为什么走 jsDelivr，而不是直接指主仓库

浏览器跨域取这个文件时，提供文件的服务器必须发两个头，缺一不可：

| | `Access-Control-Allow-Origin` | `Content-Type` |
|---|---|---|
| git.iphyxel.com raw | **无** | `text/plain` |
| raw.githubusercontent.com | `*` | `text/plain` |
| **jsDelivr** | `*` | `application/javascript` |

- 缺 **ACAO**：浏览器在把响应交给页面之前就拦掉（`blocked by CORS policy`）。
  这一层**客户端无解**——fetch、Blob、任何写法都拿不到数据，不是代码能绕的。
- 缺 **正确 Content-Type**：`import()` 按 HTML 规范拒绝执行
  （`Expected a JavaScript-or-Wasm module script but the server responded with
  a MIME type of "text/plain"`）。

git.iphyxel.com 两样都缺，且要补得动反向代理配置。jsDelivr 两样都给，但只认 GitHub
与 npm 作为来源——**产物放在 GitHub 就是为了这个，没有别的原因**。那个仓库不放源码，
只是一个拿响应头用的分发点。

（如果哪天给 git.iphyxel.com 的反代补上了这两个头，就可以直接
`await import('https://git.iphyxel.com/zwang/Astra/raw/branch/release/astra.js')`，
GitHub 这一环连同 `release` 分支一起就都不需要了。）

## 发布地址钉 commit

接入地址里用 `release` 分支的**提交 SHA**，不用 `@release` 这类分支或标签引用。

钉 commit 的内容是不可变的：jsDelivr 命中即正确，推完立刻能取到，也不需要 `?t=`
——不可变内容没有"拿到旧版"这回事。SHA 每次发布都变，它同时就是"我现在跑的是哪一版"
的答案。`pnpm release` 结束时会打印当前该贴的完整一行。

分支与标签引用走的是另一套：jsDelivr 会缓存"引用 → commit"的解析结果，而 `purge`
只作用于文件路径、不作用于这层解析。因此凡是会被重写的引用（这条 release 分支每次
发布都强制重写成新的孤儿提交），都不能用来验证刚发布的改动。

## 判断一次发布是否真的到位

推送成功 ≠ 线上可取到。判据只有一条：**从 CDN 把钉住的 `version.json` 取回来，
核对 `commit` 等于本次源码提交**。

`pnpm release` 已内置这一步，通过时打印：

```
▸ 校验线上产物
  线上产物已对上源码提交 <shortCommit>
```

对不上会中止发布并返回非零退出码。手工核对用下一节的代码。`purge` 接口返回的 `"status": "finished"`
不构成发布到位的证据，不要拿它当判据。

## 检查当前跑的是哪一版

```js
const v = await (await fetch(
  'https://cdn.jsdelivr.net/gh/Serialeo/Astra@<release 提交 SHA>/version.json',
)).json();
console.log(v.shortCommit, v.builtAt);
```

## 两个仓库都必须公开

`import()` 是不带凭据的匿名请求；jsDelivr 也只能取公开仓库。任一改成私有，酒馆端即失败。
