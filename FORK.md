# Fork 维护说明

本仓库是 [herdrdev/herdr](https://github.com/herdrdev/herdr) 的个人 fork，用于长期
维护上游暂不接受的功能。上游贡献流程见 `CONTRIBUTING.md`：未经
`.github/APPROVED_CONTRIBUTORS` 授权的实现类 PR 会被自动关闭，因此自定义改动在
本 fork 内自行集成，不再以 PR 形式回流上游。

**在改动本仓库前先读这份文件。** 上游的 `AGENTS.md` 依然适用于代码风格和项目约定；
这份文件只描述 fork 特有的分支与同步规则。

## Remote 约定

| remote   | 地址                              | 用途                     |
| -------- | --------------------------------- | ------------------------ |
| `origin` | `git@github.com:herdrdev/herdr`   | 上游，只读，永不要推送   |
| `fork`   | `git@github.com:hxy91819/herdr`   | 自己的 fork，推送目标    |

## 分支模型

```
origin/master（上游）
      │  fast-forward only
      ▼
  upstream-sync         上游镜像，绝不在其上提交
      │
      ├─► <feature>     每个特性一条分支，rebase 跟进上游
      │
      └─► dist/main     集成分支 = upstream-sync + 所有特性分支
```

- **`upstream-sync`** — 上游 `master` 的镜像，只允许 fast-forward。它保证任意时刻都
  能用 `git log upstream-sync..dist/main` 看清我们自己加了什么。
- **`<feature>`** — 每个特性一条独立分支，从 `upstream-sync` 切出，上游更新后用
  rebase 跟进。特性之间互不依赖，上游一旦接受某个改动，直接删掉对应分支即可，
  其余分支不受影响。当前分支：
  - `codebuddy-detection` — CodeBuddy Code 的 agent 检测支持
- **`dist/main`** — 集成分支，fork 的默认分支，也是实际构建和使用的版本。它用
  merge（不是 rebase）合入上游和特性分支，历史稳定、不重写，多台机器 pull 不会分叉。
  冲突只在 `dist/main` 上解决一次，解法由 git rerere 记录后自动复用。

### 没有本地 `master`

本地 `master` 已删除。原因是上游 `ci.yml`、`website.yml`、`label-next-release-issues.yml`、
`nix.yml`、`windows-arm64.yml` 全部以 `push: branches: [master]`（或 `windows`）触发。
只要往 fork 推名为 `master` 的分支，每次同步都会跑一遍完整 CI 并尝试部署网站。

> **绝对不要向 fork 推送名为 `master` 或 `windows` 的分支，也不要同步上游的 tag**
> （`release.yml` 由 `v*` tag 触发，会在 fork 上尝试发版）。

## 日常同步

```bash
# 1. 更新上游镜像（只允许 ff，有分叉说明镜像被污染，需排查）
git fetch origin
git checkout upstream-sync && git merge --ff-only origin/master

# 2. 特性分支跟进上游，在此解决冲突
git checkout <feature> && git rebase upstream-sync
git push --force-with-lease fork <feature>

# 3. 合入集成分支
git checkout dist/main && git merge upstream-sync && git merge <feature>
git push fork dist/main
```

`upstream-sync`、`dist/main` 的推送已由 `.github/workflows/upstream-sync.yml` 每日自动
完成，上述命令主要在需要人工解冲突或本地开发时使用。

## 新增功能：先选层，再动手

Herdr 有三层扩展手段。**改源码是最后一层**——它是唯一需要跟着上游永久维护的选择。

### 第一层：本地 manifest override（零改动）

调整**已内置** agent 的检测规则，把清单放到平台配置目录即可：

```text
~/.config/herdr/agent-detection/<agent>.toml
```

本地覆盖优先级最高，支持热加载（`herdr server reload-agent-manifests`），不碰源码、
也不碰本 fork。规则调参、临时绕过误判，一律先走这一层。

> **边界**：覆盖是按 `Agent` 枚举查找的（`src/detect/manifest.rs` 的 `override_path`，
> 文件名取 `agent_label(agent)`），所以**只能覆盖已经存在的 agent，不能凭空新增一种**。

### 第二层：插件（零改动，可分享）

插件是一个带 `herdr-plugin.toml` 的目录，命令可以是 Bash、Node、Python、PowerShell、
Rust 二进制等任意 argv 程序。整个 Herdr CLI 就是插件 API，插件内用 `HERDR_BIN_PATH`
回调 Herdr，避免自己处理 Unix socket / Windows named pipe 的差异。

```bash
herdr plugin link /path/to/plugin        # 本地开发用 link
herdr plugin install owner/repo/subdir   # 安装他人发布的
```

插件 v1 能做的声明式能力：`actions`、`events` 钩子、`startup` 钩子、`panes`、
`link_handlers`，以及把按键绑到 action 上。工作流、自动化、外部集成、自定义面板、
Git/工单系统联动，都**先考虑做成插件**。

> **边界**：插件 v1 **不能**在运行时注册 action、**不能**注册新的 agent 检测清单、
> **不能**提供原生非终端 UI。需要这些能力才到第三层。

参考实现：`ogulcancelik/herdr-plugin-examples`（官方不维护，当作抄的范例）。

### 第三层：改源码（本 fork 的活）

只有前两层都做不到时才改源码。典型例子：**新增一种 agent**——必须动 `Agent` 枚举、
`agent_label`、两处清单、三语文档，插件和 override 都做不到。走下节的完整流程。

## 改源码的完整流程

1. **切分支**：从 `upstream-sync` 切出，一个 feature 一条分支，名字用 kebab-case。

   ```bash
   git fetch origin
   git checkout upstream-sync && git checkout -b my-feature
   ```

2. **改代码**：遵守上游 `AGENTS.md` 的通用约定（Rust 生产代码不用 `unwrap()`、平台相关
   代码放 `src/platform/<os>.rs`、渲染保持纯函数、不引入没必要的依赖）。提交用小写
   conventional commit。

3. **验证**：见"验证"一节。

4. **跟进上游**：`git rebase upstream-sync`，冲突按提交逐个解决，然后
   `git push --force-with-lease fork my-feature`。

5. **合入集成分支**：`git checkout dist/main && git merge my-feature && git push fork dist/main`。
   这一步**保持手动**，不自动化——把什么合进可用构建应该是个主动决定。

## 冲突处理

- 已开启 `git config rerere.enabled true`，同一处冲突的解决方式会被记录并自动复用。
- **特性分支用 rebase 跟进上游，冲突在 rebase 过程中按提交逐个解决**，保持每个提交
  独立可审。
- **`dist/main` 只做 merge，永不 rebase**。它的历史一旦推送就不再改写。
- 高频冲突点是 `docs/next/CHANGELOG.md` 和 `docs/next/website/**` 的 agent kind 列表
  ——上游每加一个新 agent 就会改一遍。解决方式是取上游版本，再插入自己的条目，
  保持顺序与 `Agent::ALL` 一致。
- 上游把 agent 清单拆成了两处，需要**同时**更新：
  - `src/detect/manifests/` — 通过 `include_str!` 编译进二进制
  - `distribution/agent-detection/` — 运行时热更新用，并在 `index.toml` 登记

## 验证

上游要求提交前跑 `just check`（格式检查 + `cargo nextest` + 维护脚本测试）。只改了清单
和文档、没动 Rust 时，上游自带的校验脚本更快也够用。

```bash
# 完整验证（需要 zig / just / cargo-nextest）
just check

# 只做类型检查和测试
cargo check --all-targets
cargo nextest run

# 只改了 agent 清单和文档时的快速校验
python3 scripts/agent_detection_manifest_check.py   # 两处清单与 index 的一致性
python3 scripts/config_reference_check.py           # 配置参考文档同步
python3 scripts/docs_translation_parity.py          # en / ja / zh-cn 文档对齐
PYTHONPATH=. python3 -m scripts.test_changelog       # changelog 格式
```

`docs_translation_parity.py` 特别值得跑：本 fork 的补丁同时改了英文、日文、简体中文
三份文档，漏改一种语言就会在这里暴露。

### 环境准备

`build.rs` 调用 Zig 编译 vendored `libghostty-vt`，当前要求 **Zig 0.15.2**。缺少时
`cargo check` 会以 `zig executable not found` 失败，可用 `ZIG` 环境变量指向二进制。

```bash
# Linux x86_64
curl -fsSL https://ziglang.org/download/0.15.2/zig-x86_64-linux-0.15.2.tar.xz \
  | tar -xJ -C /opt
export PATH="/opt/zig-x86_64-linux-0.15.2:$PATH"
zig version
```

本机（当前开发环境）Zig 0.15.2 已装在 `/opt/zig/zig-x86_64-linux-0.15.2/zig`，并已
软链到 `/usr/local/bin/zig`，无需再装。

**已知坑**：如果 zig 构建报
`failed to spawn and capture stdio from .../uucode_build_tables: FileNotFound`，
且目标文件实际存在，原因是本机 `/root/.cache -> /data/cache-migrated/.cache` 是
符号链接：zig 按**逻辑路径**（5 层）计算子进程可执行文件的相对路径，内核却按**物理
路径**（6 层）解析 `..`，差一层导致 ENOENT。修法是把 zig 全局缓存指到无符号链接的
物理路径：

```bash
export ZIG_GLOBAL_CACHE_DIR=/data/zig-global-cache   # 物理路径，勿放 ~/.cache 下
```

`cargo check` 会继承该环境变量传给 `zig build`，直接带上它运行即可。本机已写入
`~/.cargo/config.toml` 的 `[env]`，所以直接跑 `cargo check` 也会带上该变量。

`just` 和 `cargo-nextest` 不在默认工具链里，按需安装。

## 自动化

`.github/workflows/upstream-sync.yml` 每日 UTC 16:00（北京时间 0:00）运行，也可手动
触发。它 fast-forward `upstream-sync` 到 `origin/master`，然后 merge 进 `dist/main`；
出现冲突时中止并自动开 issue，不会写坏分支。

## 备份

`backup/pre-dist-5365aaf7` 保留了建立本结构之前的原始提交（`5365aaf7`，基于上游
`6e8b138d`），确认新结构稳定后可删除：

```bash
git branch -D backup/pre-dist-5365aaf7
```

查看当前 fork 相对上游携带的全部改动：

```bash
git log --oneline upstream-sync..dist/main
```
