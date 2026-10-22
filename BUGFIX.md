# BUGFIX

## 标题

`ops_bash` 的 `script_path` 传 `.ps1` 会被当作 bash 执行（上游 issue #364 之一）：改为在参数校验阶段明确拒绝 Windows 脚本，不再以 exit 127 误报脚本出错。

版本：上游 0.4.1 → 本版 0.4.2（patch +1）。

## 证据链

1. **上游 issue**：t-auto/dsh-ops#364（open）报告：内容只有 `Write-Output "hello"` 的 `.ps1` 经 `script_path` 传入，结果 `Write-Output: command not found`、exit 127，因为文件实际被 bash 执行（报告指向当时的 `lib/execution.js:70`，现 `src/shells/execution.ts:72` 的 `argv = [file, spec.scriptPath]` 原样拼接）；而工具描述又明确建议“引号、反斜杠、非 ASCII 就走 script_path”（现 `src/shells/tool.ts` 的 `bashToolDescription()`），照建议写 PowerShell 正好撞上。
2. **修复前复现**（Linux 沙箱，Node v24.20.0，用构建后的 `lib/` 与真实 `/bin/bash`）：
   - 地面真值：`bash hello.ps1` → exit 127，第一行 stderr 为 `hello.ps1: line 1: Write-Output: command not found`，与 issue 现象一致。
   - `bashToolDefinition(...).resolve({ script_path: '<tmp>/hello.ps1' })` **不报错**，返回可执行 spec —— 即插件层对此毫无拦截，误分发成立。复现脚本退出码 1 并打印 `BUG REPRODUCED`。

## 根因

`ops_bash` 只有一个 bash 执行器，`script_path` 在参数校验（`src/shells/tool.ts` 的 `resolve()`）中只校验“非空字符串、与 `command` 二选一”，从不校验脚本类型；`bashToolDescription()` 又把所有长脚本/含引号脚本统一引向 `script_path` 且不提 shell。于是确定无法被 bash 执行的 Windows 脚本（`.ps1`、`.bat` 等）被原样交给 bash，其语法错误以 exit 127 的形式伪装成“脚本本身写错了”，误导排障方向。

## 改动文件

- `src/shells/tool.ts`（唯一实现改动）：新增 `WINDOWS_SCRIPT_EXTENSIONS`（`.ps1`、`.psm1`、`.psd1`、`.bat`、`.cmd`、`.wsf`、`.wsh`、`.vbs`、`.jse`）与 `assertBashScriptPath()`；`resolve()` 在二选一校验后调用之。命中时抛带类型的 `BashArgumentError`（字段 `script_path`），文案点明扩展名与出路：“ops_bash runs bash scripts only — run it with the host PowerShell tool, or convert it to a bash script”。无扩展名与 `.sh` 路径行为不变；扩展名提取只看最后一个路径段，避免目录名含点误判。
- `test/shells-guards.test.mjs`：在既有“带类型拒绝”用例表中新增 2 行（`deploy.ps1`、`SETUP.CMD`），与同表用例共同断言拒绝发生在生成进程之前（spawn 计数为 0）。
- `docs/CHANGELOG.md`：`[Unreleased]` 顶部新增中英双语条目（上游惯例）。
- `package.json`：version `0.4.1` → `0.4.2`，repository.url 指向本组织仓库；其余字段未动。

注：`lib/` 为 `src/` 的构建产物（上游 git 不追踪），未手改；`runtime/`、`native/target/` 同为构建产物，不在交付目录内。

## 复验结果

- 复现脚本：修复后打印 `FIXED: resolve() refused the .ps1 script_path: invalid script_path: ... is a Windows script (.ps1); ops_bash runs bash scripts only — ...`，退出码 0（修复前为 1）。
- `npm run typecheck`：通过（exit 0）。
- 定向套件：`shells-guards` 7/7、`shells-bash-run` 16/16、`shells-execution` 5/5、`policy` 13/13 全过。
- 全量套件（`node test/run.mjs`，127 个）：修复后 1 failed / 2 skipped；唯一失败为 `linux-sandbox.test.mjs` 的“只读目录写入应报 ACCESS_DENIED”一条——沙箱以 root（uid 0）运行，权限位不生效，写入成功。已用 `git stash` 在未改动的基线树上复跑该单套件，结果同样失败，确认与本修复无关（环境差异，非回归）。作为对照，无原生 worker 时基线为 4 failed / 48 skipped；本轮先用 rustup 装好工具链并 `npm run build:ptc` 构建出 `native/target/release/dsh-ops-worker`，其余先前失败的套件（discovery、presentation-readiness、resident-tools、bundle-startup）随之全部转绿。
- 门禁：`npm run check:size` 通过（385 个文件均 ≤1000 行）；`npm run validate:manifest` 73 项全过。

## 验证边界

- 本轮只修 #364 的**第一部分**（`.ps1` 误走 bash）。同 issue 的第二部分（常驻 MSYS 运行时 PATH 中 `ucrt64/bin` 排在用户 PATH 之前顶掉用户工具，`src/bash-ipc.ts` 的 PATH 拼接）**未改动**：其拼接发生在 win32 专用运行时制备路径，现有测试在非 win32 上全部跳过，Linux 沙箱内无法构造可复现的前后对照，宁可如实留空也不做无验证的改动。
- `.ps1` 的真实 PowerShell 执行不在本插件职责内（插件只发布 bash 梯级，PowerShell 由宿主 pwsh 工具承担），故修复目标是“明确拒绝并指路”，不是代为分发到 PowerShell。
- Windows 真机未验证：沙箱为 Linux x64；本修复是纯参数校验逻辑（字符串扩展名判定），不依赖平台 API，定向套件与全量套件均在 Linux 上真实运行。Windows 上 `ops_bash` 的端到端行为（宿主把 `BashArgumentError` 投影为参数拒绝）未实测。
