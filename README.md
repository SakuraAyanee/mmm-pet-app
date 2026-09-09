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

#### 角色素材渲染

✨ 视觉：将原来的爪印方块替换为透明背景的田中摩美美角色 PNG。`PetAvatar` 使用 `import` 引入图片，再通过 `<img>` 渲染；PNG 的 Alpha 通道决定哪些像素可见，透明背景不会显示为方形底色。

📐 尺寸：角色原图为 `941 × 1672`，宽高比约为 `0.563`。CSS 使用 `width: min(72vw, 22rem)` 与 `height: min(68vh, 31rem)` 限制显示范围，再以 `object-fit: contain` 保持比例完整显示，避免图像拉伸或裁切。

🎨 样式：点击容器保留为语义化的 `<button>`，因此原有的 `@click`、键盘焦点与随机移动逻辑无需改变；容器改为透明背景、零内边距，避免按钮默认样式产生额外边缘。`drop-shadow()` 会依据透明像素的轮廓生成阴影，比矩形 `box-shadow` 更贴合人物边缘。

📝 思考：目前点击区域仍是角色所在的矩形容器，而非人物轮廓的像素级命中区。这种实现更简单稳定，适合作为桌面宠物的第一版交互。

🖥️ 桌面化：为现有 Vue + Vite 前端初始化 Tauri 2，新增 `src-tauri/` Rust 原生端目录；配置 Vite 固定使用 5173 端口并忽略监听 `src-tauri/`。已通过 `npx tauri dev` 在 Windows 原生窗口中运行。

#### 透明无边框窗口与拖动柄

✨ 窗口：在 `tauri.conf.json` 中启用 `transparent: true` 与 `decorations: false`，并关闭 `shadow`。同时将网页根节点、`body` 和宠物容器背景设置为 `transparent`，使透明效果从原生窗口贯穿至 Vue 页面。

🧭 操作：无边框窗口失去系统标题栏，因此在角色右上角增加了一个三横线拖动柄。它默认隐藏，仅在鼠标悬停宠物或组件获得键盘焦点时淡入显示。

⚙️ 最终实现：拖动柄在 `mousedown`（鼠标左键按下）时调用 Tauri 的 `getCurrentWindow().startDragging()`；角色本体仍保留 `@click="movePet"` 的随机移动互动。能力配置中额外授予 `core:window:allow-start-dragging` 权限，使前端能够请求原生窗口开始拖动。

#### 拖动失效复盘：光标变成手势，窗口却没有移动

🐛 现象：最初给三横线拖动柄添加 `data-tauri-drag-region="deep"`，再尝试让按钮与每一条横线直接拥有 `data-tauri-drag-region`。鼠标移上去会显示拖动柄，光标也会变为手势，但按住并移动鼠标时，原生窗口位置不变。

🔍 排查：`cursor: grab` 是 CSS 样式，只能说明网页把鼠标显示成“可拖动”，不能证明 Tauri 收到了拖动指令。排查时还发现正在运行的 `src-tauri/target/debug/app.exe` 会锁定同名文件，导致新的调试构建无法覆盖旧程序；因此测试前要先关闭旧窗口，并使用 `npx tauri dev` 重新启动。仅运行 `npm run dev` 会打开浏览器页面，浏览器没有 Tauri 原生窗口，当然也无法移动桌面窗口。

✅ 修复：不再把窗口拖动交给 HTML 属性自动识别，而是在拖动柄的 `@mousedown.left.prevent` 中明确调用 `getCurrentWindow().startDragging()`。这个调用会从 Vue 前端发给 Tauri 的 Rust 原生窗口层；`src-tauri/capabilities/default.json` 中的 `core:window:allow-start-dragging` 则是对此操作的授权。这样“网页按钮被按下”与“Windows 窗口进入拖动状态”之间的路径是明确且可检查的。

🧠 记忆点：界面的视觉反馈、浏览器 DOM 事件和桌面窗口操作属于三个不同层次。看到手势不等于窗口能移动；需要确认事件处理函数真正调用了原生 API，并且该 API 已具备 Tauri capability 权限。

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
