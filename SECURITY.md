# 安全与公开发布检查

本仓库只公开方法说明和脱敏示例，不应成为个人大肥鱼数据的备份仓库。

## 永远不要提交

- 抖音/QQ 登录 Cookie、二维码、短信验证码和浏览器 Profile；
- 模型 API Key、NapCat Token、抖音桥令牌；
- QQ 号、抖音号、UID、sec_uid、手机号和邮箱；
- DSH session ID、私聊/群聊记录、人格记忆、关系状态；
- 私信图片、截图、视频、音频转写和运行日志；
- 带真实值的 `douyin-dm.json`、`qq-bridge-config.json` 或 `.env`；
- 备份压缩包和故障现场目录。

## 发布前检查

1. 在一个全新目录中只复制计划公开的文档和示例，不要直接对运行目录执行 `git init`。
2. 搜索可疑字段和真实路径：

   ```powershell
   rg -n -i "api[_-]?key|token|cookie|session|uid|sec_uid|qq|手机号|令牌|主人" .
   ```

3. 检查待提交文件：

   ```powershell
   git status --short
   git diff --cached --stat
   git diff --cached
   ```

4. 提交后检查完整历史，而不只是当前文件：

   ```powershell
   git log --stat --oneline
   git grep -n -i "token\|cookie\|session" $(git rev-list --all)
   ```

5. 如果密钥曾经进入 Git 历史，先吊销/轮换密钥，再清理历史。单纯删除当前文件不够。

## 运行边界

- HTTP 接口只绑定 `127.0.0.1`。
- 每个本机接口使用独立随机令牌。
- 模型只看到许可会话，不读取整个私信列表。
- 遇到验证码或风控立即停止自动化并熔断。
- 所有“已发送”结论都区分发送端回执和收件端实际收到。
- 不把 DSH、NapCat 管理后台或抖音桥端口直接映射到公网。

## 报告问题

如果问题包含真实账号、令牌、Cookie、聊天记录或截图，请不要提交到公开 Issue。先制作可复现的脱敏样例。

