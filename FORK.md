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

## 新增特性

```bash
git checkout upstream-sync
git checkout -b my-feature
# ... 提交 ...
git push -u fork my-feature
git checkout dist/main && git merge my-feature
```

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
