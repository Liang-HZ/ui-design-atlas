# RUNBOOK

面向贡献者和"完全不了解这个项目的人/agent"：读完应能独立改内容、本地预览、理解部署是怎么发生的。

## 一、架构（一页）

```
GitHub  Liang-HZ/ui-design-atlas   (main)
   │
   │  Cloudflare Pages · Git 直连（非 GitHub Actions）
   ▼
Cloudflare Pages 项目
   root directory: public/
   build command:  （无）
   │
   ▼
https://ui.liangai.org        →  public/index.html        落地页
https://ui.liangai.org/atlas  →  public/atlas/index.html  表本体
```

**纯静态、无构建、无后端、无数据库、无表单。**

因此这个项目没有备份需求（内容即 git 历史）、不需要人机验证、也不占任何服务器资源。

### 为什么不用 GitHub Actions 部署

1. 没有构建步骤 —— 起一个 CI runner 只为把两个 HTML 文件传上去，纯浪费
2. Git 直连不需要 API token，**少一个 secret 就少一个泄漏面**

**本仓库不需要任何 GitHub secret。** 如果将来引入构建步骤（比如把内容拆成 markdown 再生成 HTML），再改用 Actions + wrangler，届时才需要配部署凭据。

## 二、改内容

内容是**两个手写 HTML 文件**，没有构建步骤、没有编译产物：

| 文件 | 是什么 |
|---|---|
| `public/index.html` | 落地页 |
| `public/atlas/index.html` | 表本体（单文件，CSS 和 JS 都内联） |

直接改，改完就是最终效果。本地预览：

```bash
python3 -m http.server 8899 --directory public
# 落地页 → http://localhost:8899/
# 表本体 → http://localhost:8899/atlas/
```

### 改完必须自检

这份表自己讲的规则，它自己要遵守。四条，缺一不可：

1. **明暗双主题**下所有文字对比度 ≥ 4.5:1（表里的 12 号）
2. **1280px 和 375px** 两个尺寸下，页面**不出现横向滚动**（宽内容要在自己的容器里滚，表里的 24 号）
3. 浏览器控制台**无报错**
4. 交叉引用**无断链**（表里有 90+ 个 `#aNN` 跳转）

两个已经踩过的坑，改的时候注意：

- **grid 子项默认 `min-width:auto`**。`.atom-body` 若不置 0，里面 `min-width` 较大的演示会把整页顶宽，即使演示容器自己有 `overflow-x:auto` 也没用。
- **顶部导航是 sticky（约 37px）**。锚点跳转需要 `scroll-margin-top` 大于它，否则目标顶部会被盖住。文件里用的是 `html [id]{scroll-margin-top:52px}` —— 用 `html [id]` 而不是 `[id]`，是因为后者与下方 `.atom{scroll-margin-top:20px}` 特异性相同，而后者在源码里更靠后会赢。

## 三、部署与回滚

推到 `main` 即自动部署，无需手动操作：

```bash
git push origin main
```

Cloudflare 面板 → Workers & Pages → 本项目 → Deployments 可看到每次部署记录。

**回滚**：Deployments 列表里选中上一个成功版本 → `Rollback to this deployment`。秒级生效，不需要 revert commit。

## 四、冒烟测试

**必须断言这个站点独有的内容，不能只断言 HTTP 200。**

原因：`liangai.org` 配了泛解析，**未正确绑定的子域名不会返回 404，而是返回另一个服务的页面（同样是 200）**。只看状态码会假阳性通过，让人以为上线成功了。

```bash
# 落地页
curl -s https://ui.liangai.org/ | grep -q '把「感觉不对」' && echo "landing OK" || echo "landing FAIL"

# 表本体
curl -s https://ui.liangai.org/atlas | grep -q '30 个可以指名道姓的东西' && echo "atlas OK" || echo "atlas FAIL"
```

判断"是不是压根没配记录"：编一个不存在的子域名去 curl，如果它返回的东西和你的域名一样，说明你的域名也只是没绑定。

```bash
curl -s https://definitely-does-not-exist-9x7q.liangai.org | grep -o '<title>[^<]*</title>'
```

## 五、统计

用自托管 Umami，脚本在两个页面的 `<head>` 里。website-id 直接写在 HTML 中（它本来就会发到浏览器，不是密钥）。

### 埋点事件（只埋两个，不过度设计）

| 事件名 | 触发 | 回答什么问题 |
|---|---|---|
| `open-atlas` | 落地页点「打开图表」 | 落地页转化率 |
| `open-github` | 落地页点「在 GitHub 上看」 | 有多少人想看源码 / 想 Watch |

### 北极星

**盯 GitHub Watch 数，不是 Star 数。**

Star 是"我觉得不错"，是一次性情绪，收藏夹里再也不打开；Watch 是"我要持续跟着它"，代表用户认为它**未来还会有价值**。Star 涨得快但骗人，Watch 慢但诚实。

> 面板地址、凭据位置、建站点的操作路径等基础设施细节，见私有运维仓库，不放在这里。

## 六、常见故障

| 现象 | 多半原因 | 处理 |
|---|---|---|
| 域名返回了别的服务的页面 | Custom domain 没绑，走了泛解析 | 去 Pages 项目绑 Custom domain |
| 域名返回 522 | 刚绑完还没部署过 | 推一次 commit 触发部署 |
| 推了没部署 | Pages 项目的 production branch 不是 `main`，或 Git 授权失效 | 面板 Settings → Builds & deployments 核对分支；重新授权 GitHub |
| 页面中文乱码 | HTML 缺 `<meta charset="utf-8">` | 两个页面的 `<head>` 里都必须有 |
| 统计面板没数据 | 站点记录不存在，或域名不匹配 | 核对面板里的站点域名与实际访问域名一致 |
| 交互演示不动 | JS 报错 | 打开控制台看；表本体是单文件，JS 在文件末尾的 IIFE 里 |
| 锚点跳过去被导航栏盖住 | `scroll-margin-top` 被更靠后的规则覆盖 | 见第二节第 2 条 |

## 七、上游

- **`DESIGN.md` 规范**：[google-labs-code/design.md](https://github.com/google-labs-code/design.md)（Apache-2.0，**alpha 阶段**）
  第四部里那份可抄的示例对齐的就是它。规范字段变动时，这份表需要跟进 —— 这是本项目最主要的更新来源。
