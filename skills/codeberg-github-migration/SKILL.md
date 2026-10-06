---
name: codeberg-github-migration
description: 将纯 Codeberg 仓库迁移为“GitHub 主仓库 + Codeberg 镜像”架构时使用。触发词包括：迁移到 GitHub、GitHub 主仓库、Codeberg 镜像、建 GitHub 仓库、GitHub Pages 部署、仓库双平台。涉及跨哈希（sha256 转 sha1）时先加载 git-hash-repo-conversion 技能。不负责 GitHub/GitLab 等其他 forge 之间的迁移，也不负责从 GitHub 迁回 Codeberg。
---

# Codeberg 迁移至 GitHub 主仓库 + Codeberg 镜像

## 适用判别

执行迁移前先回答三个判别问题，决定本 checklist 的子集：

1. **源仓库哈希格式**：`git rev-parse --show-object-format`。结果为 `sha256` 时，必须先完成跨哈希转换才能推 GitHub（GitHub 服务端只收 sha1），加载 `git-hash-repo-conversion` 技能；`sha1` 则跳过转换。
2. **是否有静态站点**：仓库有无 SSG 构建（SvelteKit/Astro/Hugo 等）与站点部署需求。有则执行 Pages 相关步骤；纯代码仓库跳过全部 Pages 章节，只需仓库与镜像。
3. **现有 CI 形态**：源仓库在 Codeberg 用 Forgejo Actions 时，workflow 文件（`.forgejo/workflows/`）需适配为 GitHub Actions（`.github/workflows/`）；迁移后 Forgejo workflow 保留与否取决于镜像站是否继续部署。

## 第 0 步：准备凭据

- **GitHub 侧**：确认 `gh` CLI 已登录且 token 有 `repo`、`workflow` scope（`gh auth status`）。
- **Codeberg 侧**：请用户在 Settings → Applications 生成一个 token，权限仅需 **repository 读写**。约定：token 仅在本次会话使用，迁移完成后提醒用户吊销。
- token 会出现在会话上下文中，不落盘、不写入任何文件。

## 安全红线

> 本技能里的 Codeberg API 命令多数已在 2026-09 的批量迁移（14 仓）中实测通过（见第 6、7 步的 [已实测] 标注）；仍未验证的只剩 workflow_dispatch 触发（当时项目均无部署目标）。
> 标注 **[待实测]** 的条目来自 Gitea/Forgejo 官方文档、未在 Codeberg 实测，遇及时先小步验证。
>
> 标注 **[时效性]** 的事实随平台演化，执行时先按参考资料链接复核：GitHub 对 sha256 的支持现状、GitHub 对 `gpgsig-sha256` 提交头的接受性。

## 主流程 checklist

按序执行，方括号条件不满足则跳过该步。

### 1. 前置核查

- [ ] 确认 GitHub 账号与用户名（`gh api user`），用户 Pages 仓库名必须是 `<用户名>.github.io`
- [ ] [源仓库为 sha256] 走跨哈希转换，产出 sha1 工作副本（辅技能）；转换后逐字节验证签名保留
- [ ] [有签名提交] 试推验证 GitHub 对 `gpgsig-sha256` 头的接受性——**先推一个小提交再推全量**，被拒则回来与用户重新盘问签名策略
- [ ] 确认签名钥匙与提交邮箱已注册到 GitHub 账号（Settings → SSH and GPG keys），否则 Verified 徽章不生效
- [ ] 无 LFS（`ls .gitattributes`；有 LFS 则本技能未覆盖，停下报告）

### 2. GitHub 建仓与推送

```bash
gh repo create <user>/<repo> --public --description "..."   # 用户 Pages 仓库必须名为 <user>.github.io
git remote add origin git@github.com:<user>/<repo>.git
git push -u origin main
```

