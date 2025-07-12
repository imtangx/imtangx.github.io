---
title: 前端项目工程化Monorepo实践
date: 2025-07-13 00:33:22
tags: ['Monorepo']
---

> 基于 pnpm + Turbo + Tsup 的高性能 TypeScript Monorepo 架构实践

## 📚 目录

- [项目概述](#项目概述)
- [技术栈选择](#技术栈选择)
- [项目结构](#项目结构)
- [核心配置详解](#核心配置详解)
- [工程化工具链](#工程化工具链)
- [开发工作流](#开发工作流)
- [最佳实践](#最佳实践)
- [常见问题](#常见问题)

## 项目概述

本项目是一个高性能的 Markdown 编辑器，采用 Monorepo 架构，包含自研的词法分析器、语法解析器、AST 和渲染器。通过现代化的工程化配置，确保代码质量、开发效率和构建性能。

### 🎯 项目特点

- **Monorepo 架构**：多包管理，代码复用，统一工具链
- **TypeScript 全栈**：类型安全，开发体验优秀
- **高性能构建**：基于 esbuild 的 Tsup，构建速度提升 10-100 倍
- **完整的质量保障**：ESLint + Prettier + Husky + 自动化测试
- **现代化 CI/CD**：GitHub Actions 自动化流程

## 技术栈选择

### 包管理器：pnpm

**为什么选择 pnpm？**

```json
{
  "packageManager": "pnpm@8.15.9"
}
```

- **节省磁盘空间**：通过硬链接共享依赖，节省 50% 磁盘空间
- **安装速度快**：比 npm 快 2-3 倍
- **严格的依赖管理**：避免幽灵依赖问题
- **原生 Monorepo 支持**：workspace 功能强大

### 构建工具：Turbo + Tsup

**Turbo（Monorepo 任务调度）**
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    }
  }
}
```

- **增量构建**：只构建变更的包
- **并行执行**：多包同时构建
- **智能缓存**：避免重复构建

**Tsup（TypeScript 打包）**
```typescript
export default defineConfig({
  entry: ['src/index.ts'],
  format: ['cjs', 'esm'],    // 同时输出 CommonJS 和 ES Module
  dts: true,                 // 生成类型声明文件
  clean: true,               // 构建前清理
  sourcemap: true,           // 生成 source map
  target: 'es2020',          // 目标 ES 版本
});
```

## 项目结构

```
markdown-editor/
├── packages/                    # 核心库包
│   ├── shared/                 # 🔧 共享工具和类型
│   │   ├── src/index.ts       # 导出 logger, types 等
│   │   ├── package.json       # 包配置
│   │   ├── tsconfig.json      # TS 配置
│   │   └── tsup.config.ts     # 构建配置
│   ├── lexer/                 # 📝 词法分析器
│   ├── ast/                   # 🌳 抽象语法树
│   ├── parser/                # 🔍 语法解析器
│   ├── renderer/              # 🎨 渲染器
│   └── core/                  # 💎 核心逻辑
├── .github/workflows/          # 🚀 GitHub Actions
├── .husky/                    # 🪝 Git hooks
├── .vscode/                   # 💻 编辑器配置
├── package.json               # 根包配置
├── pnpm-workspace.yaml        # workspace 配置
├── turbo.json                 # 构建任务配置
├── tsconfig.json              # 根 TS 配置
├── eslint.config.js           # ESLint 配置
├── commitlint.config.js       # 提交规范配置
└── README.md                  # 项目文档
```

## 核心配置详解

### 1. 根目录 package.json

```json
{
  "name": "markdown-editor",
  "packageManager": "pnpm@8.15.9",        // 指定包管理器版本
  "workspaces": [                         // 定义 workspace 包
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "build": "turbo run build",           // 构建所有包
    "dev": "turbo run dev",               // 开发模式
    "lint": "eslint",                     // 代码检查
    "lint:fix": "eslint --fix",           // 自动修复
    "format": "prettier --write .",       // 格式化代码
    "typecheck": "turbo run typecheck",   // 类型检查
    "prepare": "husky"                    // Git hooks 初始化
  },
  "lint-staged": {                        // 预提交检查配置
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",                     // 先修复 lint 问题
      "prettier --write"                  // 再格式化代码
    ],
    "*.{json,md,yml,yaml}": [
      "prettier --write"                  // 格式化配置文件
    ]
  }
}
```

**关键字段解析：**
- `packageManager`：确保团队使用相同版本的包管理器
- `workspaces`：告诉 pnpm 哪些目录包含子包
- `lint-staged`：只对暂存区文件运行检查，提升性能

### 2. pnpm-workspace.yaml

```yaml
packages:
  - 'apps/*'      # 应用层包（如 React 应用）
  - 'packages/*'  # 库层包（如工具库）
```

**作用：**
- 定义 Monorepo 的包结构
- 支持 glob 模式匹配
- 支持包间依赖管理

### 3. 子包 package.json 配置

以 `packages/shared/package.json` 为例：

```json
{
  "name": "@markdown-editor/shared",      // 包名（带作用域）
  "version": "0.0.0",
  "main": "./dist/index.js",              // CommonJS 入口
  "module": "./dist/index.mjs",           // ES Module 入口
  "types": "./dist/index.d.ts",           // TypeScript 类型入口
  "exports": {                            // 现代模块导出配置
    ".": {
      "types": "./dist/index.d.ts",       // 类型声明优先级最高
      "import": "./dist/index.mjs",       // ES Module 导入
      "require": "./dist/index.js"        // CommonJS 导入
    }
  },
  "files": ["dist"],                      // 发布时包含的文件
  "scripts": {
    "build": "tsup",                      // 构建命令
    "dev": "tsup --watch",                // 开发模式（监听文件变化）
    "typecheck": "tsc --noEmit"           // 仅类型检查
  },
  "dependencies": {
    "@markdown-editor/shared": "workspace:*"  // workspace 内部依赖
  }
}
```

**exports 字段详解：**
- 现代 Node.js 和打包工具的标准
- 支持条件导出（types, import, require）
- 类型声明放在最前面确保类型优先

### 4. TypeScript 配置

**根目录 tsconfig.json：**
```json
{
  "compilerOptions": {
    "target": "es2016",                    // 编译目标
    "module": "esnext",                    // 模块系统
    "moduleResolution": "bundler",         // 模块解析策略
    "baseUrl": "./",                       // 基础路径
    "paths": {                             // 路径映射
      "@markdown-editor/shared": ["./packages/shared/src"],
      "@markdown-editor/lexer": ["./packages/lexer/src"]
    },
    "strict": true,                        // 严格模式
    "esModuleInterop": true,               // ES 模块互操作
    "skipLibCheck": true                   // 跳过库文件检查
  }
}
```

**子包 tsconfig.json：**
```json
{
  "extends": "../../tsconfig.json",       // 继承根配置
  "compilerOptions": {
    "outDir": "./dist"                     // 输出目录
  },
  "include": ["src/**/*"],                 // 包含的文件
  "exclude": ["dist", "node_modules"]      // 排除的目录
}
```

### 5. Tsup 构建配置

```typescript
// packages/shared/tsup.config.ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],               // 入口文件
  format: ['cjs', 'esm'],                // 输出格式
  dts: true,                             // 生成 .d.ts 文件
  clean: true,                           // 构建前清理
  splitting: false,                      // 禁用代码分割
  sourcemap: true,                       // 生成 sourcemap
  minify: false,                         // 开发阶段不压缩
  target: 'es2020',                      // 目标版本
  outDir: 'dist',                        // 输出目录
  external: [],                          // 外部依赖（shared 包没有）
  tsconfig: './tsconfig.json',           // TS 配置文件
});
```

## 工程化工具链

### 1. 代码质量：ESLint

**eslint.config.js（ESLint v9 配置）：**
```javascript
import js from '@eslint/js';
import tseslint from '@typescript-eslint/eslint-plugin';
import tsparser from '@typescript-eslint/parser';

export default [
  js.configs.recommended,                 // 基础 JS 规则
  {
    files: ['**/*.{js,jsx,ts,tsx}'],      // 匹配的文件
    languageOptions: {
      parser: tsparser,                   // 使用 TS 解析器
      parserOptions: {
        ecmaVersion: 'latest',
        sourceType: 'module',
        project: ['./tsconfig.json', './packages/*/tsconfig.json']
      },
      globals: {                          // 全局变量
        console: 'readonly',
        process: 'readonly'
      }
    },
    plugins: {
      '@typescript-eslint': tseslint      // TS 插件
    },
    rules: {
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/no-explicit-any': 'warn',
      'prefer-const': 'error',
      'no-var': 'error'
    }
  },
  {
    ignores: [                            // 忽略的目录
      'dist/**',
      'node_modules/**',
      '.turbo/**'
    ]
  }
];
```

### 2. 代码格式化：Prettier

**.prettierrc.json：**
```json
{
  "printWidth": 120,                     // 每行最大字符数
  "tabWidth": 2,                         // 缩进宽度
  "useTabs": false,                      // 使用空格缩进
  "semi": true,                          // 语句末尾加分号
  "singleQuote": true,                   // 使用单引号
  "trailingComma": "all",                // 尾随逗号
  "bracketSpacing": true,                // 对象括号空格
  "arrowParens": "avoid",                // 箭头函数参数括号
  "endOfLine": "lf"                      // 换行符类型
}
```

### 3. Git 工作流：Husky + lint-staged

**.husky/pre-commit：**
```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx lint-staged                          # 运行预提交检查
```

**.husky/commit-msg：**
```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx --no -- commitlint --edit $1         # 检查提交信息格式
```

### 4. 提交规范：commitlint

**commitlint.config.js：**
```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',     // 新功能
        'fix',      // 修复
        'docs',     // 文档
        'style',    // 格式
        'refactor', // 重构
        'perf',     // 性能优化
        'test',     // 测试
        'chore',    // 构建过程或辅助工具的变动
        'ci',       // CI配置
        'build',    // 构建系统
        'revert',   // 回滚
      ],
    ],
    'subject-case': [0], // 禁用主题大小写检查
  },
};
```

**提交信息格式：**
```
feat: 添加词法分析器核心功能
fix: 修复 AST 节点类型错误
docs: 更新 API 文档
style: 统一代码格式
```

### 5. CI/CD：GitHub Actions

**.github/workflows/ci.yml：**
```yaml
name: CI

