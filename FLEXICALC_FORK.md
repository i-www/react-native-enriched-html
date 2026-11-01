# FlexiCalc Fork

本仓库是 `react-native-enriched-html` 的独立 fork，用于维护 FlexiCalc 所需的原生自适应宽度测量算法、清除段落结构样式 API，以及定制发布规范。构建后的包发布到 GitHub Packages，业务项目通过 npm alias 消费发布产物，不直接引用本仓库源码。

---

## 上游来源与基线

- **上游官方仓库**：`https://github.com/software-mansion/react-native-enriched-html`
- **当前对齐基线**：`2c2239b360b35eef22a44ae9c4d9354fc4761042`（`upstream/main`）
- **当前发布版本**：`1.2.2`
- **包名**：`@i-www/react-native-enriched-html`
- **发布地址**：`https://npm.pkg.github.com`

---

## 为什么使用独立 Fork

当前偏离涉及原生布局核心（Yoga 测量机制）、Spec 定义与跨端实现（Android、iOS、Web），不适合使用 pnpm patch：
- **自适应宽度测量跨平台**：Android 与 iOS 的测量机制需要深入 TextKit、StaticLayout 和 ShadowNode 内部，完整源码更容易审查、调试与回归
- **Codegen Spec 同步**：定制 API（如 `clearParagraphStyle`）修改了原生组件 Spec，必须确保源码、TypeScript 声明和 Codegen 生成文件在同一上下文中生成
- **避免业务工程污染**：业务仓库无需承载第三方庞大的编译工具链（Bob、Yarn、Codegen）及数百个生成文件

---

## FlexiCalc 独有修改规范

本 Fork 严格恪守最小偏离原则，只保留官方主线尚未支持的 2 项独有能力，其余全量对齐官方实现。

### 1. 原生自适应宽度测量算法 (Auto-sizing Width Measurement)

**设计目标**：短内容采用自然宽度；达到 Yoga 最大约束后自适应换行；最终宽高严格符合父容器布局约束。

| 平台 | 组件 | 涉及文件 | 核心逻辑 |
| :--- | :--- | :--- | :--- |
| **Android** | 输入框 Manager | `android/.../EnrichedTextInputViewManager.kt` | 将 Yoga `widthMode` 传递至测量层，区分精确宽度（EXACTLY）与上限约束（AT_MOST） |
| **Android** | 测量缓存存储 | `android/.../MeasurementStore.kt` | 使用 `Layout.getDesiredWidth` 计算单行自然宽度，根据测量模式确定排版宽度并重新测量高度 |
| **iOS** | 输入框 | `ios/EnrichedTextInputView.mm` | 先以 `CGFLOAT_MAX` 测量自然宽度，再根据 `MIN(maxWidth, naturalWidth)` 限制换行并计算高度 |
| **iOS** | 只读文本 | `ios/EnrichedTextView.mm` | 遍历 `enumerateLineFragmentsForGlyphRange` 统计实际行片段最大宽度，空内容返回零尺寸 |
| **iOS** | 输入框 ShadowNode | `ios/internals/EnrichedTextInputViewShadowNode.mm` | 使用 `layoutConstraints.clamp` 约束原生测量宽高 |

### 2. 清除段落结构样式 API (`clearParagraphStyle`)

**设计目标**：为满足富文本工具栏一键重置标题、引用、代码块与列表样式的交互需求，提供全平台统一的原生与 Web 实现。

| 层面 | 涉及文件 | 职责与实现 |
| :--- | :--- | :--- |
| **Spec** | `src/spec/EnrichedTextInputNativeComponent.ts` | 声明原生命令 `clearParagraphStyle: (viewRef: React.ElementRef<...>) => void` |
| **Type** | `src/types.ts`、`src/native/EnrichedTextInput.tsx` | 对外导出组件 ref 方法 `clearParagraphStyle(): void` |
| **Web** | `src/web/EnrichedTextInput.tsx` | 映射至 Tiptap 编辑器的 `editor.chain().focus().clearNodes().run()` |
| **iOS** | `ios/EnrichedTextInputView.mm` | 遍历移除选中段落的 `H1~H6`、`BlockQuote`、`CodeBlock` 及 `List` 样式并触发重排 |
| **Android** | `android/.../EnrichedTextInputView.kt` | 遍历当前选区涉及段落，安全移除关联的段落与列表 span 并更新布局 |