- 硬编码了旧平台 URL 的构建脚本（prebuild/jq、站点内绝对 URL）先改再推，避免首次部署就产出错误内容。
- **功能性自引用**（纯文档仓与技能仓高发）同样先改再推：插件清单（`.claude-plugin/plugin.json` 的 homepage/repository）、
  发现索引（`.well-known/agent-skills/index.json` 的技能文件 URL）、README 里的安装命令都会固化旧域名，改名归档后即失效或绕远。
  码仓 raw 链接的形态换算：`https://<forge>/<owner>/<repo>/raw/branch/<branch>/<path>` 对应 `https://raw.githubusercontent.com/<owner>/<repo>/<branch>/<path>`；
  先替换 raw 形态再替换通用域名形态（顺序反了会把前者拆坏）。镜像 workflow 里的旧域名引用是有意为之，勿改。

### 3. GitHub Actions 恢复与适配

- **两个 workflow 目录都要翻**：`.github/workflows/` 与 `.forgejo/workflows/` 里都可能有内容，且未必匹配所在平台——实测见过 GitHub 风格的
  workflow 躺在 `.forgejo/`（在 GitHub 上不运行）与 Forgejo 风格（codeberg-medium runner、data.forgejo.org 的 cache、`forge.*` 上下文、
  `GIT_DEFAULT_HASH`）躺在 `.github/`。按目标平台重排目录，别按目录名假设平台。
- 从源仓库历史挖旧 workflow（`git log --all -- .github/workflows/`），逐个核对数据源与版本是否过时（数据源变更、pnpm/action 大版本），不能直接复用。
- **CI 壳子仓库可并入主仓**（只承载一条流水线的仓，如定时抓取+同步脚本仓）：迁入配置文件与 workflow 时注意三个坑——
  ① 触发条件加**路径过滤**（壳子仓内容只有配置，原样的 every-push 触发挪进内容仓后会随任意提交跑全量任务）；
  ② 流水线引用的制品 URL 若指向已迁移仓库的 release，需先把资产**转存**到新平台 release（迁移只带 tags、不带资产）；
  ③ **secrets 无法迁移**，需在新仓重填（凭据值只在原仓可写不可读）。
- 查各 action 当前主版本：`gh api repos/<owner>/<action>/releases/latest --jq .tag_name`，不凭记忆写版本号。
- 触发器对应迁移：Forgejo 的 `schedule`/`push` 语义与 GitHub 相同；部署用 `actions/upload-pages-artifact` + `actions/deploy-pages@v5`。

### 4. Pages 启用 [有静态站]

```bash
# GitHub 常在首次 push 后自动启用旧式分支部署（build_type=legacy），需切到 Actions 源：
gh api -X PUT repos/<user>/<repo>/pages -f build_type=workflow
gh api repos/<user>/<repo>/pages --jq '{build_type, source}'
```

部署 workflow 可手动触发验证：`gh workflow run <name> --ref main`。

### 5. 分支保护与 PR 流程

**优先用 rulesets（仓库规则集），不要用经典 branch protection**（`/branches/main/protection`）。创建规则集：

```bash
gh api -X PUT repos/<user>/<repo>/actions/permissions/workflow --input - <<'EOF'
{"default_workflow_permissions": "write", "can_approve_pull_request_reviews": true}
EOF
gh api -X POST repos/<user>/<repo>/rulesets --input - <<'EOF'
{
  "name": "main",
  "target": "branch",
  "enforcement": "active",
  "conditions": {"ref_name": {"include": ["~DEFAULT_BRANCH"], "exclude": []}},
  "bypass_actors": [],
  "rules": [
    {"type": "deletion"},
    {"type": "non_fast_forward"},
    {"type": "pull_request", "parameters": {
      "required_approving_review_count": 0,
      "dismiss_stale_reviews_on_push": false,
      "require_code_owner_review": false,
      "require_last_push_approval": false,
      "required_review_thread_resolution": false,
      "allowed_merge_methods": ["squash"]
    }}
  ]
}
EOF
gh api -X PATCH repos/<user>/<repo> -f allow_auto_merge=true
```

