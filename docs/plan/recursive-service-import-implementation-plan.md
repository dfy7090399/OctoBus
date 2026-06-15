# Recursive Service Import 实施计划

## 阶段 1：固定公共契约和测试夹具

目标：先把 recursive import 的公开契约、响应形态和测试夹具固定下来，为后续实现提供可验证边界。

依赖：

- 已确认 spec：`docs/spec/recursive-service-import-spec.md`。
- 现有 CLI 契约：`docs/design/product/cli.md` 中资源定位使用位置参数，行为使用 flag。
- 现有 multi-service package 契约：`docs/design/technical/multi-service-npm-package.md` 和 `docs/design/technical/service-package.md`。

实施工作：

1. 在 `internal/packageimport.Options` 中新增 `Recursive bool json:"recursive"`，不改变现有字段含义。
2. 定义 recursive import 返回结构，例如 `RecursiveResult`，包含 `Services []domain.Service`、按 service id 聚合的重启信息和可供 admin 响应使用的统计字段。
3. 在 `internal/packageimport/importer_test.go` 中补充多 service fixture helper，能生成：
   - package root `package.json`，含多个 bin entry。
   - 多个 service root，各自有 `service.json`、proto、config schema、secret schema。
   - 嵌套目录、空目录、`node_modules`、`.git`、隐藏目录等 discovery 干扰项。
4. 在 `internal/cli/cli_test.go`、`internal/admin/admin_test.go` 中先新增目标行为测试样例，允许初始失败，用于驱动后续阶段实现。

测试和验证：

- 运行 `go test ./internal/packageimport ./internal/cli ./internal/admin`，此阶段预期新增测试会在实现前失败。
- 不运行格式化写入命令之外的无关重构。

验收标准：

- 新增类型和测试表达 spec 中的 `--recursive SOURCE`、`recursive:true`、响应聚合和零提交失败语义。
- 单 service import 现有测试未被删除或弱化。

适用 harness 约束或命令：

- Go 文件必须保持 `gofmt` 格式。
- 相关测试位于被测 package 旁边，符合 `AGENTS.md` 测试布局要求。

## 阶段 2：实现 source 规范化和 recursive discovery

目标：让 CLI 和 importer 能正确理解 `source//scan-root`，并递归发现 package 内 service root。

依赖：

- 阶段 1 的类型和 fixture。
- 现有 `splitSourceServiceRoot`、`cleanServiceRoot`、`sourceWithServiceRoot`、`parseGitSource`。

实施工作：

1. 重构 CLI 的 `normalizeImportSource`，对非 HTTPS Git source 先拆分 `//service-dir`，只规范化 package root，再拼回 service dir。
2. 保持 HTTPS Git source 原样传给 admin API，继续由 Git parser 处理 `//service-dir` 和 `@ref`。
3. 在 `internal/packageimport` 中新增 discovery helper：
   - 输入 `packageDir` 和 scan root。
   - 输出按 package root 相对路径排序的 service roots。
   - 命中 `service.json` 后不再继续深入该目录。
   - 跳过 `node_modules`、`.git` 和隐藏目录。
   - scan root 自身含 `service.json` 时返回 scan root。
4. discovery 对 scan root 不存在、非法、空发现结果返回明确错误。

测试和验证：

- `go test ./internal/cli ./internal/packageimport`。
- 覆盖 `./pkg//nested`、`npm:./pkg//nested`、HTTPS Git source 不被 CLI 拆坏、空发现、嵌套发现、跳过目录等用例。

验收标准：

- 本地 source 规范化保留 `//service-dir`。
- recursive discovery 的输出稳定、可预测，不依赖文件系统遍历顺序。
- 单 service `splitSourceServiceRoot` 既有行为不回归。

适用 harness 约束或命令：

- `task lint` 中的 gofmt 和 `go vet ./...` 不应因本阶段新增 helper 失败。

## 阶段 3：实现 importer recursive 核心流程

目标：在 `internal/packageimport` 中实现一次准备 distribution、多 service 预校验、再逐个提交的核心导入流程。

依赖：

- 阶段 2 的 source 规范化和 discovery。
- 现有 `Importer.Import`、`prepareSource`、`buildSourcePackage`、`prepareRuntime`、`descriptors.Compile`、`replaceServiceDir`。

实施工作：

1. 新增 `Importer.ImportRecursive(ctx, opts Options) (RecursiveResult, error)`。
2. 抽取单 service import 中可复用的内部 helper：
   - 从 service root 读取并校验 manifest。
   - 从 root `package.json bin` 推断 node entry。
   - 校验 config / secret schema。
   - 编译 descriptor。
   - 组装 `domain.Service`。
   - 提交 artifact、package、runtime、descriptor 并 upsert store。
