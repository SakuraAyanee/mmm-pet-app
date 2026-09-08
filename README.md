# 🐾 AI Desktop Pet (开发中 / WIP)

> 这是一个基于 Vue 3 + TypeScript 重构的桌面 AI 宠物项目。
> 核心目标是将早期的桌面宠物架构升级为现代前端跨平台技术栈，并计划最终接入 Live2D 与大语言模型（LLM + WhisperX）实现深度交互。

## 🛠️ 核心技术栈
- **核心框架**：Vue 3 (Composition API) + Vite
- **开发语言**：TypeScript
- **桌面容器**：Tauri 2（Windows 桌面端已初始化）
- **视觉与动作**：CSS3 / `pixi-live2d-display` (计划引入阶段)
- **AI 交互**：大语言模型 API + 语音识别与合成 (计划引入阶段)

## 🚀 快速启动
在终端运行以下命令即可在本地浏览器查看当前进度：
```bash
npm install
npm run dev
```

在完成 Tauri 环境安装后，可使用下面的命令以原生桌面窗口启动应用：

```bash
npx tauri dev
```

## 📅 独立开发日记 (Update Diary)
积沙成塔。记录每天下班后的微小进展与重构思考。

### 2026-09-09

✨ 新增：创建 `PetAvatar` 组件，并在 `App.vue` 中引入。组件使用 `PetState` interface 约束宠物的 `x`、`y` 坐标与 `mood` 情绪状态。

✨ 交互：实现点击宠物方块事件。每次点击会随机更新坐标与背景色，将情绪设为 `happy`，并记录被摸次数；CSS `transition` 使移动和变色在 300ms 内平滑完成。

🖥️ 桌面化：为现有 Vue + Vite 前端初始化 Tauri 2，新增 `src-tauri/` Rust 原生端目录；配置 Vite 固定使用 5173 端口并忽略监听 `src-tauri/`。已通过 `npx tauri dev` 在 Windows 原生窗口中运行。

🛠️ 环境记录：Windows 上安装 Rust 时，`winget` 出现 `0x8a15000f`（软件源数据缺失）。改用 Rust 官方 `rustup-init.exe` 完成安装，并通过 Visual Studio Community 安装器补齐“使用 C++ 的桌面开发”组件。重新打开终端后，使用 `rustc --version` 和 `cargo --version` 验证环境。

📝 思考：Vue 负责界面和响应式交互；Tauri 用 Rust 提供原生窗口和系统能力。现阶段先确认桌面壳与前端正常协作，后续再配置透明、无边框、置顶等桌面宠物窗口特性。

### 2026-09-08
🚧 基建：完成 Vue 3 + TS + Vite 基础脚手架搭建。

⚙️ 配置：明确 .gitignore 规则，连通 GitHub 远程仓库，完成 chore: 初始化 Git 与 TS 环境 的首个里程碑。

## 待办记录板 (Template)
✨ 新增：[例如：创建第一个带有 TS 状态的宠物组件]

🐛 修复：[例如：解决组件传参时的类型推导红线]

♻️ 重构：[例如：清理默认模板，梳理目录结构]

📝 思考：[例如：记录今天在 Obsidian 里复盘的泛型知识点]
