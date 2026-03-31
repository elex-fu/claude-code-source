# Claude Code 逆向工程可行性分析报告

## 1. 源代码恢复状况

### 已恢复内容
- **TypeScript/TSX 源码**: 1,888 个文件
- **node_modules 包**: 197 个包
- **Source map**: 完整（包含 sourcesContent）

### 关键缺失
- `package.json` - 项目配置和依赖列表
- `tsconfig.json` - TypeScript 编译配置
- `bun.lock` - 依赖版本锁定
- 构建脚本和配置文件

---

## 2. 逆向工程 package.json 可行性

### 2.1 可直接获取的信息

#### 已安装的包（197个）
```bash
# 从 restored-src/node_modules 提取
ls restored-src/node_modules/ | sort
```

包含核心包：
- **React 生态**: `react`, `react-reconciler`
- **CLI 工具**: `commander`, `chalk`, `ink` (内部)
- **类型校验**: `zod`, `zod-to-json-schema`
- **HTTP 客户端**: `axios`, `node-fetch`, `undici`
- **AWS/GCP**: `@aws-sdk/*`, `google-auth-library`
- **MCP**: `@modelcontextprotocol/sdk`, `@ant/computer-use-mcp`
- **OpenTelemetry**: `@opentelemetry/api`, `@opentelemetry/core`
- **图片处理**: `sharp` (平台特定的可选依赖)

#### 从 package/cli.js 提取
```json
{
  "name": "@anthropic-ai/claude-code",
  "version": "2.1.88",
  "bin": { "claude": "cli.js" },
  "engines": { "node": ">=18.0.0" },
  "type": "module"
}
```

#### 从源代码 import 分析提取
```javascript
// 核心依赖（高频使用）
"react": "^18.x"
"react-reconciler": "^0.29.x"
"zod": "^3.x"
"@anthropic-ai/sdk": "^0.x"
"axios": "^1.x"
"chalk": "^5.x"
"commander": "^11.x"
"diff": "^5.x"
"execa": "^8.x"
"figures": "^6.x"
"lodash-es": "^4.x"
"marked": "^9.x"
"strip-ansi": "^7.x"
"usehooks-ts": "^3.x"
"ignore": "^5.x"
"chokidar": "^3.x"
"lru-cache": "^10.x"
"qrcode": "^1.x"
"semver": "^7.x"
"ws": "^8.x"
"xss": "^1.x"
```

### 2.2 版本推断策略

#### 可行方法：
1. **从 node_modules 读取**
   ```bash
   cat restored-src/node_modules/zod/package.json | jq '.version'
   ```

2. **从源代码 import 路径推断**
   - `import { something } from 'zod/v4'` → zod v4+
   - `import React from 'react'` → React 18+ (使用 compiler-runtime)

3. **依赖兼容性矩阵**
   - 基于已知的 Anthropic SDK 版本
   - 基于 Bun 构建工具的发布时间

### 2.3 可行性评估

| 方面 | 可行性 | 难度 | 备注 |
|------|--------|------|------|
| 依赖列表 | ⭐⭐⭐⭐⭐ | 低 | 从 import + node_modules 完全可恢复 |
| 版本号 | ⭐⭐⭐⭐ | 中 | 可从 node_modules 读取，或推断 |
| scripts | ⭐⭐ | 高 | 未知构建流程 |
| 元数据 | ⭐⭐⭐⭐⭐ | 低 | 从 package/cli.js 可提取 |

**结论**: package.json 可 80% 准确重建

---

## 3. 逆向工程 tsconfig.json 可行性

### 3.1 可推断的配置

#### 从源代码分析：
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "jsxImportSource": "react",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "allowSyntheticDefaultImports": true,
    "paths": {
      "ink": ["./src/ink.ts"],
      "src/*": ["./src/*"]
    }
  }
}
```

#### 推断依据：
1. **target**: 使用 `node:fs/promises` → Node 18+ → ES2022
2. **module**: Bun 构建 → ESNext
3. **jsx**: 使用 React 组件 → react-jsx
4. **paths**: 从 import 路径 `src/...` 推断

### 3.2 Bun 特有配置

```typescript
// 源代码中的 Bun 特定导入
import { feature } from 'bun:bundle'  // 出现 196 次
```

需要 Bun 运行时环境，无法通过标准 tsc 编译。

### 3.3 可行性评估

| 方面 | 可行性 | 难度 | 备注 |
|------|--------|------|------|
| 基础配置 | ⭐⭐⭐⭐⭐ | 低 | 从代码模式可完全推断 |
| paths 映射 | ⭐⭐⭐⭐ | 中 | 需要分析所有 import |
| Bun 配置 | ⭐⭐ | 高 | 需要 Bun 构建系统 |
| 宏配置 | ⭐ | 极高 | MACRO.VERSION 等需要构建时替换 |

**结论**: tsconfig.json 基础配置可 90% 准确重建，但 Bun 特定功能无法复现

---

## 4. 关键障碍

### 4.1 Bun:bundle 特性

源代码中 196 处使用 `bun:bundle`：
```typescript
import { feature } from 'bun:bundle'

