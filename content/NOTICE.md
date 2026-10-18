# SnapOtter（cloud.lazycat.app.snapotter1）打包说明

- 上游项目：SnapOtter — https://github.com/snapotter-hq/SnapOtter
- 上游版本：v2.2.0（commit `ad2056b9869954d6f35dd471bc6b5673ca0b50ff`）
- 许可证：**AGPL-3.0**（`LICENSE`）。注意仓库里 `packages/enterprise/` 是
  **SnapOtter Commercial License**（`packages/enterprise/LICENSE`），
  明确「不受 AGPL-3.0 覆盖；未经商业协议不得用于生产、不得作为服务提供、**不得再分发**」。

## 本包使用的镜像（自建，已移除商业许可目录）

`snapotter/snapotter:2.2.0` 官方镜像里**包含** `packages/enterprise/`
（官方 `docker/Dockerfile` 第 607-612、710 行显式 `COPY packages/enterprise/...`，
注释写明「packages/enterprise is required for ALL commercial features」），
直接中转该镜像等于再分发商业许可代码。因此本包改为：

1. 以官方镜像为底，`RUN rm -rf /app/packages/enterprise` 删除该目录；
2. 把结果 **flatten 成单层**（`docker export` + `docker import`），
   使镜像的**任何一个层**都不再包含该目录（普通 `RUN rm` 派生镜像仍会把原层带上）；
3. 推到公开镜像源后由 `lzc-cli appstore copy-image` 转入懒猫镜像源，
   清单里使用的是 copy-image 实际返回的 `registry.lazycat.cloud/...` 地址。

社区版功能不依赖该目录（上游 `LICENSING.md`：「The community edition is fully functional
without any code from packages/enterprise」），因此删除后应用功能不变，
企业特性（授权校验、S3 存储、审计归档等）自然不可用。

- 图标：取自上游 `branding/logo-512.png`，仅做无损尺寸与调色板压缩。
- 未改动上游源码；未修改任何业务文件。

## 打包侧改动

1. 关闭匿名产品分析与错误监控（`SNAPOTTER_TELEMETRY=0`，Sentry/PostHog 不再上报）。
2. 免密登录：`AUTH_ENABLED=false`，访问控制交给微服账号体系（网关 SSO），
   前端读 `/api/v1/config/auth` 后不再跳登录页。
3. 界面默认简体中文（`DEFAULT_LOCALE=zh-CN`）。
4. 数据卷 `/lzcapp/var/data` 挂到容器 `/data`，内嵌 PostgreSQL 17、Redis 8、
   上传文件与按需模型全部持久化。
5. **网盘文件拦截**（官方《自动拦截文件选择器》专题）：
   - `lazycat-file-bridge`（request）：同源桥接 `/__lazycat_file_bridge/*` → `file.<微服域名>`；
   - `open-save-chooser`（browser）：官方 `lzc-file-chooser-inject.js`，接管
     `input[type=file]`、`showOpen/SaveFilePicker` 与 blob 下载锚点；
   - `snapotter-http-download-bridge`（browser）：补接应用结果下载用的
     「同源普通 HTTP 下载锚点」（官方钩子只认 `blob:`/`data:`）。
