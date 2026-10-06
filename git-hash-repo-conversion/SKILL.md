---
name: git-hash-repo-conversion
description: 将 Git 仓库在 sha256 与 sha1 对象格式之间转换时使用。触发词包括：对象格式转换、sha256 转 sha1、sha1 转 sha256、跨哈希迁移、签名保留、重写历史推 GitHub、object-format。纯文档/普通仓库转换不需要签名字节级保留时不加载。不负责常规 rebase/filter-repo 式历史清理。
---

# Git 跨哈希仓库转换（sha256 ↔ sha1）

## 核心机制

仓库对象格式由 `extensions.objectFormat` 固定，`git clone` 无法指定目标格式，跨格式 fetch 被拒（`mismatched algorithms: client sha1; server sha256`）。唯一通用路径是 **fast-export / fast-import 管道**：导出流是哈希无关的中间格式，import 时按目标仓库格式重算全部 OID。

## 转换管道

```bash
git -C <源仓库> fast-export --all --signed-commits=verbatim --reencode=no > /tmp/agent-cache/<名>.fi
git init --object-format=sha1 -b main <目标目录>          # 或 sha256
git -C <目标目录> fast-import < /tmp/agent-cache/<名>.fi
git -C <目标目录> checkout main
```

- `--signed-commits=verbatim` 保留签名字节；默认行为是剥离（strip），会静默丢签名。
- 转换后所有对象 OID 全部改变（提交、树、blob），旧 OID 的外部引用失效；ref 名单与提交数应与源仓库一致，`fsck` 应干净。

## 签名的机制与边界

- 签名（`gpgsig` 或 `gpgsig-sha256` 头）覆盖提交对象的编码内容，其中包含 tree/parent 的 OID。跨哈希转换重算全部 OID，签名**必然**无法覆盖新编码——这是数学事实，不是缺陷。
- 签名字节本身只是提交头数据，verbatim 保留后：签名块逐字节不变，但其语义指向旧哈希编码。
- **sha256 仓库**中签名头名为 `gpgsig-sha256`（签名载荷 = 该提交的 sha256 编码，减去签名头）；**sha1 仓库**中为 `gpgsig`。Git 的哈希过渡设计定义了该机制，详见参考链接。
- **在纯 sha1 仓库中，git 无法验证 `gpgsig-sha256` 签名**：`verify-commit` 视为无签名（`%G?` 为 N），因为没有翻译表无法还原 sha256 编码。若要让签名可验证：
  - 对全部历史重签：`rebase --exec 'git commit --amend --no-edit -S'`，代价是签名时间戳变为重签时刻；
  - 或接受“字节保留但不可 git 内验证”，改用离线方式验证。

## 离线验证签名字节完好

用源仓库还原签名载荷、配合目标仓库的签名字节跑 gpg：

```bash
# 1. 源提交内容剥掉 gpgsig-sha256 头（续行以空格开头）作为载荷
# 2. 目标仓库同名提交提取 gpgsig-sha256 头作为签名
# 3. gpg --verify sig.asc payload.txt
```

批量验证用 marks 配对：

```bash
git -C <源> fast-export --all --signed-commits=verbatim --export-marks=marks-src
git -C <目标> fast-import --export-marks=marks-dst < <导出流>    # 幂等，对象确定性哈希
```

marks 文件格式 `:<编号> <oid>`，两侧按编号配对后逐提交比对/验证。

## 推送目标平台的接受性

- **GitHub（sha1）**：接受带 `gpgsig-sha256` 头的历史（2026-09 实测）；新提交以 `gpgsig`（sha1 格式）签名则正常 Verified。GitHub 至今不支持 sha256 仓库推送，`extensions.compatObjectFormat` 变通亦无效，详见参考链接。[时效性]
- **Codeberg**：原生支持 sha256 仓库；现有 sha256 仓库无法接收 sha1 对象（改历史需另建目标格式仓库，改名+新建策略见 codeberg-github-migration 技能）。
- push 前先试推小提交探路，被拒无损可回退。

## compatObjectFormat 的现状

`extensions.compatObjectFormat` 设计目标是仓库内双向翻译（含“compat 格式下验证签名”），但截至 Git 2.55 实测：fast-import 绕过翻译表写入会导致仓库半残（连新提交都报 `Failed to convert`），跨格式 fetch 互操作也未实现。**不要在生产路径使用**，走 fast-export/import 管道。

## 参考资料

- Git hash 过渡设计（`gpgsig-sha256` 头语义、兼容机制）：<https://git-scm.com/docs/hash-function-transition>
- GitHub 拒收 sha256 push 实证（含 compatObjectFormat 失败记录）：<https://github.com/orgs/community/discussions/154056>
- fast-export/fast-import 文档：<https://git-scm.com/docs/git-fast-import>