on:
  push:
    branches: [master, main]
  pull_request:
    branches: [master, main]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          
      - name: Setup pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 8.15.9
          
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
        
      - name: Run linting
        run: pnpm run lint
        
      - name: Check formatting
        run: pnpm run format:check
        
      - name: Type check
        run: pnpm run typecheck
        
      - name: Build packages
        run: pnpm run build
```

### 6. 开发环境：VSCode 配置

**.vscode/settings.json：**
```json
{
  "editor.formatOnSave": true,           // 保存时自动格式化
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"   // 保存时自动修复 ESLint 问题
  },
  "typescript.preferences.includePackageJsonAutoImports": "auto",
  "eslint.workingDirectories": [         // ESLint 工作目录
    "./packages/*",
    "./apps/*"
  ]
}
```

**.vscode/extensions.json：**
```json
{
  "recommendations": [                   // 推荐扩展
    "esbenp.prettier-vscode",           // Prettier
    "dbaeumer.vscode-eslint",           // ESLint
    "ms-vscode.vscode-typescript-next", // TypeScript
    "yzhang.markdown-all-in-one"        // Markdown 支持
  ]
}
```

## 开发工作流

### 1. 项目初始化

```bash
# 1. 克隆项目
git clone <repository-url>
cd markdown-editor