3. recursive import 中只执行一次 source prepare、build、runtime dependency preparation 和 local SDK replacement。
4. 对所有 discovered services 做提交前预校验：
   - `service.json.name` 作为 service id，并调用 `domain.ValidateID`。
   - 同一批次内 service id 不重复。
   - bin entry、schema、proto descriptor 全部可用。
5. 预校验全部成功后，按 service root 排序逐个提交。
6. 每个提交后的 `PackageSource` 使用原始规范化 source 加该 service root；`ServiceRoot` 使用 package root 相对路径；展示名沿用现有 `importServiceName` 规则。

测试和验证：

- `go test ./internal/packageimport`。
- 覆盖成功导入多个 service、scan root 子树导入、root service 导入、重导入保留展示名、schema path、descriptor methods、node entry、package source。
- 覆盖重复 ID、非法 ID、缺 bin entry、坏 schema、坏 proto、空发现结果均不提交任何 service。

验收标准：

- recursive import 成功后 store 中存在多个 service 记录，artifact/runtime/descriptor layout 与单 service import 一致。
- 提交前可发现和可校验错误保持零提交。
- 不新增数据库 schema。

风险和停止条件：

- 如果为“只准备一次 runtime”引入大范围复制逻辑且破坏现有单 service import，应停止并先拆出最小共享 helper，确保单 service import 测试全部通过。
- 如果发现无法在文件系统层面保证跨 service 强事务，不实现强事务；按 spec 保留提交阶段部分成功的错误上报。

适用 harness 约束或命令：

- 聚焦测试：`go test ./internal/packageimport`。
- 变更不得降低 `scripts/test-coverage.sh` 可统计的 package 覆盖。

## 阶段 4：接入 Admin API 和重启编排

目标：让 daemon 的现有 import endpoint 同时支持单 service 和 recursive import，并沿用安全输出和 instance 重启语义。

依赖：

- 阶段 3 的 `ImportRecursive`。
- 现有 `handleServiceImport` 和 `restartEnabledServiceInstances`。

实施工作：

1. 在 `handleServiceImport` 中按 `req.Recursive` 分流：
   - `false` 走现有单 service import 响应。
   - `true` 调用 `Importer.ImportRecursive`。
2. recursive 请求校验：
   - `source` 必填。
   - `service_id` 必须为空。
   - `name` 必须为空。
3. 为 recursive import 增加日志字段，避免记录 secret/config 内容和未脱敏 credential。
4. 对 recursive 结果中的每个 service 调用 `restartEnabledServiceInstances`，按 service id 聚合 `restarted_instances` 和 `restart_errors`。
5. 只要任一 service 有重启错误，返回 HTTP 409 和 `status:"degraded"`；否则返回 HTTP 200。
6. 保持单 service import 的响应 JSON 和状态码完全兼容。

测试和验证：

- `go test ./internal/admin`。
- 覆盖 recursive 成功响应、请求校验错误、重启 degraded 响应、on-demand service 不重启、日志不泄露 credential。
- 继续覆盖现有单 service import 和 restart failure 测试。

验收标准：

- `POST /admin/v1/services/import` 可以用 `recursive:true` 导入多 service。
- 单 service import API 客户端不需要修改。
- Git credential、secret、config 不出现在日志、SQLite 或响应中。

适用 harness 约束或命令：

- `AGENTS.md` 安全要求：不提交日志、数据目录、artifact 或 secrets。
- CI 会运行 `go test ./cmd/... ./internal/...`，本阶段必须满足 internal 测试。

## 阶段 5：接入 CLI 和服务包验证脚本

目标：提供用户可用的 `octobus service import --recursive SOURCE`，并让 services 聚合包验证脚本使用新能力。

依赖：

- 阶段 4 的 admin API。
- 阶段 2 的 CLI source 规范化。

实施工作：

1. 在 `serviceImportCommand` 中新增 `--recursive` bool flag。
2. 调整 args 校验：
   - 非 recursive：保持 `SERVICE SOURCE`。
   - recursive：只接受 `SOURCE`。
   - recursive 与 `--name` 同用时报错。
3. recursive 模式请求体发送 `recursive:true`、`source`、`offline`、`reinstall`、`build`，不发送 `service_id`。
4. 更新 help 文案，保持 `--id` 不出现在 help 中。
5. 修改 `services/scripts/import-check-all.mjs`：
   - 使用 `octobus service import --recursive <root>` 一次导入。
   - 通过 `octobus service list` 校验每个 `discoverServices(root)` 结果都存在，并校验 `ID`、`ServiceRoot`、`NodeEntry`。
   - 保留参数校验测试。
6. 更新 `services/tests/validate-service-package.test.mjs` 中 import-check 对新命令的断言。

测试和验证：

- `go test ./internal/cli ./internal/admin ./internal/packageimport`。
- `node --test services/tests/validate-service-package.test.mjs`。
- 可选手动验证：构建 `bin/octobus` 后运行 `cd services && npm run import:check -- --octobus ../bin/octobus`。

验收标准：

