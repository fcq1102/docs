# GitHub 文档仓库搭建与踩坑记录

> **日期**：2026-11-26
> **仓库**：`fcq1102/docs`（私有）
> **环境**：macOS Intel + zsh + WorkBuddy + GitHub Desktop
> **性质**：工具 / 环境搭建经验复盘

## 一、目标

把工作文档统一托管到 GitHub，实现三个能力：

1. **版本追踪**：每份文档的每次变更都是一次 commit，历史可回溯
2. **跨终端同步**：本地与线上一致，任意一端都能看到全部文档
3. **自动化提交**：文档写完后一条命令完成提交推送，不依赖人工操作

## 二、最终方案

**一句话**：本地仓库 + git 通道提交推送；凭证走钥匙串 GitHub Desktop token（API 与 git 双通道并行）；完全绕开连接器的写权限限制。

```
┌─────────────────────────────────────────────────┐
│  文档写入（本地 / WorkBuddy）                      │
└──────────────────┬──────────────────────────────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
   git 通道（主）          API 通道（备）
   git_docs.py            gh_docs.py
   add→commit→            list/read/
   pull→push              write/delete
        │                     │
        └──────────┬──────────┘
                   ▼
      钥匙串 token（gho_，实时读取）
                   ▼
        fcq1102/docs（私有仓库）
```

**关键设计决策**：

| 决策 | 原因 |
|------|------|
| 凭证从钥匙串实时读取 | 不明文落盘，安全；token 只在内存中使用 |
| git 清代理走直连 | 系统代理对 `github.com:443` 间歇性 502 / HTTP2 错误 |
| pull --rebase 后再 push | 避免非快进冲突，保证提交链线性 |
| API 通道保留作备 | 网络异常时仍能读写仓库 |

## 三、改动清单

| 项目 | 内容 | 位置 |
|------|------|------|
| 仓库初始化 | README.md（带目录索引） | 根目录 |
| 记忆汇总 | `agent.json`（跨终端信息与长期记忆，敏感信息脱敏） | 根目录 |
| 四大业务目录 | 运维 / 项目交付 / 合规 / 接口文档（各带 README） | 一级目录 |
| 两个新目录 | 学习资料 / 电商相关（各带 README） | 一级目录 |
| git 通道脚本 | `git_docs.py`（add→commit→pull→push） | `.workbuddy/scripts/` |
| API 通道脚本 | `gh_docs.py`（list/read/write/delete，支持中文路径） | `.workbuddy/scripts/` |
| 噪声清理 | `.gitignore` 忽略 `.DS_Store` | 根目录 |

**仓库最终结构**：

```
README.md
agent.json
├── 运维/           OMNIHUB 运维、月报周报、服务器维度、账号权限
├── 项目交付/       甲方周报月报、CPF 冠军项目、备案进度、工作总结
├── 合规/           ISO 27001、SOC 2、供应商尽调、安全方案、等保备案
├── 接口文档/       OMS↔M3 接口、CPF OpenAPI、信用余额/返利/SSO 评审
├── 学习资料/       技术笔记、工具心得、环境搭建记录（本文档所在）
└── 电商相关/       B2B 商城项目、电商业务规则、OMS/WMS 流程
```

## 四、踩坑记录

### 坑 1：GitHub 连接器无写权限（403）

- **现象**：`create_or_update_file` / `push_files` 均返回 `403 Resource not accessible by integration`，连创建仓库都失败。
- **根因**：WorkBuddy 的 GitHub App 应用级权限被限制，只有读权限，没有写权限。
- **尝试**：反复重新授权、改仓库可见性——**均无效**。
- **结论**：这是 App 层面的设计限制，不是仓库设置问题，重连解决不了。

### 坑 2：私有仓库对连接器返回 404

- **现象**：`fcq1102/docs` 明明存在（登录态可见），连接器读目录却是 404；匿名 curl API 也是 404。
- **根因**：GitHub 对应用无权访问的私有仓库一律伪装成 404（与"不存在"表现一致）；OAuth 授权范围最初未包含私有仓库权限。
- **排查技巧**：用 `409 Git Repository is empty` 与 `404 Not Found` 区分——前者说明**能访问**（只是空仓库），后者才是**无权访问**。
- **解决**：重新授权后读权限恢复（404 → 409）；写权限仍卡在坑 1。

### 坑 3：连接器 ID 用简称查不到

- **现象**：`list_installed_plugins(pluginId="github")` 返回空，以为没连上。
- **根因**：系统内完整 ID 是 `github@workbuddy-connector-plugins-official`，简称匹配不到。
- **解决**：用完整 ID 查询；或直接用 MCP 工具调用验证是否连通。

### 坑 4：gh CLI 装不上

- **现象**：`brew install gh` 卡死、被 SIGTERM（137）；官方二进制下载全部失败。
- **根因**：
  - Intel Mac + 新版 macOS 组合下 Homebrew 没有 gh 的预编译 bottle，只能源码编译，极慢；
  - `github.com` release 下载通道被网络全面拦截（直连 HTTP 000，代理 CONNECT 502）。