> **必知差异**：`required_approving_review_count: 0` 对单人仓库是防死锁关键——要求审批会因无法自批而永久卡住 PR。`bypass_actors: []` 意味着任何人（含管理员）都受规则约束，直推被拒、一切走 PR。是否合意由用户在盘问中决定，勿默认。
>
> **私有仓的现实约束**（免费账户实测）：私有仓**无法启用 ruleset 或经典分支保护**（API 返回 403 `Upgrade to GitHub Pro or make this repository public`）；
> 私有仓的 Actions 消耗计费分钟；私有 Codeberg 仓匿名 API 读不到内容，同步核验需带 token。迁移私有仓前把这三条告知用户。
>
> **纯文档仓的保护变体**：无 CI、无评审需求、且作者高频迭代的文档/技能仓，ruleset 用 `deletion` + `non_fast_forward` 两条即可——拿到防误删与禁强推的安全网，不引入单人协作无收益的 PR 门槛。是否加 `pull_request` 规则取决于用户，别默认全套。
>
> **迁移期红线**：ruleset 会拦住首次/批量推送（报错 `push declined due to repository rule violations`，**`git push --dry-run` 不报此错，必须以真实推送为准**）。迁移推送的合法路径有三，按优选顺序：① 单次推送走 PR + 规则允许的合并方式（若仅允许 squash 而需保留多提交历史，
> 此路不通）；② 临时把规则集 `enforcement` 置为 `disabled`（先 GET 备份全文，推完立即 PUT 恢复 `active`，并用一次试推验证拒绝恢复）；③ 加临时 bypass_actor（不推荐，勿留后患）。
>
> **dependabot 运行的三重陷阱**（恢复 CI 时逐条核对，否则 auto-merge 会永远卡死）：
> ① clippy/测试在新工具链下的新 lint 会让 check 变红；② dependabot 触发的运行**读不到 Actions secrets**，
> 依赖 token 的步骤（codecov 上传等）必失败——token 需另行存入 **Dependabot secrets** 存储；
> ③ 第三方审查 App（如 AI reviewer）会发结论为 `neutral` 的 check，而 `poseidon/wait-for-status-checks`
> 只接受 `success`/`skipped`，门禁因此恒败——用 `ignore_pattern` 把审查类 check 从等待列表剔除。
> 另注：`pull_request` 事件的 workflow 取自 PR head 提交，修复要等 dependabot rebase/recreate 分支后才生效。
>
> **GITHUB_TOKEN 事件抑制**：用 `secrets.GITHUB_TOKEN` 完成的推送/合并**不会触发后续 workflow**（典型场景：dependabot 的 auto-merge workflow 用 GITHUB_TOKEN 合并 PR，其 push 不会触发 mirror/CI）。影响：镜像同步只对“人推动的提交”即时生效，
> bot 合并需靠 cron 兜底。两条对策：镜像 cron 加密（日频，绑定漂移≤ 天）或把 auto-merge 的凭据换成 PAT（即时，但多一份凭据轮换）。
>
> **已有经典分支保护的仓库：转换而非叠加**。仓库若已在用经典保护（`/branches/main/protection` 可读到），不要叠加新 ruleset，而是等价转换后删除经典配置。
> 映射关系：`enforce_admins: true` → ruleset 的 `bypass_actors: []`（空即无人豁免）；`required_status_checks.contexts` → `required_status_checks` 规则（`strict` → `strict_required_status_checks_policy`）；
> `allow_force_pushes` / `allow_deletions` 的 false → `non_fast_forward` / `deletion` 规则；`required_approving_review_count` 等 → `pull_request` 规则参数，
> `allowed_merge_methods` 需与仓库设置的合并方式一致（`allow_squash_merge` 等，别写出仓库不允许的合并方式）。转换顺序：先建 ruleset 并读回核对，再
> `gh api -X DELETE repos/{owner}/{repo}/branches/main/protection`，最后用一次试推验证拒绝仍然生效。
> 若仓库已有他人建的 ruleset（例如用户自己迁移过），先 GET 读全量规则核对是否已覆盖，避免重复建规则互相打架。
>
### 6. Codeberg 镜像侧重构