# 2. 安装依赖
pnpm install

# 3. 构建项目
pnpm run build

# 4. 开发模式
pnpm run dev
```

### 2. 日常开发

```bash
# 开发模式（监听文件变化）
pnpm run dev

# 代码检查
pnpm run lint

# 自动修复
pnpm run lint:fix

# 格式化代码
pnpm run format

# 类型检查
pnpm run typecheck

# 构建项目
pnpm run build
```

### 3. 提交代码

```bash
# 添加文件
git add .

# 提交（会自动运行 lint-staged 检查）
git commit -m "feat: 添加新功能"

# 推送代码
git push
```

### 4. 添加新包

```bash
# 1. 创建包目录
mkdir packages/new-package

# 2. 初始化 package.json
cd packages/new-package
pnpm init

# 3. 配置包信息
# 编辑 package.json，参考其他包的配置

# 4. 创建源码目录
mkdir src
echo "export const hello = 'world';" > src/index.ts

# 5. 添加构建配置
# 复制其他包的 tsconfig.json 和 tsup.config.ts

# 6. 安装依赖并构建
cd ../..
pnpm install
pnpm run build
```

## 最佳实践

### 1. 包设计原则

**单一职责原则**
```
shared/    -> 通用工具和类型
lexer/     -> 词法分析
ast/       -> AST 定义和操作
parser/    -> 语法解析
renderer/  -> 渲染逻辑
core/      -> 核心整合
```

**依赖关系**
```
core -> parser + renderer
parser -> lexer + ast
renderer -> ast
lexer -> shared
ast -> shared
```

### 2. 代码组织

**导出策略**
```typescript
// ✅ 推荐：index.ts 入口文件使用 re-export
export * from './token';
export * from './lexer';
export * from './types';

// ✅ 也推荐：明确的具名导出（更好的 tree-shaking）
export { TokenType, Token } from './token';
export { Lexer } from './lexer';
export type { BaseNode, Position } from './types';

