# Harbor Issues 分析报告

**生成日期**: 2026-04-22
**分析时间范围**: 2026-04-15 至 2026-04-22
**Issue 总数**: 33 个

---

## 一、关键问题分析

### 1. 并发标签创建竞态条件 (Critical)
**Issue**: [#23118](https://github.com/goharbor/harbor/issues/23118) - Race Condition in Concurrent Tag Creation Causes Silent Tag Update Failure

**问题描述**:
当两个进程同时推送不同的镜像到同一个新标签（如 `latest`）时，存在严重的竞态条件。第二个推送会遇到唯一约束冲突，但错误被静默吞掉，导致标签指向错误的镜像。

**影响**:
- **数据损坏**: 静默的数据不一致，客户端认为推送成功但实际标签指向错误镜像
- **代理缓存失效**: `proxy.controller.EnsureTag` 收到 `tagID=0`，导致代理拉取完全失败
- **CI/CD 风险**: 并发推送常用标签（`latest`, `main`）时违反"最后推送获胜"的预期

**建议修复**: 在冲突时重试 `Ensure` 操作，正确更新现有标签。

---

### 2. 在线 GC 竞态条件导致数据丢失 (Critical)
**Issue**: [#23116](https://github.com/goharbor/harbor/issues/23116) - Critical Race Condition in Online GC Leading to Data Loss

**问题描述**:
在线垃圾回收（GC）与并发镜像推送/拉取操作之间存在竞态条件。GC 可能删除刚刚被"触及"或正在关联的 blob，导致持久化数据丢失。

**根因分析**:
1. GC 使用 2 小时安全窗口判断 blob 是否可删除
2. 当用户推送使用相同基础层的新镜像时，`project_blob` 记录更新但 `blob.update_time` 不更新
3. GC 可能删除仍在使用的 blob

**影响**:
- **数据丢失**: 持久化 blob 被删除
- **镜像损坏**: 镜像无法拉取
- **生产环境高风险**: 影响使用在线 GC 的生产注册表

---

### 3. Docker 代理缓存在 v2.15 中损坏 (High)
**Issue**: [#23145](https://github.com/goharbor/harbor/issues/23145) - docker proxy cache broken in v2.15

**问题描述**:
在 Harbor v2.15 中配置 Docker Hub 代理缓存时，拉取操作失败，返回 "not found" 错误。错误日志显示上游返回 403 Forbidden。

**临时解决方案**: 降级到 v2.14.3 可正常工作。

---

### 4. Docker 登录失败 (High)
**Issue**: [#23138](https://github.com/goharbor/harbor/issues/23138) - docker login failing in v2.14.x due to new Host request header behaviour

**问题描述**:
从 v2.13.2 升级到 v2.14.x 后，OIDC 重定向 URL 在 `docker login` 时失败。问题是 PR #21898 引入的变化导致 token 服务 URL 计算方式改变。

**影响**: 无法使用 OIDC 登录 Docker。

---

### 5. CSRF Token 问题导致镜像不可见 (High)
**Issue**: [#23001](https://github.com/goharbor/harbor/issues/23001) - CSRF token not found in request

**问题描述**:
Harbor v2.13.x 中，注册表通知端点从 `/service/notifications` 移动到 `/registry/notifications`，但 CSRF 中间件跳过器未更新。

**影响**:
- 推送的镜像不显示在 Web UI 或 API 中
- `harbor-registry` 发送的通知被 CSRF 保护拒绝

---

## 二、性能问题

### 6. GC 删除 API 性能问题 (High)
**Issue**: [#22995](https://github.com/goharbor/harbor/issues/22995) - perf: slow delete API caused by full table scan

**问题描述**:
`FindBlobsShouldUnassociatedWithProject` 查询在大型环境（300万+ artifacts）中执行全表扫描，导致删除 API 响应时间长达 97 秒/每个 blob。

**优化建议**: 使用 `LIMIT 1` 子查询，将查询时间从 97 秒降至 0.117 毫秒。

---

## 三、UI/UX 问题

### 7. 仓库列表状态丢失
**Issue**: [#23142](https://github.com/goharbor/harbor/issues/23142)

**问题**: 浏览仓库时，导航到仓库详情后返回，页码和搜索条件丢失。

**建议**: 使用 URL 查询参数或会话存储保存状态。

---

### 8. 标签项重叠
**Issue**: [#23110](https://github.com/goharbor/harbor/issues/23110)

**问题**: 当镜像有大量标签时，UI 布局破坏，标签项重叠。

---

### 9. 复制 Artifact 链接失败
**Issue**: [#23115](https://github.com/goharbor/harbor/issues/23115)

**问题**: v2.15 中从 UI 复制 artifact 链接不工作。

---

### 10. SBOM 详情页面空白过多
**Issue**: [#23111](https://github.com/goharbor/harbor/issues/23111)

**问题**: SBOM 详情页面显示过多空白，影响空间利用率。

---

## 四、功能请求

### 11. 基于上下文的管理员访问控制
**Issue**: [#23140](https://github.com/goharbor/harbor/issues/23140)

**请求**: 根据网络上下文（源 IP/子网）动态限制管理员权限，实现零信任安全模型。

---

### 12. 复制失败监控通知
**Issue**: [#23126](https://github.com/goharbor/harbor/issues/23126)

**请求**: 支持仅在复制失败时发送 Webhook 通知（如 Slack）。

---

### 13. Robot 账户 IP ACL
**Issue**: [#23092](https://github.com/goharbor/harbor/issues/23092)

**请求**: 为 Robot 账户添加可选的 IP/CIDR 白名单限制。

---

### 14. 代理缓存镜像过滤
**Issue**: [#13231](https://github.com/goharbor/harbor/issues/13231)

**请求**: 为代理缓存添加镜像过滤/白名单功能，减少复制规则配置。

---

## 五、其他问题

### 15. 仓库删除失败
**Issue**: [#23127](https://github.com/goharbor/harbor/issues/23127)

**问题**: 删除包含 OCI Image Index 的仓库时，级联删除子 artifacts 后迭代删除失败。

---

### 16. 代理缓存无 HTTP 超时
**Issue**: [#23104](https://github.com/goharbor/harbor/issues/23104)

**问题**: 代理缓存认证请求（auth scheme detection 和 token fetching）没有配置超时，可能导致请求无限挂起。

---

### 17. 仓库更新时间未更新
**Issue**: [#23149](https://github.com/goharbor/harbor/issues/23149)

**问题**: 推送或删除标签时，仓库和 artifact 的 `update_time` 未更新。

---

### 18. CSRF 安全漏洞缓解
**Issue**: [#22312](https://github.com/goharbor/harbor/issues/22312)

**问题**: 建议将 CSRF 中间件从 `github.com/gorilla/csrf` 迁移到 `filippo.io/csrf/gorilla` 以缓解 CVE-2025-24358 和 CVE-2025-47909。

---

### 19. in-toto 层扫描问题
**Issue**: [#22408](https://github.com/goharbor/harbor/issues/22408)

**问题**: 包含 in-toto 层的 manifest 未标记为"不支持扫描"，导致在配置了漏洞预防的项目中无法正常拉取。

---

### 20. ECR 公共镜像端点支持
**Issue**: [#22346](https://github.com/goharbor/harbor/issues/22346)

**问题**: 当前 ECR 正则表达式不支持公共镜像端点（如 `ecr-public.us-east-1.amazonaws.com`）。

---

## 六、已关闭问题

### 21. GC 任务信息单位不一致
**Issue**: [#23122](https://github.com/goharbor/harbor/issues/23122) - **已关闭**

前端以 KB 显示 GC 大小信息，而日志以 MB 显示，造成混淆。

---

### 22. SBOM 创建时间无效
**Issue**: [#23112](https://github.com/goharbor/harbor/issues/23112) - **已关闭**

API 返回 SBOM 创建时间为零值。

---

### 23. AWS IAM 角色 S3 存储问题
**Issue**: [#23139](https://github.com/goharbor/harbor/issues/23139) - **已关闭**

Docker Compose 部署在 EC2 上无法使用实例 IAM 角色。

---

### 24. 项目配额列表不完整
**Issue**: [#22864](https://github.com/goharbor/harbor/issues/22864) - **已关闭**

配额页面只显示部分项目。

---

### 25. 复制目标命名空间问题
**Issue**: [#22715](https://github.com/goharbor/harbor/issues/22715) - **已关闭**

不指定命名空间时复制到 ECR 失败。

---

### 26. GetByUsername 上下文问题
**Issue**: [#22653](https://github.com/goharbor/harbor/issues/22653) - **已关闭**

OIDC 用户查询未考虑上下文取消，导致 PostgreSQL 连接耗尽。

---

### 27. Immutable Artifact 删除问题
**Issue**: [#22543](https://github.com/goharbor/harbor/issues/22543) - **已关闭**

标记为 immutable 的 artifact 无法通过 UI 删除。

---

## 七、社区/推广相关

### 28. ACMM 徽章
**Issue**: [#23146](https://github.com/goharbor/harbor/issues/23146)

KubeStellar Console 提供 AI 能力成熟度模型（ACMM）评估，Harbor 得分 12 项标准，建议添加徽章以展示 AI 原生项目成熟度。

---

### 29. Harbor 安装引导任务
**Issue**: [#23006](https://github.com/goharbor/harbor/issues/23006)

KubeStellar Console 提供 Harbor 安装引导任务，支持实时验证和故障排除。

---

## 八、长期未解决问题

### 30. AWS IAM 角色支持
**Issue**: [#16490](https://github.com/goharbor/harbor/issues/16490) - **自 2022 年 3 月开启**

EKS 上使用 IAM 角色 for Service Accounts (IRSA) 的支持问题。

---

### 31. 代理缓存项目镜像列表不显示
**Issue**: [#19609](https://github.com/goharbor/harbor/issues/19609) - **自 2023 年 11 月开启**

代理缓存项目中的镜像实际存在但不在仓库列表中显示。

---

### 32. Artifactory 标签删除
**Issue**: [#22714](https://github.com/goharbor/harbor/issues/22714)

从 Harbor 复制到 Artifactory 时，仅删除标签（非 artifact）失败。

---

### 33. 配置文件支持代理缓存白名单
**Issue**: [#22957](https://github.com/goharbor/harbor/issues/22957)

请求在 `harbor.yml` 中添加 `PERMITTED_REGISTRY_TYPES_FOR_PROXY_CACHE` 和 `REPLICATION_ADAPTER_WHITELIST` 配置。

---

## 总结

本周 Harbor 共有 **33 个活跃 Issue**，其中：

- **Critical 级别**: 2 个（并发竞态条件问题）
- **High 级别**: 5 个（功能损坏、性能问题）
- **功能请求**: 4 个
- **UI/UX 问题**: 4 个
- **已关闭**: 7 个
- **长期未解决**: 4 个

**建议优先处理**:
1. #23118 和 #23116 - 并发竞态条件可能导致数据损坏或丢失
2. #23145 - v2.15 代理缓存功能损坏
3. #23001 - CSRF 问题导致镜像不可见
4. #22995 - GC 性能问题严重影响大型部署

---

*报告由 AI 自动生成，仅供参考。*