- CLI 用户能执行 `octobus service import --recursive npm:@chaitin-ai/octobus-tentacles`。
- `--recursive --name`、缺 source、多 source 都给出明确错误。
- services 包 import check 不再使用旧的 `--id` 形态。

适用 harness 约束或命令：

- CLI 行为变化需要补充 CLI 单测。
- Node 脚本仍使用仓库已有 `node:test` 风格，不引入新测试框架。

## 阶段 6：补充端到端覆盖和文档

目标：证明 daemon + CLI + store + runtime artifact 路径整体可用，并更新用户可见文档。

依赖：

- 阶段 5 的 CLI 和 services 脚本。

实施工作：

1. 在 `internal/integration` 或 `tests/e2e` 增加 recursive import 场景：
   - 启动 daemon。
   - 导入本地 multi-service fixture 或 `services/` 子集 fixture。
   - 执行 `service list` 验证多个 service 出现。
   - 如 fixture 包含 long-running instance 更新路径，覆盖重启聚合响应。
2. 更新 `README.md`、`README.zh-CN.md` 和相关设计文档，加入 `service import --recursive SOURCE` 示例和 `source//some-dir` 作为 scan root 的说明。
3. 确认 `docs/spec/recursive-service-import-spec.md` 与实现文档没有冲突；必要时小幅同步文案，不扩大 scope。

测试和验证：

- `go test ./internal/cli ./internal/admin ./internal/packageimport ./internal/integration`。
- `go test ./tests/e2e -count=1`，因为本变更触达 CLI、package import 和 daemon 管理路径。
- `node --test services/tests/validate-service-package.test.mjs`。
- `task lint`。

验收标准：

- 用户文档和设计文档都描述 `--recursive`，不出现 `--all` alias。
- e2e 或 integration 能证明 `service list` 输出 recursive 导入的多个 service。
- 新增文档不承诺首版不做事项中的能力。

适用 harness 约束或命令：

- `AGENTS.md` 要求 package import、CLI、daemon 相关变更后运行 e2e。
- `task lint` 必须通过 gofmt 和 `go vet ./...`。

## 阶段 7：完整质量门禁和收尾

目标：完成仓库级验证，确保计划结果满足 harness 覆盖率、构建和发布前检查要求。

依赖：

- 阶段 1 到阶段 6 全部完成。

实施工作：

1. 检查 `git status --short`，确认没有误提交 `bin/`、coverage、node_modules、日志、data dir 或 service artifact。
2. 运行 focused tests，确认失败定位清晰：
   - `go test ./internal/cli ./internal/admin ./internal/packageimport`
   - `go test ./internal/integration`
   - `node --test services/tests/validate-service-package.test.mjs`
3. 运行 e2e：
   - `go test ./tests/e2e -count=1`
4. 运行完整 Task 门禁：
   - `task test`
   - `task lint`
   - `task build`
   - 最终可用 `task all` 等价串联 lint/test/build。

测试和验证：

- `task test` 会构建 SDK、安装示例依赖、运行 coverage 脚本、执行 minimum 示例。
- `task lint` 覆盖 gofmt 和 `go vet ./...`。
- `task build` 生成静态 `bin/octobus` 并检查动态链接。

验收标准：

- 所有 focused tests、e2e 和 Task 门禁通过。
- `task test` 输出仍满足 `AGENTS.md` 中 unit、integration、e2e 各 60% 和 overall 90% 覆盖要求。
- 最终 diff 只包含 recursive import 相关代码、测试、脚本和文档。

风险和停止条件：

- 如果 `task test` 失败但 focused tests 全部通过，必须保留失败命令和关键错误输出，先定位是否为 SDK/example/e2e 环境依赖问题。
- 如果覆盖率低于 harness 要求，不通过减少断言或跳过测试解决；应补充与 recursive import 行为直接相关的测试。

适用 harness 约束或命令：

- `AGENTS.md` 的 PR 要求：总结变更、列出 tests run；涉及 CLI 行为、service package format 或 runtime requirements 的变化必须在 PR 中说明。

## 首版不做的事项

- 不实现 `--all` alias。
- 不实现 include/exclude 过滤、service id 映射文件或手动重命名批量导入项。
- 不自动创建 instance、capset 或 method binding。
- 不把 `octobus-tentacles` dispatcher 注册为特殊 service。
- 不新增数据库 schema，不改变 `service.json` manifest schema。
- 不实现跨多个 service 的强事务回滚。
- 不扩展 SDK multi-service dispatcher 生成能力。

## 计划规则

- 每个阶段完成后必须保持项目可构建、相关 focused tests 可运行。
- 任何阶段发现 spec 与实现边界冲突，先更新 spec 或回到用户确认，不在代码中隐式改变公开语义。
- 不混入无关重构；共享 helper 只围绕 recursive import 和现有单 service import 去重。
- 所有错误信息应包含足够上下文，但不得泄露 credential、config 或 secret 内容。
