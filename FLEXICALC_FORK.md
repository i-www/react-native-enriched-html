# FlexiCalc Fork

本仓库是 `react-native-enriched-html` 的独立 fork，用于维护上游尚未发布的颜色能力、FlexiCalc 所需的原生自适应宽度测量，以及 React Native `0.85` 的原生元素类型适配。构建后的包发布到 GitHub Packages，业务项目只消费发布产物，不直接引用本仓库目录。

## 上游来源

- 上游仓库：`https://github.com/software-mansion/react-native-enriched-html`
- 当前主线基线：`2c2239b360b35eef22a44ae9c4d9354fc4761042`（upstream/main）
- 最近发布版本：`v1.2.2`
- 包名：`@i-www/react-native-enriched-html`
- 当前版本：`1.2.2`
- 发布地址：`https://npm.pkg.github.com`

PR `#641` 的功能分支仍基于 `1.0.0` 附近的旧主线，不能直接作为长期基线。本 fork 以最新上游 `main` 为基线，只移植 PR 的净功能改动，避免丢失后续修复。

## 为什么使用独立 Fork

当前偏离同时涉及 TypeScript、Android、iOS、C++、原生 spec 和 Codegen，不适合继续使用 pnpm patch：

- 原生测量修改跨越多个平台和组件，完整源码更容易审查与调试。
- PR `#641` 修改原生 spec，必须保证源码、JavaScript、类型声明和 Codegen 产物同步。
- React Native 升级会同时影响原生元素类型和生成代码，patch 容易产生上下文冲突或遗漏生成文件。
- 业务仓库无需提交第三方完整源码及数百个生成文件，也不会在安装时运行第三方 Bob、Yarn 或 Codegen。
- `file:` 或 workspace 引用不适合 EAS 与 CI，因为构建上下文不包含业务仓库外的同级目录。

## FlexiCalc 独有修改

### 自适应宽度测量

统一规则：短内容使用自然宽度；达到 Yoga 最大宽度后使用最大宽度并自动换行；最终宽高必须满足布局约束。

| 平台 | 组件 | 文件与入口 | 修改内容 |
| --- | --- | --- | --- |
| Android | 输入框 | `android/src/main/java/com/swmansion/enriched/textinput/EnrichedTextInputViewManager.kt` 的 `measure` | 将 Yoga `widthMode` 传递给测量层，区分精确宽度与最大可用宽度。 |
| Android | 输入框 | `android/src/main/java/com/swmansion/enriched/textinput/MeasurementStore.kt` 的测量与缓存逻辑 | 使用 `Layout.getDesiredWidth` 计算自然宽度；按 `EXACTLY`、`AT_MOST` 和无约束模式确定最终宽度；按最终宽度重新计算换行高度。 |
| iOS | 输入框 | `ios/EnrichedTextInputView.mm` 的 `measureSize:` | 先测量 attributed string 自然宽度，再限制到最大宽度并重新测量高度。 |
| iOS | 只读文本 | `ios/EnrichedTextView.mm` 的 `measureSize:` | 统计 TextKit 行片段实际宽度，空内容返回零尺寸。 |
| iOS | 输入框 ShadowNode | `ios/internals/EnrichedTextInputViewShadowNode.mm` 的 `measureContent` | 使用 `layoutConstraints.clamp` 同时限制原生测量宽高。 |

Android 只读文本继续使用上游测量逻辑：单行返回实际行宽，多行使用可用宽度。

### 清除段落样式 API (`clearParagraphStyle`)

为满足 FlexiCalc 工具栏快速重置标题、引用、列表等结构样式的需求，提供 `clearParagraphStyle` 统一方法：