if (feature('SOME_FEATURE')) {
  // 编译时条件代码
}
```

**问题**：
- `feature()` 是 Bun 构建时的死代码消除(DCE)机制
- 在运行时不可用
- 需要 Bun 的构建流程才能正确处理

### 4.2 构建宏 (Macros)

```typescript
// cli.tsx 第40行
console.log(`${MACRO.VERSION} (Claude Code)`);
```

**问题**：
- `MACRO.VERSION` 需要构建时注入
- 无法通过逆向工程获取原始值

### 4.3 可选依赖的平台特定性

```json
{
  "optionalDependencies": {
    "@img/sharp-darwin-arm64": "^0.34.2",
    "@img/sharp-darwin-x64": "^0.34.2",
    ...
  }
}
```

这些已经包含在恢复的 node_modules 中。

### 4.4 内部包

从源代码发现内部包：
- `@ant/computer-use-mcp` - 已恢复
- `@ant/claude-for-chrome-mcp` - 已恢复
- `ink` - 实际路径是 `./src/ink.ts`

---

## 5. 逆向工程实现步骤

### 5.1 提取 package.json

```bash
#!/bin/bash
# 1. 提取包版本
for pkg in $(ls restored-src/node_modules/); do
  if [ -f "restored-src/node_modules/$pkg/package.json" ]; then
    version=$(cat "restored-src/node_modules/$pkg/package.json" | jq -r '.version // "unknown"')
    echo "\"$pkg\": \"$version\""
  fi
done
```

### 5.2 分析 import 依赖

```bash
# 提取所有外部 import
grep -rh "^import\|^from" restored-src/src --include="*.ts" --include="*.tsx" \
  | grep -oE "from\s+['\"][@a-zA-Z0-9_-]+(/[a-zA-Z0-9_-]+)*['\"]" \
  | grep -oE "[@a-zA-Z0-9_-]+(/[a-zA-Z0-9_-]+)*" \
  | sort | uniq
```

### 5.3 创建 tsconfig.json

基于代码分析创建配置（见上文）。

---

## 6. 打包可行性结论

### 直接编译：❌ 不可行

原因：
1. 依赖 Bun 构建系统特有的功能
2. 需要处理 `bun:bundle` 的 feature flags
3. 需要替换 MACRO.* 宏

### 使用 Bun 构建：⚠️ 部分可行

如果能获取到：
1. Bun 的构建配置（bun.build.ts 或 build.ts）
2. 宏定义文件
3. feature flags 配置

则可能可以重建。

### 替代方案：✅ 可行

1. **直接使用已编译版本**
   - `package/cli.js` 是完整可运行的
   - 配合 `package/package.json` 可以 `npm install -g`

2. **源码阅读/修改**
   - 恢复的源代码可用于理解、分析
   - 可用于 IDE 的智能提示
   - 可用于安全审计

---

## 7. 重建 package.json 示例

```json
{
  "name": "@anthropic-ai/claude-code",
  "version": "2.1.88",
  "description": "Claude Code CLI",
  "type": "module",
  "bin": {
    "claude": "./dist/cli.js"
  },
  "engines": {
    "node": ">=18.0.0"
  },
  "scripts": {
    "build": "bun build ./src/entrypoints/cli.tsx --outfile=dist/cli.js",
    "dev": "bun run ./src/entrypoints/cli.tsx"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.x",
    "@aws-sdk/client-sso": "^3.x",
    "@aws-sdk/nested-clients": "^3.x",
    "@azure/msal-common": "^14.x",
    "@azure/msal-node": "^2.x",
    "@commander-js/extra-typings": "^11.x",
    "@growthbook/growthbook": "^1.x",
    "@img/sharp-darwin-arm64": "^0.34.2",
    "@inquirer/core": "^9.x",
    "@modelcontextprotocol/sdk": "^1.x",
    "@opentelemetry/api": "^1.x",
    "@typespec/compiler": "^0.x",
    "ajv": "^8.x",
    "axios": "^1.x",
    "chalk": "^5.x",
    "chokidar": "^3.x",
    "commander": "^11.x",
    "diff": "^5.x",
    "execa": "^8.x",
    "figures": "^6.x",
    "fuse.js": "^7.x",
    "google-auth-library": "^9.x",
    "ignore": "^5.x",
    "ink": "file:./src/ink.ts",
    "lodash-es": "^4.x",
    "lru-cache": "^10.x",
    "marked": "^9.x",
    "open": "^10.x",
    "qrcode": "^1.x",
    "react": "^18.x",
    "react-reconciler": "^0.29.x",
    "semver": "^7.x",
    "sharp": "^0.34.x",
    "strip-ansi": "^7.x",
    "tree-kill": "^1.x",
    "undici": "^6.x",
    "usehooks-ts": "^3.x",
    "ws": "^8.x",
    "xss": "^1.x",
    "yaml": "^2.x",
    "zod": "^3.x",
    "zod-to-json-schema": "^3.x"
  },
  "devDependencies": {
    "@types/react": "^18.x",
    "@types/react-reconciler": "^0.x",
    "bun-types": "latest"
  }
}
```

---

## 8. 最终评估

| 任务 | 可行性 | 工作量 | 输出质量 |
|------|--------|--------|----------|
| 重建 package.json | 85% | 4-6小时 | 良好 |
| 重建 tsconfig.json | 90% | 2-3小时 | 良好 |
| 重建 bun 构建配置 | 30% | 2-3天 | 不确定 |
| 完整可编译项目 | 40% | 1-2周 | 未知 |
| 可用 CLI 包 | 95% | 已存在 | 完美 |

**建议**：如需使用 CLI，直接使用 `package/` 目录；如需研究源码，恢复的源代码已足够。
