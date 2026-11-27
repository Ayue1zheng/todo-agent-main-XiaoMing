# 小明智能体 · 中文待办清单

一个用中文对话管理待办的智能体。用户说「帮我记一下买牛奶」，小明就把事项加入清单；说「牛奶买好了」，它就只把对应任务标记为完成。任务保存在本地，刷新页面或重启服务后仍可查询。

底层使用 DeepSeek Function Calling：模型负责理解中文意图，真正的增删改查由服务端工具执行，避免模型「口头声称已完成」却没有修改数据。

## 功能

- 通过聊天创建、查看、标记完成、删除任务
- 自然语言简称匹配，例如用「牛奶」指代「买牛奶」
- 匹配不到或匹配到多条时先说明或追问，不猜测着修改数据
- 一句话创建多条任务
- 修改任务标题、把已完成任务恢复为待办
- 清单展示优化：待办在前、标明完成状态、显示剩余待办数、空清单有明确提示
- 任务持久化到本地 JSON，重启服务后仍可查询
- 普通寒暄和能力咨询不会修改任务

## 快速开始

需要 Node.js 24.12 或更高版本。

```sh
npm ci
```

把 `.env.example` 复制为 `.env`，填入 DeepSeek API Key：

```sh
DEEPSEEK_API_KEY=你的密钥
```

启动服务：

```sh
npm run dev
```

浏览器打开 <http://127.0.0.1:3000>，发送「帮我记一下买牛奶」即可开始。

> Windows PowerShell 如果提示「无法加载文件 npm.ps1，因为在此系统上禁止运行脚本」，改用 `npm.cmd run dev`，或先执行 `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` 临时放开当前窗口。

密钥只保存在本地 `.env`，已被 `.gitignore` 排除，不会进入仓库。

## 验收对话

| 用户对小明说 | 预期结果 |
| --- | --- |
| 帮我记一下买牛奶 | 新增待办「买牛奶」 |
| 再记一条取快递 | 清单里多出「取快递」 |
| 我有哪些任务？ | 列出两条任务，能看出各自完成状态 |
| 牛奶买好了 | 「买牛奶」变已完成，「取快递」仍待办 |
| 取快递这条删掉吧 | 删除「取快递」 |
| 再给我看看清单 | 只剩已完成的「买牛奶」 |

重启服务后再问一次清单，仍应看到已完成的「买牛奶」，用于验证持久化。

## 实现思路

1. 在 [server/agent.ts](server/agent.ts) 中定义四个工具：`list_tasks`、`create_task`、`update_task`、`delete_task`。
2. 每轮把对话历史和工具定义交给 DeepSeek；模型返回工具调用后，由服务端执行工具，再把结果交还模型，直到模型生成最终回复。
3. 支持同一轮返回多个工具调用，按顺序执行；最多循环 12 轮，防止异常响应导致死循环。
4. 工具参数用 Zod 在服务端二次校验，不信任模型输出。无效 JSON、空标题、非法字段、不存在的 id 都会作为工具错误反馈给模型。
5. `createReply(tasks, getClient)` 注入与 HTTP 任务 API 相同的 `TaskStore`，聊天和 `/api/tasks` 共享同一份数据，不重复实现存储。
6. 任务变更必须经过工具执行，模型不能只回复「已完成」而不改数据。

## 目录结构

| 路径 | 用途 |
| --- | --- |
| [server/agent.ts](server/agent.ts) | 智能体实现：提示词、工具定义、参数校验、工具调用循环 |
| [server/index.ts](server/index.ts) | 启动服务，向智能体注入任务存储 |
| [server/agent.test.ts](server/agent.test.ts) | 智能体测试：验收对话、多工具调用、边界情况 |
| [server/model.ts](server/model.ts) | DeepSeek 客户端与模型配置 |
| [server/app.ts](server/app.ts) | 接收对话历史，挂载聊天与任务 API |
| [server/tasks.ts](server/tasks.ts) | 任务 API 的输入校验和 HTTP 响应 |
| [server/task-store.ts](server/task-store.ts) | 读写本地 JSON 任务清单 |
| [server/openapi.yaml](server/openapi.yaml) | 任务接口定义 |
| [src/App.tsx](src/App.tsx) | 聊天界面 |

## 测试

```sh
npm test
npm run typecheck
npm run lint
npm run build
```

`npm test` 共 8 项，覆盖六句验收对话、多轮与并行工具调用、无效参数、不存在的任务、普通寒暄、任务 API 和 JSON 持久化。测试使用模拟模型，不会消耗 API 额度。

真实模型验证记录见 [SUBMIT.md](SUBMIT.md)。

## 已知限制

- 本地 JSON 存储不支持多进程并发写同一个文件。
- 批量操作中若后续工具调用失败，已成功的操作不会自动回滚。
- 聊天历史不跨请求保存工具调用信息，每轮按需重新查询清单，避免使用过期 id。