- **尝试**：华为云 / 清华 / 中科大镜像、多个 ghproxy 代理、git 协议、浏览器下载——全部失败。
- **结论**：gh CLI 这条路在本机环境走不通，放弃。

### 坑 5：代理导致 git 通道 502 / HTTP2 framing 错误

- **现象**：`git clone` / `pull` / `push` 间歇性报 `CONNECT tunnel failed, response 502`、`Error in the HTTP2 framing layer`、`Empty reply from server`。
- **根因**：系统代理（`127.0.0.1:56917`）对 `github.com:443` 的 CONNECT 隧道不稳定；HTTP2 帧在代理下容易出错。
- **解决**（脚本已固化）：
  1. 清除 `https_proxy` / `HTTPS_PROXY` / `http_proxy` / `all_proxy` 等环境变量，**直连** github.com；
  2. 强制 `http.version = HTTP/1.1`，绕开 HTTP2 framing 问题。
- **验证**：清代理后 `git push` 一次成功（`bd0359e..10a6be0 main -> main`）。

### 坑 6：本地仓库是空的（线上有、本地看不到）

- **现象**：线上仓库有 7 次提交和一堆文件，本地 `/Users/fanchangqing/Documents/GitHub/docs` 只有 `.git`，零 commit。
- **根因**：所有文档都通过 API 直接写远程，从未同步本地；早期 `git clone` 因代理 502 失败，本地仓库一直空着。
- **解决**：网络恢复后 `git fetch origin main` 成功，`git merge FETCH_HEAD` 把远程全部内容拉到本地。
- **教训**：双通道并行时，必须保证本地仓库与远程定期同步，否则会出现"线上有、本地看不到"。

### 坑 7：中文路径导致 API 写入失败

- **现象**：通过 `gh_docs.py` 创建中文目录时报错。
- **根因**：中文路径未做 URL 编码，直接拼进请求 URL。
- **解决**：对路径逐段做 `urllib.parse.quote(s, safe="")`，并注意**先分离查询串**（`?ref=main`）再编码路径，避免把 `?` 一起编码掉。

### 坑 8：明文凭证的安全隐患

- **现象**：一度把 token 明文写入 `~/.git-credentials` 并设 `credential.helper = store`。
- **风险**：token 以明文躺在磁盘上，任何本机进程都能读取。
- **处理**：立即删除明文文件，恢复 `credential.helper = osxkeychain`；后续方案改为钥匙串实时读取 + 临时 `GIT_ASKPASS`（进程结束即删，不落盘、不进命令行历史）。

## 五、最终凭证方案

```
1. security find-generic-password -s "GitHub - https://api.github.com" -w
   → 读出钥匙串中 GitHub Desktop 登录 token（gho_ 开头，账号 fcq1102，带写权限）

2. 创建临时 askpass 脚本（/tmp，权限 700，内容仅 echo "$GH_TOKEN"）

3. git 环境注入：
   - GIT_ASKPASS=<临时脚本>
   - GH_TOKEN=<token>
   - 清除所有 *_proxy 环境变量
   - http.version = HTTP/1.1

4. 命令执行完毕后删除临时脚本
```

**安全边界**：token 不明文落盘、不进 shell 历史、不写进 `agent.json` 等文档。

## 六、使用指南

### git 通道（日常首选）

```bash
# 提交指定文件
python3 .workbuddy/scripts/git_docs.py "提交说明" 文件路径1 文件路径2

# 提交全部变更
python3 .workbuddy/scripts/git_docs.py "提交说明"
```

流程：`git add → git commit → git pull --rebase → git push`，自动处理冲突保护。

### API 通道（备用）

```bash
python3 .workbuddy/scripts/gh_docs.py list [目录]          # 列目录
python3 .workbuddy/scripts/gh_docs.py read <文件路径>       # 读文件
python3 .workbuddy/scripts/gh_docs.py write <路径> <本地文件> ["提交说明"]  # 写文件
python3 .workbuddy/scripts/gh_docs.py delete <路径>         # 删文件
```

## 七、经验总结

1. **先分清"读不到"还是"没权限"**：GitHub 对无权访问的私有仓库返回 404，与不存在一致；用 409 空仓库错误可以区分。
2. **App 权限限制无法靠重连解决**：应用级写权限缺失时，反复授权/改可见性都是徒劳，果断换凭证方案。
3. **本机已登录的桌面应用是宝库**：GitHub Desktop 的钥匙串凭证带完整写权限，绕开了连接器与 gh CLI 的双重限制。
4. **代理是 git 通道的头号敌人**：遇到 502 / HTTP2 framing，优先清代理 + 降 HTTP/1.1，而不是反复重试。
5. **凭证安全优先**：实时读取 + 临时传递，永远不落盘明文。
6. **双通道要同步**：API 写入后记得本地 fetch/merge，避免本地仓库落后导致后续 push 冲突。

---

> 本文档记录于 2026-11-26，git 通道端到端回归测试通过（commit `38aae89`）。
