# AGENT_COMMS_PROTOCOL

> 统一约束：所有 pixelhd 相关 CLI（android-cli / game-cli / content-cli / maps-cli / infra-cli / qa-cli，以及将来新增）必须通过这个仓库进行高频通讯，而不是在对话里输出长篇报告给用户。

## 1. 会话开始前

- 每次启动或继续一个任务前，必须在各自 CLI 里执行：
  - `cd C:/Users/KAKA/pixelhd-agent-reports`
  - `git pull`
- 只从这里读取总监命令（director-commands-*.md）和其他 CLI 的最新报告。

## 2. 行为粒度与报告

- 任何**非 trivial** 行为（例如：改一处逻辑、跑一组测试、调整一个地图、验证一次 Android 构建）完成后，必须：
  1. 在自己目录下追加/更新一份短报告（P*-*.md）。
  2. 把当前动作的关键信息写进报告，而不是写给用户：
     - 做了什么
     - 改了哪些文件（路径）
     - 如何复现/验证
     - 结果如何 / TODO 是什么
  3. 通过 git 记录行为：
     - `cd C:/Users/KAKA/pixelhd-agent-reports`
     - `git add .`
     - `git commit -m "<your-id>: <简短描述>"`

- 不允许等用户说“上传报告”才提交；每次行为结束都必须自己提交报告并 commit。

## 3. 对话输出约束

- 与用户的对话只保留**极简摘要**：
  - 1–3 行说明“已拉取 / 已更新报告 / 提交了什么”。
  - 附带报告文件路径，方便需要时查看详细内容。
- 禁止在对话中粘贴长篇报告或大段日志；把细节写进各自目录的 .md 报告里即可。
- 禁止在对话中询问“要不要上传报告/commit”；一律自动按照本协议执行。

## 4. 代码仓库协作

- pixelhd 代码仓库路径：`C:/Users/KAKA/pixelhd`。
- 改动代码时：
  - 自己决定是否 commit 到 pixelhd（根据当前任务和总监命令）；
  - 所有行为必须在 agent-reports 仓库里留下对应报告（哪怕代码暂未 commit）。

## 5. 目录边界

- 每个 CLI 只允许修改自己目录下的文件和根目录通用文件（例如本协议、director-commands-*.md）。
- 不允许擅自修改其他 CLI 目录下的报告或 WHOAMI 文件。

> 若本协议与将来的 director-commands 有冲突，以 **最新的 director-commands** 为准，但“高频通过 git 报告进度、对话只给简短摘要” 这一点始终有效。
