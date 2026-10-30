# dsh-pause

> **已停止更新**。这个功能已经并入合并版插件 [dsh-toolkit](https://github.com/MSSZSD3925/dsh-toolkit)，
> 后续的修复与新功能都只在那一处维护。本仓库保留历史代码，仅供查阅，不再接受改动。

给 DSH 加一个「暂停」按钮：**你点一下，我在下一个能停的地方停住，等你说完再继续。**

## 为什么需要它

插话（运行中发消息）确实会在"下一个步进边界"被取走，但边界什么时候到不由你决定 ——
我可能一口气跑几分钟的工具链。你看到思考发现不对，却来不及打完字。
这个插件把边界变成"先停住、时间归你"：点一下按钮置位，我在下一个边界停下，
你慢慢打字；发出来（或点「继续」）我就走。

平时它**完全不干扰** —— 没点按钮时钩子直接放行。

## 用法

重启 DSH 后，界面**左下角**会出现一个小条：「暂停」按钮 + 状态文字（点过之后还会出现「继续」）。

- 点「暂停」→ 状态显示"已请求暂停，等这一步到边界" → 我在下一个 `agent/pre-step` 停住 → 显示"已停住，等你说话"
- 你发一条消息 → 自动继续（也可以用「继续」按钮，或等 `waitSeconds` 超时）

## 配置（可选）

默认值够用。要改就在 profile 的 patch 里给自己的插件条目加 config：

```yaml
- insert:
    - id: dsh-pause
      name: dsh-pause
      config:
        routePrefix: /dsh-pause   # 路由前缀
        waitSeconds: 0            # 0 = 一直等；>0 = 该秒数后自动继续
```

## 三个挂载点（都是只读解包 + 实机验证得到的，不是猜的）

1. **暂停点**：`dsh-agent-loop` 的 `agent/pre-step` 瀑布钩子
   （监听器签名 `async (payload, next) => …`，`await` 住就是停住，`next()` 就是放行）。
2. **路由注册**：`ctx.webServer.register` 收的是**路由对象**
   `{ kind: 'exact', path, handler }` —— 传裸函数会静默不注册任何路由（端点一直 404）。
3. **界面注入**：桌面端唯一通道是 `webserver/index-inject` 的结构化行
   （`tapIndex` 在桌面端不生效，因为壳子的 index.html 从安装包静态读盘）；
   必须在 `apply()` 里**第一时间**注册（注入表由宿主启动时一次性收集）；
   行类型用 `kind: 'script'` 内联，且由内联脚本自己建 `<script src>` 并吞掉 `onerror`
   （用 `script-src` 一旦加载失败会 reject `__DSH_BOOT_READY__`，整个应用起不来）。

## 安装（本地 link）

```bash
# profile 目录（如 ~/.dsh/profiles/desktop）里
pnpm install                 # 让 pnpm 把 link: 依赖写进锁文件
```
profile 的 `package.json` 需要三处：`dependencies` 里的 `link:` 路径、
`dsh.profile.bundles` 里加插件名、以及插件自己的 `dsh.bundle.patch` 指向 `cordis.patch.yml`。

**只建 node_modules 链接、不跑 pnpm install，插件不会被加载** —— 加载器按 pnpm 记的依赖图解析。