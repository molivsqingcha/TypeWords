# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TypeWords（词文记）— 开源英语单词和文章练习工具。支持跟写、听写、默写、自测等多种练习模式，内置 FSRS 间隔重复算法。主站 <https://typewords.cc>。

## Commands

```bash
pnpm install          # 安装依赖
pnpm dev              # 启动 Nuxt 开发服务器 (localhost:5567)
pnpm build            # Nuxt 静态构建 (pnpm -F @typewords/nuxt static)
pnpm build-web        # 等同于 pnpm build
pnpm build-vscode-web # 构建 VSCode Web 版
pnpm dev-vscode-web   # 开发 VSCode Web 版
```

- 项目使用 pnpm workspace，无 lint/test 命令
- **Agent 模式下改完代码后不要运行 Nuxt 生产构建来验证**（见 .cursor/rules/agent-no-nuxt-build-verify.mdc）
- 使用 husky + commitizen (cz-conventional-changelog) 管理提交信息

## Architecture

### Monorepo Structure (pnpm workspace)

```
apps/
  nuxt/              # 主 Web 应用 (Nuxt 4 + Vue 3)
    app/
      pages/         # 路由页面
      layouts/       # 布局 (default, empty)
      plugins/       # 客户端/服务端插件
      components/    # 页面级组件
      assets/        # CSS 样式
    server/          # Nitro API routes
    i18n/locales/    # 多语言 JSON 文件 (14 种语言)
  vscode-web/        # VSCode Webview 版本 (Vite + Vue 3)
  vscode/            # VSCode Extension (TypeScript)
packages/
  core/              # 核心业务逻辑
  base/              # 基础 UI 组件库
  libs/              # 工具库 (翻译、qs 解析等)
```

### 包依赖关系

`core` → `base` + `libs`（core 依赖 base 和 libs）

### packages/core 内部结构

```
src/
  stores/            # Pinia 状态管理
    base.ts          # 词典/书籍/学习进度数据
    setting.ts       # 用户设置 (快捷键、音效、练习模式等)
    practice.ts      # 练习阶段状态 (计时、阶段切换)
    runtime.ts       # 运行时状态 (全局 loading、路由数据)
    user.ts          # 用户登录相关
  components/        # 业务组件
    word/            # 单词练习组件 (TypeWord.vue 为核心)
    article/         # 文章练习组件
    list/            # 词库/书籍列表
    setting/         # 设置面板
    dialog/          # 弹窗类
  hooks/             # 可复用逻辑
    event.ts         # 键盘事件监听 + 快捷键分发
    dict.ts          # 词典数据获取
    sound.ts         # 音效系统
    theme.ts         # 主题切换
    fsrs.ts          # FSRS 间隔重复算法
    article.ts       # 文章练习逻辑
    translate.ts     # 翻译服务
  apis/              # API 请求 (dict, user, member, words)
  utils/             # 工具函数
    eventBus.ts      # 基于 mitt 的事件总线
    cache.ts         # 本地缓存 key 定义
    supabase.ts      # Supabase 客户端
    word-test.ts     # 单词测试工具
  config/
    env.ts           # 环境配置、快捷键映射、练习阶段映射
    auth.ts          # 认证配置
  composables/       # Vue composables
    useDataSyncPersistence.ts  # 数据同步持久化
    usePracticePersistence.ts  # 练习进度持久化
  types/             # TypeScript 类型定义
    types.ts         # Word, Dict, Article, PracticeData 等核心类型
    enum.ts          # 枚举 (练习模式、阶段、快捷键等)
```

### 关键数据流

1. **用户设置**: `idb-keyval` (IndexedDB) 本地持久化 → `settingStore` → 可选 Supabase 同步
2. **词典数据**: JSON 文件 (public/list/) → `baseStore` → 本地 IndexedDB 缓存
3. **练习进度**: `practiceStore` + `practiceData` → `usePracticePersistence` 自动保存到 IndexedDB
4. **键盘事件**: `useStartKeyboardEventListener` 拦截键盘 → 区分快捷键/输入字符 → `mitt eventBus` 分发
5. **FSRS 算法**: `ts-fsrs` 库管理单词复习间隔，数据存在 `baseStore.fsrsData`

### 练习模式 (WordPracticeMode)

- **System** (系统模式): 跟写 → 听写 → 默写 → 自测旧词 → 听写旧词 → 默写旧词
- **Free** (自由模式): 只跟写
- **IdentifyOnly** / **DictationOnly** / **ListenOnly**: 单一模式
- **Shuffle** (随机复习): 随机抽取已学单词
- **Review** (复习): 自测 → 听写 → 默写

### 路由页面

```
/                         # 首页 (引导页)
/words                    # 单词学习主页
/dict-list                # 词库列表
/dict                     # 词典管理
/practice-words/:id       # 单词练习页 (核心)
/words-test/:id           # 单词测试页
/articles                 # 文章学习主页
/book-list                # 书籍列表
/book/:id                 # 书籍详情
/practice-articles/:id    # 文章练习页
/setting                  # 设置页
/login                    # 登录
/user                     # 用户中心
```

### 关键约定

- 使用 `unplugin-auto-import` + `unplugin-vue-components`，Vue API 和组件自动导入
- 使用 `vue-macros` (`$ref`, `$computed`, `$defineModel` 等 Reactivity Transform)
- CSS 使用 UnoCSS (类 Tailwind) + Scoped SCSS
- 快捷键定义在 `packages/core/src/config/env.ts` 的 `DefaultShortcutKeyMap`
- 练习阶段控制流在 `apps/nuxt/app/pages/(words)/practice-words/[id].vue`
