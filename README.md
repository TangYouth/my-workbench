# My Workbench

一个基于 Vue 3 + Vite 的 Codex 工作台，用来集中处理常用的本地配置与 Skills 查看。

## 项目简介

当前工作台聚焦两个核心功能：

- `Codex 配置切换`
- `Codex Skills 查看`

界面整体采用黑客终端风格：

- 深色背景与网格化氛围
- 荧光绿 / 赛博蓝双主色
- 类终端字体与状态指示
- 面板式信息组织与悬浮说明

## 功能说明

### 1. Codex 配置切换

工作台会读取本地 `.codex` 目录中的配置备份文件，并展示可切换的配置列表。

配置规则：

- 默认配置目录：
  - Linux / macOS: `~/.codex/`
  - Windows: `%USERPROFILE%\.codex\`
- 一个配置需要同时存在两类文件：
  - `auth.json.xxx.bak`
  - `config.toml.xxx.bak`
- 某些系统会自动补扩展名，像 `auth.json.xxx.bak.json`、`config.toml.xxx.bak.toml` 也会被兼容识别。
- 点击切换时，会先把当前 `auth.json` 和 `config.toml` 备份成 `recent`，再写入目标配置内容。
- 当前活动配置会通过对比 `auth.json` 内容自动识别。

注意：

- 修改配置后，需要重启 Codex 桌面端才能生效。
- 浏览器首次访问本地目录时需要用户授权，授权后会尝试在后续刷新中自动恢复。

### 2. Codex Skills 查看

工作台会读取本地 skills 目录，并展示所有包含 `SKILL.md` 的 skill 文件夹。

Skills 规则：

- 默认 Skills 目录：
  - Linux / macOS: `~/.codex/skills/`
  - Windows: `%USERPROFILE%\.codex\skills\`
- Skills 名称使用文件夹名称。
- 只有包含 `SKILL.md` 的文件夹才会被识别为有效 skill。
- 点击某个 skill 后，会弹出终端风格面板展示 `SKILL.md` 原文内容。
- 弹窗内支持一键复制 `SKILL.md` 原文。


## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Type-Check, Compile and Minify for Production

```sh
npm run build
```
