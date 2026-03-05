# Harbor CSP 加固指南

## CSP 是什么
Content Security Policy（CSP）是一组由浏览器执行的安全策略，服务端通过响应头（如 `Content-Security-Policy`）告诉浏览器“哪些来源的脚本、样式、图片、连接、表单提交等资源允许被加载或执行”。

如果未设置 CSP，或仅设置了非常宽松/不完整的策略，常见风险包括：
- XSS 利用面扩大：一旦页面存在注入点，攻击者更容易执行恶意脚本。
- 数据外传风险上升：恶意脚本可向攻击者控制域名发起请求传输敏感信息。
- 第三方资源投毒影响更大：缺少来源约束时，浏览器更容易加载非预期脚本资源。
- 点击劫持防护缺失：若未限制页面嵌入，可能被恶意站点通过 iframe 诱导操作。

## 适用范围
本文总结了当前仓库中 Content Security Policy（CSP）的实现现状，并给出一套兼顾安全性与前端可用性的实用加固方案。

## 当前仓库现状

### 1）仓库内置了 CSP，但策略较简
当使用 Harbor Chart 自带的 `nginx` 作为入口时，渲染出的配置为：

```nginx
add_header Content-Security-Policy "frame-ancestors 'none'";
```

这条配置的含义：
- `frame-ancestors 'none'` 表示当前页面禁止被任何站点以 `frame/iframe/object/embed` 方式嵌入。
- 它主要解决点击劫持问题，是一个有价值但范围很窄的防护点。

为什么它不满足完整 CSP 要求：
- 它没有约束脚本、样式、图片、网络请求等主要资源加载行为（例如 `script-src`、`style-src`、`connect-src`、`img-src`）。
- 它没有提供常见高价值指令（如 `default-src`、`base-uri`、`form-action`、`object-src`）的明确限制。
- 从扫描器视角看，这属于“策略存在但不完整”，因此仍会被判定为弱 CSP 或不满足安全基线。

