# ai-daily-cron

定时中转：每天 08:00（缅甸）触发私有仓库 [`melvin688/ai-daily-engine`](https://github.com/melvin688/ai-daily-engine) 的 `daily-brief` workflow。

## 为什么存在

GitHub 免费个人账号的**私有**仓库不会触发 `schedule` 事件——workflow 显示 `active`、cron 表达式正确、Actions 状态页全绿，但运行记录里 `schedule` 事件始终为 0。

实测：一个 `*/5` 的探针 workflow 推上去后 14 分钟、3 个调度点、0 次运行；手动 `workflow_dispatch` 立刻成功。

public 仓库的 `schedule` 正常，所以把定时器挪到这里。这个仓库没有业务代码，只发一次 HTTP 请求。

## 一次性配置

需要在**本仓库**的 `Settings → Secrets and variables → Actions` 里加一个 `DISPATCH_TOKEN`：

1. 打开 https://github.com/settings/personal-access-tokens/new
2. Token name 填 `ai-daily-cron-dispatch`
3. Expiration 选 1 年（到期会静默失败，建议同时设个提醒）
4. Repository access → **Only select repositories** → 只勾 `ai-daily-engine`
5. Permissions → Repository permissions → **Actions: Read and write**，其余保持 No access
6. 生成后复制，粘到本仓库的 `DISPATCH_TOKEN` secret 里

命令行等价操作（在本地终端跑，token 不经过对话）：

```bash
gh secret set DISPATCH_TOKEN --repo melvin688/ai-daily-cron
```

## 调度时间

| cron (UTC) | 缅甸时间 | 用途 |
|---|---|---|
| `30 1 * * *` | 08:00 | 主 slot |
| `30 4 * * *` | 11:00 | 兜底 |

两个 cron 都会触发同一个 workflow。私有仓库在投递前会检查 `out/state.json` 的 `last_run`，**当天已投递则跳过**，所以兜底不会重复发消息。需要强制重发时，在私有仓库手动触发并勾 `force=true`。

## 排错

手动跑一次本仓库的 workflow（Actions → trigger-daily-brief → Run workflow），看日志里的 HTTP 码：

| 现象 | 原因 |
|---|---|
| `HTTP 204` | 正常，私仓已开始运行 |
| `HTTP 401` | token 过期或被撤销，重新生成 |
| `HTTP 403` | token 没给 `Actions: Read and write`，或者没勾对这个仓库 |
| `HTTP 404` | token 的 Repository access 里没包含 `ai-daily-engine` |

GitHub 的定时触发通常有 5–15 分钟延迟，属于正常范围。