// ❌ 避免：在非入口文件中过度使用 export *
// 可能导致命名冲突和不清晰的 API
```

**export * 的优势：**
- **便于维护**：添加新导出时无需修改 index.ts
- **简洁清晰**：入口文件作为聚合点，一目了然
- **标准做法**：大多数库都采用这种模式
- **TypeScript 友好**：类型信息完整保留

**何时使用 export * vs 具名导出：**
```typescript
// 场景1: 库的主入口 - 使用 export *
// packages/lexer/src/index.ts
export * from './lexer';
export * from './token';

// 场景2: 需要重命名或选择性导出 - 使用具名导出
export { Lexer as MarkdownLexer } from './lexer';
export { TokenType } from './token';
// 不导出内部实现细节

// 场景3: 混合使用
export * from './public-api';
export { internalUtilAsPublic } from './internal';
```

**类型定义**
```typescript
// shared/src/types.ts
export interface BaseNode {
  type: string;
  start: number;
  end: number;
}

export interface Position {
  line: number;
  column: number;
  offset: number;
}
```

### 3. 构建优化

**Tsup 配置优化**
```typescript
export default defineConfig({
  // 开发环境
  minify: false,
  sourcemap: true,
  
  // 生产环境可以启用
  // minify: true,
  // sourcemap: false,
  
  // 代码分割（库通常不需要）
  splitting: false,
  
  // 外部依赖（不打包到产物中）
  external: ['@markdown-editor/shared']
});
```

### 4. 版本管理

**语义化版本**
```
0.1.0 -> 初始版本
0.1.1 -> 补丁版本（bug 修复）
0.2.0 -> 次版本（新功能）
1.0.0 -> 主版本（破坏性变更）
```

**推荐工具：Changesets**
```bash
# 安装 changesets
pnpm add -D @changesets/cli

# 初始化
pnpm changeset init

# 添加变更
pnpm changeset

# 发布版本
pnpm changeset version
pnpm changeset publish
```

## 常见问题

### 1. pnpm 相关

**Q: pnpm 版本不匹配怎么办？**
```bash
# 启用 corepack
corepack enable

# 准备指定版本
corepack prepare pnpm@8.15.9 --activate

# 验证版本
pnpm --version
```

**Q: workspace 依赖找不到？**
```json
// 确保 pnpm-workspace.yaml 配置正确
{
  "dependencies": {
    "@markdown-editor/shared": "workspace:*"  // 使用 workspace: 协议
  }
}
```

### 2. TypeScript 相关

**Q: 模块解析失败？**
```json
// tsconfig.json
{
  "compilerOptions": {
    "moduleResolution": "bundler",      // 使用 bundler 解析
    "baseUrl": "./",                   // 设置基础路径
    "paths": {                         // 配置路径映射
      "@markdown-editor/*": ["./packages/*/src"]
    }
  }
}
```

**Q: 类型声明文件生成失败？**
```typescript
// tsup.config.ts
export default defineConfig({
  dts: true,                          // 启用类型声明文件生成
  tsconfig: './tsconfig.json',        // 指定 TS 配置文件
});
```

### 3. 构建相关

**Q: 构建缓存问题？**
```bash
# 清理构建缓存
pnpm run build --force

# 清理 Turbo 缓存
rm -rf .turbo

# 清理所有 dist 目录
rm -rf packages/*/dist
```

**Q: 依赖循环问题？**
```
检查包的依赖关系，确保没有循环依赖：
A -> B -> C -> A (错误)

正确的依赖层次：
shared <- lexer <- parser <- core
       <- ast    <- renderer <-
```

### 4. Git 工作流

**Q: 提交被 hook 阻止？**
```bash
# 检查代码格式
pnpm run lint:fix
pnpm run format

# 检查提交信息格式
git commit -m "feat: 正确的提交信息格式"
```

**Q: 跳过 Git hooks（不推荐）？**
```bash
# 紧急情况下跳过 hooks
git commit --no-verify -m "emergency fix"
```

## 总结

这套工程化配置提供了：

1. **高性能构建**：Tsup + Turbo 构建速度极快
2. **类型安全**：完整的 TypeScript 配置
3. **代码质量**：ESLint + Prettier + Git hooks
4. **开发体验**：VSCode 配置 + 热重载
5. **自动化流程**：GitHub Actions CI/CD
6. **包管理**：pnpm workspace + 依赖管理

通过这套配置，你可以：
- 🚀 快速开发和构建
- 🔒 保证代码质量
- 🤝 团队协作顺畅
- 📦 轻松发布包
- 🔄 自动化测试和部署

这是一个可扩展、可维护的现代前端 Monorepo 架构，适合中大型项目开发。