```bash
# Codeberg API 前缀：https://codeberg.org/api/v1
curl -s -X PATCH 'https://codeberg.org/api/v1/repos/<user>/<旧仓>' \
  -H "Authorization: token $CODEBERG_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"name": "<repo>-archive"}'                                    # [已实测] rename，repository scope 即可
curl -s -X POST 'https://codeberg.org/api/v1/user/repos' \
  -H "Authorization: token $CODEBERG_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"name": "<repo>"}'                                            # [已实测] 需 token 含 user 读写 scope；新建默认 sha1 且 has_actions=false
# 私有仓在 body 里加 "private": true（归档仓与镜像仓同可见性）
```

> > **归档仓的生命周期**：`<repo>-archive` 保留原始哈希历史（原始 OID 与平台侧 issue/PR/release 记录），是迁移期的安全网。长期保留或稳定后删除由用户定：
> 删除前确认三件事：①镜像已多日同步正常、无仍在使用原始 OID 的引用；②归档仓里**独有的 release 资产**已转存到新平台
> （迁移只带 tags 不带资产，若目标平台缺资产则从旧平台下载后用 `gh release create --verify-tag` 重建并核对 sha256）；③清楚删除后旧仓库名通常已被新镜像仓占据，历史链接会指向镜像（内容一致、OID 不同）。
>
> **必知差异**：Codeberg 现役 Pages 是 git-pages（旧 pages-server 维护模式）。部署域名与仓库名绑定：根域名 `<user>.codeberg.page` 要求发起部署的仓库命名为 `pages`；子路径站点要求仓库名匹配 `{user}.codeberg.page/{repo}`；
> 名字不匹配需 PAT。改名到新建 `pages` 之间，镜像站内容短暂空窗（旧静态部署仍在服务，内容为旧构建）。

### 7. 镜像同步

deploy key 路线（推荐，见“批量使用”节）：

```bash
ssh-keygen -t ed25519 -f /tmp/cb-deploy -N '' -C 'github-actions-mirror'
# 公钥经 API 加为 deploy key（写权限）：
curl -s -X POST "https://codeberg.org/api/v1/repos/<user>/<repo>/keys" \
  -H "Authorization: token $CODEBERG_TOKEN" \
  -H 'Content-Type: application/json' \
  -d "{\"key\": \"$(cat /tmp/cb-deploy.pub)\", \"title\": \"github-actions-mirror\", \"read_only\": false}"
# 私钥入 GitHub Secrets，随后删除本地私钥文件：
gh secret set CODEBERG_DEPLOY_KEY -R <user>/<repo>.github.io < /tmp/cb-deploy && rm /tmp/cb-deploy
```

同步 workflow（push main 即同步 + 每日 cron 兜底；日频而非周频的原因见第 5 步的 GITHUB_TOKEN 事件抑制说明——bot 合并只能靠 cron 收敛）：

```yaml
name: Mirror to Codeberg
on:
  workflow_dispatch:
  push:
    branches: "main"
  schedule:
    # 每日兜底：bot（GITHUB_TOKEN）合并不触发本 workflow，漂移由 cron 收敛
    - cron: "30 4 * * *"
jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }
      - name: Set up deploy key
        env: { DEPLOY_KEY: "${{ secrets.CODEBERG_DEPLOY_KEY }}" }
        run: |
          mkdir -p ~/.ssh
          echo "$DEPLOY_KEY" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan codeberg.org >> ~/.ssh/known_hosts
      - run: |
          git remote add codeberg git@codeberg.org:<user>/<repo>.git
          git push codeberg main
```