证据位置：
- [`templates/nginx/configmap-http.yaml:65`](../../templates/nginx/configmap-http.yaml#L65)
- [`templates/nginx/configmap-https.yaml:84`](../../templates/nginx/configmap-https.yaml#L84)
- [`subtree/harbor/make/photon/prepare/templates/nginx/nginx.http.conf.jinja:57`](../../subtree/harbor/make/photon/prepare/templates/nginx/nginx.http.conf.jinja#L57)
- [`subtree/harbor/make/photon/prepare/templates/nginx/nginx.https.conf.jinja:84`](../../subtree/harbor/make/photon/prepare/templates/nginx/nginx.https.conf.jinja#L84)

### 2）未发现独立的 CSP values 开关
在 [`values.yaml`](../../values.yaml) 中未发现 `csp.enabled`、`csp.policy` 这类专用配置项。

社区里有人给 harbor 提过 PR: [Add Content-Security-Policy header templating for portal](https://github.com/goharbor/harbor-helm/pull/2240)

希望能在 Chart 里加上 CSP 相关配置项，但社区成员以 “更倾向于在 Ingress 或外部反代层加 header，而不是塞进 portal 内部的 nginx 中” 而拒绝了。 

### 3）默认部署模式不会使用 Chart 自带 nginx
[`values.yaml:76`](../../values.yaml#L76) 默认值是 `expose.type: ingress`，且 [`values.yaml:556`](../../values.yaml#L556) 的注释已说明 ingress 模式下不会使用 Chart 自带 `nginx`。  
因此在该模式下，CSP 通常需要在 ingress/controller/load balancer 层配置。

## 为什么扫描器仍然会报 CSP 问题
当前内置策略只覆盖了点击劫持防护（`frame-ancestors 'none'`）。  
多数扫描器还会检查以下指令是否显式定义：
- `default-src`
- `script-src`
- `object-src`
- `base-uri`
- `form-action`

## 推荐基线策略（兼容优先，尽量避免渲染异常）
建议先使用以下基线（目标是尽量避免页面渲染异常）：

```text
default-src 'self' https: http: data: blob:;
base-uri 'self';
object-src 'none';
script-src 'self' https: http: data: blob: 'unsafe-inline' 'unsafe-eval';
style-src 'self' https: http: data: blob: 'unsafe-inline';
img-src 'self' https: http: data: blob:;
font-src 'self' https: http: data: blob:;
connect-src 'self' https: http: ws: wss:;
frame-ancestors 'none';
form-action 'self' https: http:;
frame-src 'self' https: http: data: blob:;
manifest-src 'self' https: http: data: blob:;
worker-src 'self' https: http: data: blob:;
```

### 基线策略参数逐项说明
- `default-src 'self' https: http: data: blob:`：默认兜底策略。为降低误拦截风险，放行同源与常见协议来源。
- `base-uri 'self'`：限制 HTML `<base>` 标签只能指向同源，避免攻击者篡改相对链接解析基准。
- `object-src 'none'`：禁止 `object/embed/applet` 等插件内容加载，降低历史插件型攻击面。
- `script-src 'self' https: http: data: blob: 'unsafe-inline' 'unsafe-eval'`：脚本策略采用兼容优先，覆盖 `javascript:` 链接、第三方库运行时行为等潜在场景，减少功能中断概率。
- `style-src 'self' https: http: data: blob: 'unsafe-inline'`：放行内联样式与常见样式来源，兼容 Angular/Swagger UI 的样式加载方式。
- `img-src 'self' https: http: data: blob:`：兼容同源图片、`data:` 图标和 `blob:` 对象 URL。
- `font-src 'self' https: http: data: blob:`：兼容字体静态资源与内联字体数据。
- `connect-src 'self' https: http: ws: wss:`：兼容 API 调用与可能的 WebSocket 连接。
- `frame-ancestors 'none'`：禁止页面被任何站点嵌入，主要用于防点击劫持。
- `form-action 'self' https: http:`：限制表单提交目标到同源/HTTP(S)。
- `frame-src 'self' https: http: data: blob:`：兼容页面可能存在的框架加载场景，降低误杀风险。
- `manifest-src 'self' https: http: data: blob:`：兼容 Manifest 等元资源加载。
- `worker-src 'self' https: http: data: blob:`：兼容 Worker 与 Blob Worker 场景。

说明：
- `style-src 'unsafe-inline'` 在 Angular/组件化样式场景中通常是必要妥协。
- 该基线以“兼容优先”为目标，`script-src` 放宽用于降低上线初期 UI 异常风险。
- 稳定运行后，建议基于 `Report-Only` 告警逐步收紧；`object-src 'none'`、`base-uri 'self'`、`form-action` 可优先保持严格。

### 与“不配置 CSP”相比，这套兼容基线的增益与边界
相对完全不配置 CSP，这套策略仍然提供了以下有效防护：
- `frame-ancestors 'none'`：有效阻断页面被第三方站点嵌入（点击劫持场景）。
- `object-src 'none'`：禁用 `object/embed/applet` 等高风险插件载体。
- `base-uri 'self'`：限制 `<base>` 被恶意篡改，降低相对链接劫持风险。
- 浏览器端建立了最小策略边界，不再是“完全无约束”状态。

同时，这套策略是“兼容优先”，安全能力存在明确上限：
- `script-src` 放宽到 `https/http/data/blob + unsafe-inline + unsafe-eval`，对 XSS 的抑制能力明显弱于严格 CSP。
- `connect-src` 放宽到 `https/http/ws/wss`，外联约束较弱。
- `form-action` 放宽到 `self + https/http`，无法强约束表单只提交到同源。

结论：
- 该策略不会让 CSP “失效”，但其定位不是强防护，而是“先避免渲染异常、再逐步收紧”。
- 如果目标是显著提升 XSS 防护强度，需要在 `Report-Only` 观察后持续收紧 `script-src/connect-src/form-action`。

响应头模式区别：
- `Content-Security-Policy`：强制模式。浏览器会直接拦截违反策略的资源加载或执行，能实际降低 XSS/注入类风险，但策略过严会立即导致页面功能异常。
- `Content-Security-Policy-Report-Only`：观察模式。浏览器不会拦截违规行为，只记录并上报违规事件（开发者工具可见，若配置 `report-uri/report-to` 也可发送到服务端），适合上线前验证策略影响面。
- 推荐顺序：先以 `Content-Security-Policy-Report-Only` 灰度观察并修正策略，再切换到 `Content-Security-Policy` 强制生效。

## 上线策略（推荐）
1. 先启用 `Content-Security-Policy-Report-Only`，观察 3-7 天。
2. 收集浏览器控制台和 ingress/proxy 日志中的违规记录。
3. 仅为真实运行所需来源添加最小例外。
4. 再切换为强制生效的 `Content-Security-Policy`。

## 按暴露方式的配置建议

### Ingress 模式（`expose.type=ingress`）
在 ingress/controller 层配置 CSP。可通过 [`values.yaml:114`](../../values.yaml#L114) 的 `expose.ingress.annotations` 下发控制器特定的响应头配置。

### 非 Ingress 模式（`clusterIP`、`nodePort`、`loadBalancer`）
此时使用 Harbor Chart 自带 `nginx`。  
当前 CSP 在 nginx 模板中为硬编码，未通过 values 提供可直接配置的策略字段。

## 实践原则
生产环境不要一开始就使用“理论上最严格”的 CSP。  
建议采用“默认严格 + 基于证据最小放行”的方法，在不破坏 Harbor UI 的前提下满足安全要求。

## 难度评估与主要卡点（基于 Harbor 源码）

### 难度结论
在“不影响现有页面渲染”的前提下，将 Harbor 调整到“尽可能严格”的 CSP，整体难度为中高到高。  
难点不在于拼出一条 CSP 字符串，而在于覆盖真实运行路径中的前端细节、历史兼容行为和不同部署入口差异。

### 主要卡点
1. `style-src` 很难去掉 `'unsafe-inline'`  
   当前前端包含内联样式和运行时样式注入场景，例如：
   - [`subtree/harbor/src/portal/app-swagger-ui/index.html:9`](../../subtree/harbor/src/portal/app-swagger-ui/index.html#L9)
   - [`subtree/harbor/src/portal/app-swagger-ui/webpack.prod.js:15`](../../subtree/harbor/src/portal/app-swagger-ui/webpack.prod.js#L15)
   - [`subtree/harbor/src/portal/src/app/shared/shared.module.ts:117`](../../subtree/harbor/src/portal/src/app/shared/shared.module.ts#L117)

2. `img-src` 不能只配 `'self'`  
   Artifact 图标会被转换为 `data:` URL 再渲染：
   - [`subtree/harbor/src/portal/src/app/base/project/repository/artifact/artifact.service.ts:75`](../../subtree/harbor/src/portal/src/app/base/project/repository/artifact/artifact.service.ts#L75)

3. Markdown 渲染路径会引入外部资源不确定性  
   README/License 等内容通过 `[innerHTML]` 渲染，若策略过严可能导致内容展示异常：
   - [`subtree/harbor/src/portal/src/app/base/project/repository/artifact/artifact-additions/summary/summary.component.html:3`](../../subtree/harbor/src/portal/src/app/base/project/repository/artifact/artifact-additions/summary/summary.component.html#L3)
   - [`subtree/harbor/src/portal/src/app/base/project/repository/artifact/artifact-additions/license/license.component.html:7`](../../subtree/harbor/src/portal/src/app/base/project/repository/artifact/artifact-additions/license/license.component.html#L7)

4. 文件下载与 Blob URL 场景需要考虑  
   前端存在 `data:` 下载链接和 `blob:` 文件下载：
   - [`subtree/harbor/src/portal/src/app/shared/components/view-token/view-token.component.ts:114`](../../subtree/harbor/src/portal/src/app/shared/components/view-token/view-token.component.ts#L114)
   - [`subtree/harbor/src/portal/src/app/shared/components/operation/operation.service.ts:33`](../../subtree/harbor/src/portal/src/app/shared/components/operation/operation.service.ts#L33)
   - [`subtree/harbor/src/portal/src/app/shared/units/utils.ts:500`](../../subtree/harbor/src/portal/src/app/shared/units/utils.ts#L500)
   - [`subtree/harbor/src/portal/src/app/shared/units/utils.ts:520`](../../subtree/harbor/src/portal/src/app/shared/units/utils.ts#L520)

5. 两套前端资产需统一纳入策略验证  
   Harbor Portal 与 Swagger UI 是独立构建并合并到最终镜像，策略需要同时覆盖：
   - [`subtree/harbor/src/portal/docker-build/Dockerfile:16`](../../subtree/harbor/src/portal/docker-build/Dockerfile#L16)
   - [`subtree/harbor/src/portal/docker-build/Dockerfile:24`](../../subtree/harbor/src/portal/docker-build/Dockerfile#L24)

6. 部署模式决定 CSP 生效层  
   默认 `ingress` 模式下不使用 Chart 内置 nginx，CSP 落点通常在 ingress/controller：
   - [`values.yaml:556`](../../values.yaml#L556)

## 现有 e2e 对 CSP 影响的检出能力评估

### 结论
以当前入口 [`testing/testdata/script/run-harbor-e2e.sh`](../../testing/testdata/script/run-harbor-e2e.sh) 且 `TEST_SUITE=daily` 运行时，基本无法有效检出“CSP 收紧导致 UI 功能异常”的问题。

### 原因分析
1. 脚本当前实际执行的是 Setup + API 测试，而非完整 UI 回归：
   - [`testing/testdata/script/run-harbor-e2e.sh:130`](../../testing/testdata/script/run-harbor-e2e.sh#L130)
   - [`testing/testdata/script/run-harbor-e2e.sh:131`](../../testing/testdata/script/run-harbor-e2e.sh#L131)

2. `Setup.robot` 主要是环境准备与版本检查，UI 覆盖很浅：
   - [`subtree/harbor/tests/robot-cases/Group1-Nightly/Setup.robot:23`](../../subtree/harbor/tests/robot-cases/Group1-Nightly/Setup.robot#L23)
   - [`subtree/harbor/tests/robot-cases/Group1-Nightly/Setup.robot:28`](../../subtree/harbor/tests/robot-cases/Group1-Nightly/Setup.robot#L28)

3. `API_DB_SUCCESS.robot` 主要执行 Python API 用例，不覆盖浏览器渲染链路：
   - [`subtree/harbor/tests/robot-cases/Group0-BAT/API_DB_SUCCESS.robot:21`](../../subtree/harbor/tests/robot-cases/Group0-BAT/API_DB_SUCCESS.robot#L21)

4. `daily` 仅影响当前执行集合中的标签排除，不会自动切换到完整 UI 套件：
   - [`testing/testdata/script/run-harbor-e2e.sh:57`](../../testing/testdata/script/run-harbor-e2e.sh#L57)
   - [`testing/testdata/script/run-harbor-e2e.sh:58`](../../testing/testdata/script/run-harbor-e2e.sh#L58)

5. 仓库里确实存在大量 Selenium UI 用例，但当前脚本未执行这些文件：
   - 示例：[`subtree/harbor/tests/robot-cases/Group1-Nightly/Common.robot:31`](../../subtree/harbor/tests/robot-cases/Group1-Nightly/Common.robot#L31)
   - 官方 full test 示例：[`subtree/harbor/tests/e2e_setup/README.md:60`](../../subtree/harbor/tests/e2e_setup/README.md#L60)

### 实务建议
如果要把 e2e 作为 CSP 变更的回归门禁，至少需要在现有流程中补充完整的 UI 套件执行（不仅是 Setup + API），否则无法可靠发现策略收紧后的前端回归问题。
