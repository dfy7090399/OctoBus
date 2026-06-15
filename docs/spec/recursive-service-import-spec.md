# Recursive Service Import 技术方案

## 背景与目标

OctoBus 已支持一个 npm-compatible distribution package 包含多个 service root，并可通过
`//service-dir` 选择其中一个 service 导入。`services/` 下的
`@chaitin-ai/octobus-tentacles` 已按这种 multi-service distribution 组织：根
`package.json bin` 暴露多个 service command，每个子 service root 自己包含
`service.json`、proto、schema 和实现文件。

当前用户必须逐个执行：

```bash
octobus service import safeline-waf npm:@chaitin-ai/octobus-tentacles//chaitin__safeline-waf
octobus service import fortinet-fw npm:@chaitin-ai/octobus-tentacles//fortinet__fw
```

目标是在现有 package contract 之上新增批量导入能力：

```bash
octobus service import --recursive npm:@chaitin-ai/octobus-tentacles
```

该命令递归发现 distribution package 内所有 service root，并把每个 service 注册到
OctoBus。导入完成后，`octobus service list` 必须能列出本次发现并导入的所有 service。

## 现状和 harness 约束

- `AGENTS.md` 要求使用 Task 作为主工作流：`task lint`、`task test`、`task build`，
  PR 前运行 `task`；Go 代码必须 `gofmt`、`go vet ./...` clean。
- `AGENTS.md` 要求单元测试、集成测试、e2e 测试各至少 60% 覆盖，整体覆盖至少 90%；
  package import、CLI、daemon 启动、runtime 或 supervision 变化后需要跑相关 e2e。
- `docs/design/product/cli.md` 规定 CLI 资源定位使用位置参数，不使用 `--id` 这类资源
  定位 flag；行为 flag 可表达导入策略。
- `docs/design/technical/multi-service-npm-package.md` 和
  `docs/design/technical/service-package.md` 规定 distribution package root 与 service
  root 分离：依赖安装和 root `package.json bin` 解析都以 distribution package root
  为准，descriptor、schema 和 `service.json` 解析以 service root 为准。
- 现有 admin API 是 `POST /admin/v1/services/import`，单 service import 已负责导入后
  重启该 service 的 enabled long-running instances，并在重启失败时返回 degraded。
- 当前 `internal/packageimport.Importer.Import` 一次只提交一个 `domain.Service`；
  `internal/store.ListServices` 只返回已写入 SQLite 的 services 表记录，不会从 package
  artifact 自动展开。

## 核心概念或领域模型

- recursive import：`service import` 的批量模式。它导入一个 distribution package 中递
  归发现到的多个 service root，而不是导入单个位置参数 service id。
- scan root：递归发现的起点。未指定 `//service-dir` 时为 package root；指定
  `source//some-dir` 时为 package root 下的 `some-dir`。scan root 只限制发现范围，不裁
  剪 artifact，也不改变 dependency install root。
- discovered service：scan root 下包含 `service.json` 的目录。其 package 内相对路径写
  入 `domain.Service.ServiceRoot`。
- service id：recursive import 中由该 discovered service 的 `service.json.name` 决定，
  必须通过现有 `domain.ValidateID("service", id)` 校验，并且同一次 recursive import 内
  不允许重复。
- prepared distribution：一次 fetch/build/unpack/install 后得到的 package、runtime 基
  础目录和 artifact metadata。recursive import 对同一个 source 只准备一次 distribution，
  然后为每个 discovered service 生成各自 descriptor 和 service artifact 目录。

## 架构和组件边界

- CLI 层负责参数形态、互斥校验、source 本地路径规范化和请求 admin API；不直接读取
  package、不写 SQLite。
- Admin API 层负责区分单 service import 与 recursive import，统一日志、安全输出、
  HTTP 状态码和导入后 instance 重启。
- `internal/packageimport` 负责 source 准备、recursive discovery、manifest/bin/schema
  校验、descriptor 编译、artifact/runtime 提交和 store upsert。
- Store schema 不新增表或字段。每个 discovered service 仍写入一条现有 `services` 记录，
  并沿用当前 artifact layout：

```text
{data_dir}/artifacts/services/{service_id}/
  package/
  runtime/
  descriptor.protoset
  <source-artifact>
```

## API、CLI、配置、数据模型或协议变化

CLI 新增形态：

```bash
octobus service import --recursive SOURCE [--build auto|always|never] [--offline] [--reinstall]
```

规则：

- 非 recursive 模式保持现有形态：`octobus service import SERVICE SOURCE ...`。
- recursive 模式只接受一个位置参数 `SOURCE`，不接受 service id。
- recursive 模式禁止 `--name`，因为每个 service 的展示名按现有规则分别来自已有记录、
  `service.json.displayName` 或 `service.json.name`。
- recursive 模式继续支持 `--build`、`--offline`、`--reinstall`。
- 公开命令使用 `--recursive`，不提供 `--all` alias。

Admin API 请求体在现有 `packageimport.Options` 兼容扩展：

```json
{
  "recursive": true,
  "source": "npm:@chaitin-ai/octobus-tentacles",
  "offline": false,
  "reinstall": false,
  "build": "auto"
}
```

单 service import 请求和响应保持兼容。recursive import 成功响应为：

```json
{
  "services": [],
  "service_count": 0,
  "restarted_instances": {
    "service-id": []
  },
  "restart_errors": {
    "service-id": []
  }
}
```

如果 import 已提交但部分 service 的 instance 重启失败，HTTP 状态码为 `409`，响应增加：

```json
{
  "status": "degraded"
}
```

每个 recursive 导入的 service 字段语义：