- **Spec & Types**: `src/spec/EnrichedTextInputNativeComponent.ts`、`src/types.ts`、`src/native/EnrichedTextInput.tsx`。
- **iOS**: `ios/EnrichedTextInputView.mm` 中移除选中段落的 H1~H6、BlockQuote、CodeBlock 以及 List 样式。
- **Android**: `android/.../EnrichedTextInputView.kt` 中遍历移除当前段落上的 paragraphSpans 与 listSpans。
- **Web**: `src/web/EnrichedTextInput.tsx` 映射至 Tiptap 的 `clearNodes()`。

## 移植的上游能力

官方已在 `v1.2.0`（PR `#817`）中正式合入了全平台 Custom Style 支持（前景色、背景色、字号、字体），覆盖 Android、iOS 与 Web。本 Fork 已完整同步并采用官方最新实现，不再单独维护旧版 PR `#641` 分支。

## 构建产物

以下目录是修改事实来源：

- `src`
- `android/src`
- `ios` 中排除 `ios/generated` 的源码
- `cpp`

以下目录由 Bob 和 React Native Codegen 生成，禁止手工修改，也不提交到 Git：

- `lib`
- `ios/generated`
- `android/generated`

修改 TypeScript、原生 spec 或公开类型后执行：

```bash
corepack yarn install
corepack yarn build
```

`prepack` 会在发布前执行完整构建。发布包必须包含 `lib`、`ios/generated` 和 `android/generated`，业务项目安装发布包时不会执行构建工具链。

## 发布与消费

发布前确认 GitHub Packages 身份和包内容：

```bash
npm whoami --registry=https://npm.pkg.github.com
npm pack --dry-run --registry=https://npm.pkg.github.com
```

确认清单包含 JavaScript、类型声明、原生源码和双端 Codegen 产物后发布：

```bash
npm publish --registry=https://npm.pkg.github.com
```

业务项目通过 npm alias 保持原导入名不变：

```json
{
  "react-native-enriched-html": "npm:@i-www/react-native-enriched-html@1.2.0"
}
```

业务项目的 `.npmrc` 必须为 `@i-www` 配置 GitHub Packages registry 和具有 `read:packages` 权限的 token。EAS 与 CI 应通过 Secret 注入 token，不要提交真实凭据。

## 验证要求

- 运行 `corepack yarn build`，确认源码与生成产物同步。
- 运行 `corepack yarn test`、`corepack yarn lint` 和 `corepack yarn typecheck:lib`。
- 运行 `npm pack --dry-run`，确认发布包包含全部运行时与原生构建文件。
- 在业务项目中确认 lockfile 指向 GitHub Packages，而不是 workspace 或 patch。
- 原生代码、spec 或 Codegen 变化后重新执行 Expo prebuild 并构建 Dev Client。
- 自适应宽度同时验证垂直列表和无限画布中的编辑与只读状态。

## 同步上游规范（Vendor Branching 三分支 Merge 模式）

遵循业界针对已发布长期 Fork 的标准，本地维护 `upstream-main` 镜像官方主干，通过 `git merge` 合入 `main`（严禁使用 Rebase 改写已发布版本的历史 Commit SHA）：

1. **记录基线与拉取**：切换至 `upstream-main` 分支拉取官方最新 commit 或发布 tag（`git pull upstream main`）。
2. **合并与偏离检查**：切回 `main` 分支合并官方镜像（`git merge upstream-main`），检查本地偏离是否已被上游官方实现。
3. **收敛定制改动**：解决冲突，只保留上游仍缺失的独有改动（自适应宽度测量与 `clearParagraphStyle`）。
4. **验证与发版**：提升 package.json 版本号，执行 `yarn build` 与 `yarn test` 校验后发布。
5. **业务依赖更新**：在 `flexicalc-rn` 中更新 package.json 的 alias 版本并重新安装依赖。
6. **全链路回归**：重新构建 Dev Client，回归输入框、只读文本与无限画布的自适应宽度表现。

当官方正式发布版同时满足自适应宽度测量、段落样式重置与当前 React Native 版本要求后，应删除本地 Fork 并恢复官方 npm 依赖。
