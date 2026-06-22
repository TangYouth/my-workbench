# My Workbench

一个 Vue 3 + Vite 构建的本地 Codex 工作台，采用黑客终端 / cyberpunk system panel 风格，集中提供 Codex 管理与 Todo 管理。

## 功能

- `Codex`：切换本地 `.codex` 配置备份，查看并复制 `SKILL.md` 内容。
- `Todo`：新增待办、筛选状态、调整优先级、查看任务详情与进度。

## 说明

- Codex 配置默认读取 `~/.codex/`，配置备份需成对存在：`auth.json.xxx.bak` 与 `config.toml.xxx.bak`。
- 切换配置会先备份当前 `auth.json` / `config.toml` 为 `recent`，再写入目标配置；切换后需要重启 Codex 桌面端。
- Todo 数据存储在浏览器 `localStorage`，key 为 `codex-workbench.todo.tasks`。
- 已完成 Todo 会记录完成时间；页面打开时会自动清理完成超过 `120h` 的任务。

## 开发

```sh
npm install
npm run dev
```

## 构建

```sh
npm run build
```