- Forgejo 侧镜像仓库的 workflow 适配：`.forgejo/workflows/` 里 `GIT_DEFAULT_HASH` 环境变量与实际格式不符时必须移除（checkout 会报 mismatched algorithms）。
- **[已实测] Codeberg 仓库级 Actions 开关**：`PATCH /repos/{owner}/{repo}` 带 `has_actions` 可用（2026-09-14）；注意 **API 新建仓库默认 `has_actions: false`**——需要部署的站点项目必须显式开启，
纯代码镜像仓则保持关闭以防 workflow 误触发。push 事件早于启用会被错过且不补跑，需手动触发。
- **[待实测] workflow_dispatch 触发**：`POST /repos/{owner}/{repo}/actions/workflows/<workflow_id>/dispatches`（bmsrs 无站点无部署目标未涉及；首个带 Pages 的镜像项目继续校准）。
- 首次部署验证：镜像站内容应以构建产物的新 URL 判定新鲜度，HTTP 200 可能只是旧部署残留。

### 7.5 遗留 PR 的处置（归档仓库复活后常见）

归档前留下的开放 PR（release-plz 发布 PR、dependabot 依赖 PR）在取消归档后**会被各自的 bot 重新接管并刷新**——判断是否“陈旧”必须看最后提交时间，而不是最初创建时间：
`gh pr view <n> --json updatedAt,commits --jq '...'`，再用 `gh api repos/{o}/{r}/compare/main...<head sha>` 看 ahead/behind。
补充一类批量清理：**休眠仓**（数月无提交）里 renovate/dependabot 的陈旧 bump PR——仓库不活跃时 bot 不再刷新它们，若依赖写法是 semver 范围（只需更新锁文件）则可整批关闭，仓库恢复活动后 bot 会重建。
两条实测教训：① release-plz 的 release PR 是“待你合并的发布”而非遗留物，关掉它是错的（它每天被刷新）；② dependabot 的依赖 PR 关闭前先确认该依赖
是否仍在 `Cargo.toml`/lockfile 里——项目删掉的依赖，其 bump PR 才是真陈旧。用户本人的历史 PR（author 不是 bot）关闭前更要先读 diff，里面可能有未合入的真实工作。

### 8. 收尾验证清单

- GitHub：远端 HEAD 与本地一致、提交 Verified（新提交）、Pages 200、保护规则生效（试推被拒）
- Codeberg：镜像 main HEAD 与 GitHub 一致、镜像站 200 且内容新鲜、旧归档仓库原史完整
- 双侧 tables/构建产物 URL 指向约定域名（主站域名）
- 发布资产：目标平台的 release 资产与源平台一致（缺失则按第 6 步的转存路径补齐）
- 本地工作副本：用户若有本地克隆，对象格式与 origin 需重接——sha256 克隆推不了 sha1 主仓，就地替换为 GitHub 克隆（旧克隆改名保留或删除）；
  chezmoi 之类按路径解析的工具不受替换影响，但 `.github/` 等点开头目录在其源状态里属元数据、不会被部署
- 提醒用户吊销第 0 步的 Codeberg token

## 批量使用

- 首个项目选**最简单**的（无签名、无特殊数据管线），作为 API 命令与流程的校准载体；[待实测] 项逐条验证并回填本技能。
- 每项目固定验证：第 8 步收尾清单，一项不落。
- 失败项目排队记录原因，不阻塞后续项目；同因失败不重试，先诊断。
- token 按项目单独生成或一token多用由用户定，默认建议用完即吊销。**scope 提示：纯 repository scope 建不了仓**——`POST /user/repos` 需 token 含 user 读写（2026-09-14 实测），批量场景建议 token 一次带 user + repository 双 scope，避免二次索要。

## 批量迁移实况（2026-09）

两批共 14 仓完成迁移，覆盖五种形态（静态站、纯代码库、纯文档/技能仓、私有仓、CI 壳子仓）；另有 6 仓原地归档、12 个归档仓按期清理。
共性数据：单仓 41–767 提交，签名逐字节验证全过；镜像 deploy key 一仓一钥、cron 错峰。最大的时间成本不在转换本身，而在依赖自动化（dependabot 门禁的三重陷阱）
与本地克隆的重接；错误率最高的两处是“按目录名假设平台”（.forgejo/.github 用反）与“迁移只带 tags 不带 release 资产”，已分别固化为第 3、6 步的红线。