- `ID`：`service.json.name`。
- `Name`：已有 service 名称优先；否则 `displayName` 优先；否则 `service.json.name`。
- `PackageSource`：规范化 source 加上该 service root，例如
  `npm:@chaitin-ai/octobus-tentacles//chaitin__safeline-waf`。
- `ServiceRoot`：package root 相对 service root，例如 `chaitin__safeline-waf`。
- `NodeEntry`：根 `package.json bin[service.json.name]` 解析出的 package root 相对路径。

Source 规范化要求：

- 本地 source 的 `//service-dir` 必须在 CLI 规范化时保留。
- `./services//chaitin__safeline-waf` 应规范化为
  `/abs/path/services//chaitin__safeline-waf`。
- `npm:./services//chaitin__safeline-waf` 应规范化为
  `npm:/abs/path/services//chaitin__safeline-waf`。
- HTTPS Git source 继续由现有 Git source parser 处理 `//service-dir` 和 `@ref`，并保持凭
  据脱敏。

## 工作流和失败语义

recursive import 的主流程：

1. 获取 source artifact，按 `--build` 策略构建或打包 distribution package。
2. 准备一次 runtime 基础目录和 npm dependencies。
3. 从 scan root 递归发现 discovered services，结果按 service root 稳定排序。
4. 对所有 discovered services 做提交前校验：service id、重复 ID、manifest、root
   `package.json bin` entry、schema 文件、proto descriptor。
5. 只有提交前校验全部成功，才逐个写入 artifact/runtime/descriptor 并 upsert service。
6. 提交完成后，对本次导入的每个 long-running service 重启 enabled instances。

发现规则：

- 含 `service.json` 的目录是一个 service root。
- 发现一个 service root 后不继续深入该目录。
- 递归扫描跳过 `node_modules`、`.git` 和隐藏缓存目录。
- scan root 自身含 `service.json` 时，scan root 本身也是 discovered service。
- scan root 不存在、非法或没有发现任何 service 时，命令失败，不提交 service。

失败语义：

- source 获取、构建、依赖安装、discovery 或提交前校验失败：HTTP 400；不得提交任何
  service 记录或替换已有 service 当前版本。
- 单个 service 的 artifact/runtime/descriptor 提交沿用现有 rollback 保护该 service。
- 首版不提供跨多个 service 的 SQLite/文件系统全量事务；若极少数提交阶段系统错误发生
  在部分 service 已成功提交之后，已提交的 service 保留，错误响应必须明确指出失败项。
- instance 重启失败不回滚已导入 service；返回 HTTP 409 degraded，并按 service id 列出
  restarted instances 和 restart errors。
- Git credential、secret、config 内容不得出现在日志、SQLite、admin API 响应或 CLI 输出。

## 测试、质量门禁和验收标准

必须覆盖：

- CLI 单测：`--recursive SOURCE` 请求体包含 `recursive:true` 且不包含 `service_id`；
  缺 source、多 source、`--recursive --name` 都报错；本地 `//service-dir` 规范化不丢失。
- Importer 单测：从一个多 service package 递归导入多个 service；`source//some-dir`
  只扫描该子树；root 自身为 service root 时可导入；`PackageSource`、`ServiceRoot`、
  `NodeEntry`、schema path 和 methods metadata 正确。
- Importer 失败单测：重复 `service.json.name`、非法 ID、缺 bin entry、坏 schema、坏 proto
  和空发现结果均不提交任何 service。
- Admin / integration 测试：recursive import 后 `service list` 返回全部导入项；重导入保
  留用户改过的展示名；enabled long-running instances 按 service 重启；重启失败返回
  `409 degraded`。
- 安全测试：HTTPS Git credential 在 recursive import 响应、日志和 SQLite 中仍脱敏。
- 服务包侧测试：`services/scripts/import-check-all.mjs` 使用
  `octobus service import --recursive <root>` 验证 `services/` 聚合包可一次导入全部
  service。

质量门禁：

- 变更完成后至少运行相关包测试：`go test ./internal/cli ./internal/admin ./internal/packageimport`。
- 涉及 CLI、package import 和 supervision，完整验收需要运行 `task test`；提交前按
  `AGENTS.md` 要求运行 `task`。
- 若修改 e2e 覆盖路径，运行 `go test ./tests/e2e -count=1`。

验收标准：

- `octobus service import --recursive npm:@chaitin-ai/octobus-tentacles` 成功后，
  `octobus service list` 输出该 package 下所有合法 service。
- 单 service import 的现有请求、响应、runtime、descriptor 和重启语义不回归。
- recursive import 不要求新增数据库 schema 或破坏现有 artifact/runtime layout。

## 首版不做事项

- 不提供 `--all` alias。
- 不提供用户指定 service id 映射文件或 include/exclude 过滤。
- 不自动创建 instance、capset 或 method binding。
- 不把 package root dispatcher `octobus-tentacles` 注册成一个特殊 service。
- 不做跨 service 的强事务回滚机制。
- 不改变 service package manifest schema。
- 不调整 SDK multi-service dispatcher 生成能力；该能力仍属于 SDK roadmap 的后续工作。

## 关键假设和已确认决策

- 用户已确认公开命令使用 `--recursive`。
- 用户已确认 recursive import 的 service id 使用 `service.json.name`。
- 用户已确认 `source//some-dir` 在 recursive 模式下作为递归扫描根。
- 用户已确认采用“提交前预校验”策略：可发现和可校验错误必须在提交前失败并保持零提交。
- 现有 `service.json.name` 与根 `package.json bin` key 一致的 multi-service package 契约
  继续作为批量导入的基础。