---

## 提交拓扑规范（纯净 Rebase / 原子 Feature Commits 模型）

为了确保与官方代码库无缝对齐并便于随时向上游提 PR，本 Fork 严格采用**纯净线性历史**，严禁引入任何 merge commit。

每次与官方对齐或升级版本时，必须保持以下 3 个标准的原子 Commit 拓扑：

```text
HEAD -> main (origin/main)
 │
 ├── Commit 3: chore: 升级至 x.y.z 并维护 FlexiCalc Fork 配置规范
 │             (修改 package.json, yarn.lock, FLEXICALC_FORK.md)
 │
 ├── Commit 2: feat(api): 提供 clearParagraphStyle 段落重置接口
 │             (覆盖 Spec, Types, Web, iOS, Android)
 │
 ├── Commit 1: feat(measurement): 实现原生自适应宽度测量算法
 │             (覆盖 iOS TextInput/TextView/ShadowNode, Android TextInput/MeasurementStore)
 │
 └── Upstream Baseline: 官方最新主干或 Release Tag (upstream/main)
```

---

## 业界标准上游对齐与发布操作流程

当需要同步上游官方新版本或主干提交时，遵循以下标准化流程：

### 1. 更新上游镜像并重置工作基线
```bash
git fetch upstream
git checkout -B upstream-main upstream/main
```

### 2. 基于上游基线重新应用定制原子提交
以 `upstream/main` 创建临时工作分支，依次 cherry-pick 或重新应用 3 个纯净原子提交：
```bash
git checkout -b align-upstream upstream/main

# 依次应用特性改动并分别创建原子提交：
# 1. 提交自适应宽度测量算法
git commit -m "feat(measurement): 实现原生自适应宽度测量算法"

# 2. 提交清除段落结构样式 API
git commit -m "feat(api): 提供 clearParagraphStyle 段落重置接口"

# 3. 提交 Fork 工程配置（升级版本号至新版本，如 1.2.x）
git commit -m "chore: 升级至 1.2.x 并维护 FlexiCalc Fork 配置规范"
```

### 3. 全量构建与严格质量校验
在本地必须完成全套构建与测试，确保 100% 通过无任何告警：
```bash
# 生成 Codegen 原生代码与 TS 声明
corepack yarn build

# 运行全套单元测试（必须 14 个套件全部通过）
corepack yarn test

# 运行 TypeScript 类型检查与代码规范检查
corepack yarn typecheck:lib
corepack yarn lint
```

### 4. 远程分支更新与 GitHub Packages 发布
```bash
# 将当前分支切换为 main 并强推至远程建立干净线性历史
git branch -M main
git push -u origin main --force

# 创建对应版本的 Git Tag 并推送
git tag -f v1.2.x
git push origin v1.2.x --force

# 正式发布包到 GitHub Packages（不可变版本策略）
npm publish
```

### 5. 业务主工程（`flexicalc-rn`）集成升级
1. 在 `flexicalc-rn/package.json` 中更新别名版本：
   ```json
   "react-native-enriched-html": "npm:@i-www/react-native-enriched-html@1.2.x"
   ```
2. 安装依赖并更新 Pods：
   ```bash
   pnpm install
   cd ios && pod install
   ```
3. 验证主工程构建：
   ```bash
   xcodebuild -workspace ios/FlexiCalc.xcworkspace -scheme FlexiCalc -configuration Release -destination 'generic/platform=iOS' build
   ```

---

## 退出机制

当上游官方版本满足以下条件时，应废除此 Fork 并直接替换为官方 npm 包：
1. 官方正式合入了段落样式一键重置接口（`clearParagraphStyle` 或同等 API）
2. 官方原生输入框与只读文本完全支持无限画布所需的自适应宽度测量
3. 官方包与当前 React Native 版本完全兼容