## 案例附录：bmsrs 项目迁移快照（2026-09-14，纯代码库形态）

- 源仓库 sha256、322 提交、284 个 `gpgsig-sha256` 签名；转换后签名字节逐字节保留，GitHub 全数接受。
- 与 pages 的差异点（纯代码仓库形态）：
  - 无站点：跳过全部 Pages 章节；**Codeberg 镜像仓 Actions 保持 false**（API 新建默认即 false），`.forgejo/workflows` 直接从主分支删除，防止 release-plz 在镜像侧误触发。
  - **GitHub 新建仓库的 GITHUB_TOKEN 默认只读**（workflow 内 `permissions:` 只能降不能升）：需要 Actions 写 API 的自动化必须改仓库设置，见下条命令。
  - 设置命令：`PUT /repos/{owner}/{repo}/actions/permissions/workflow`，body 设 `default_workflow_permissions: write`；
需要 Actions **创建 PR**（如 release-plz）还须 `can_approve_pull_request_reviews: true`（设置名虽叫 approve，实际控制创建与批准两件事）。
  - workflow YAML 同一 step 出现两个 `env` 键会被 GitHub 解析即败（0 秒红，报“workflow file issue”而非运行时错误）——移植 workflow 时注意平台 YAML 解析器严格性差异。
  - release-plz 适配要点：`--forge gitea` 改 `--forge github`；GitHub runner 无 Rust 工具链与 cargo-binstall，需补 `dtolnay/rust-toolchain@stable` 与 binstall 安装脚本步。

## 案例附录：pages 项目迁移快照（2026-09-14，静态站形态）

- 源仓库 sha256 + 114 个 `gpgsig-sha256` 签名提交；fast-export/import 转 sha1 后签名字节逐字节保留，GitHub 全数接受。
- 用户决策：签名原样保留（不重签）；分支保护 enforce_admins 不豁免 + 0 必需批准；ci 恢复旧三 job 结构；dependabot 周频。
- 坑位实录：
  - GitHub 首推后自动启用旧式 Pages（build_type=legacy），需显式 PUT 切 workflow 模式，且 `POST .../pages` 已存在时报 409。
  - 镜像站 HTTP 200 不等于部署成功——旧静态内容会继续服务，以产物内 URL 判新鲜度。
  - Codeberg 与 GitHub 的 runner jq 版本对 `@uri` 括号编码不同，产物 URL 等价可用（SvelteKit 路由解码后等价），非缺陷。
  - `pages` 与 `pages-archive` 的 Forgejo workflow 需要分别适配：新仓移除 sha256 env，归档仓的 cron 部署因域名不匹配会失败报红（可禁用其 Actions）。

## 参考资料

官方文档：

- Forgejo repo mirror：<https://forgejo.org/docs/v15.0/user/repo-mirror/>
- Forgejo Actions reference（schedule/dispatch 语义）：<https://forgejo.org/docs/v15.0/user/actions/reference/>
- Gitea API repo-edit（rename、`has_actions`、`object_format_name`）：<https://docs.gitea.com/api/next/operations/repo-edit>
- Gitea API deploy key（repoCreateKey）：<https://docs.gitea.com/api/next/operations/repo-create-key>
- GitHub Pages：<https://docs.github.com/pages>
- GitHub branch protection API：<https://docs.github.com/rest/branches/branch-protection>
- GitHub workflow dispatch API：<https://docs.github.com/rest/actions/workflows>
- Codeberg Actions 限额与 fair-use：<https://codeberg.org/actions/meta>
- Codeberg API swagger（实测端点与 scope 报错以此为据）：<https://codeberg.org/api/swagger>

关键社区实证：

- GitHub 拒收 sha256 push 与 compatObjectFormat 实测：<https://github.com/orgs/community/discussions/154056>
- git-pages 部署机制与域名绑定规则实测：<https://andre601.ch/blog/2026/04-24-using-git-pages-on-codeberg/>
- Git hash 过渡设计（`gpgsig-sha256` 头语义）：<https://git-scm.com/docs/hash-function-transition>
