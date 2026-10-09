# linxira-hwd-detector · Agent 开发规范

> **档位**:A · 系统源仓
> **本仓职责**:硬件探测与驱动配置工具(CHWD 1.23.0 全量移植),提供完整 `lhwd` CLI 与供
> `linxira-hardware-driver-manager` 消费的无参只读 JSON 报告。
> 通用约束见工作区总纲 `f:\Linxira-OS\AGENTS.md` 与发布规范 `linxira-os/docs/RELEASE_STANDARD.md`;
> 本文只写本仓特有内容。

## 职责与边界

- 上游为 CachyOS CHWD 1.23.0(fork 点与出处见 `UPSTREAM.md`),授权 `GPL-3.0-only`,Git 历史保留。
- **双二进制,同一份 `src/main.rs`**:
  - `lhwd` —— 完整 CHWD CLI(`--list` / `--autoconfigure` / `--install <profile>` / `--remove <profile>`)。
  - `linxira-hwd-detector` —— 无参模式,输出只读 JSON 硬件报告;任何参数都会路由到完整 CLI。
- 无参模式的**固定契约**:只读 `/sys/bus/pci/devices/*/{class,vendor,device}`、
  `/sys/devices/virtual/dmi/id/{sys_vendor,product_name,chassis_type}`、`/proc/cpuinfo`;
  输出单个 JSON 文档,`schema_version` 版本化契约、`detector.version` 版本化实现,数组排序保证确定性,
  缺失证据用 `null`/空数组 + 结构化 warning。零副作用。
- 输出的稳定 profile ID 仅描述硬件类别(`cpu.*` / `graphics.*` / `vm.*`),**故意不含包或变更策略**;
  ID→动作的映射由 `linxira-hardware-driver-manager` 拥有。
- **不做内核构建**:上游 CachyOS 会为新硬件构建定制内核,本仓只随 `linux` / `linux-lts` 通用内核,
  因此**不含**内核构建 worktree。
- 驱动档案数据部署于 `/var/lib/chwd/db/<subdir>/profiles.toml`(上游布局,由包安装)。

## 目录布局

- `src/main.rs`、`src/lib.rs`(`[lib] name = "lhwd"`)
- `libpci/`、`libusb/`(path 依赖,本地 crate)
- `tests/detection.rs`、`tests/fixtures/*.json`、`tests/profiles/*.toml`
- `build.rs`、`i18n.toml`、`rustfmt.toml`、`Cargo.toml`、`UPSTREAM.md`、`VERSION`

## 本地校验

需 Linux 头文件 `libpci-dev`、`libusb-1.0-0-dev`(CI 用 `apt-get install -y libpci-dev libusb-1.0-0-dev`):

```sh
cargo fmt --all -- --check
cargo test --all-targets --locked
cargo clippy --all-targets --locked -- -D warnings
cargo build --release --locked
./target/release/linxira-hwd-detector   # 应输出 schema_version == 1 的 JSON
test -x ./target/release/lhwd && ./target/release/lhwd --help | grep -q autoconfigure
```

## 版本与发布

- 版本唯一来源是本仓根 `VERSION`(当前 `1.23.0`);`Cargo.toml` 的 `version` 必须与其同步。
- 改 `VERSION` 即触发 `.github/workflows/release.yml`(分支 `main`,路径 `VERSION`)自动建 `v<ver>` Release,
  随后进入 `[linxira]` 包仓库全自动 auto-bump 链。**禁止手工 bump**。

## 禁区

- 无参二进制必须保持零副作用与 JSON 契约(`schema_version`)稳定;不得新增参数分支破坏"无参=只读报告"。
- 不在本仓引入内核构建流程;不把硬件类别 ID 与包/变更策略耦合进探测层。
- 不引入 `cachyos` / `aur` / `seafoam` 仓库引用;不直接 `sudo`/`su`/`doas`/`run0`(提权只走 `linxira-components` + polkit)。
- 不抹除上游归属与 `GPL-3.0-only` 许可;不 `git add -A`(勿带入 `target/` 产物)。