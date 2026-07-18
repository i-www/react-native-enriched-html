# FlexiCalc Fork

本仓库是 `react-native-enriched-html` 的独立 fork，用于维护上游尚未发布的颜色能力、FlexiCalc 所需的原生自适应宽度测量，以及 React Native `0.85` 的原生元素类型适配。构建后的包发布到 GitHub Packages，业务项目只消费发布产物，不直接引用本仓库目录。

## 上游来源

- 上游仓库：`https://github.com/software-mansion/react-native-enriched-html`
- 当前主线基线：`df8c17bc7da5256286a88adf38d88733cf913ab8`
- 最近发布版本：`v1.0.1`
- iOS 颜色能力：上游 PR `#641`
- PR head：`4ab75310b7d6201ab8389f9e634f44dd6d530a50`
- 包名：`@i-www/react-native-enriched-html`
- 当前版本：`1.0.1-flexicalc.2`
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

### React Native 0.85 类型适配

React Native `0.85` 使用 `ReactNativeElement` 表达具体宿主实例。以下内部 ref 与事件目标类型从 `NativeMethods` 调整为 `ReactNativeElement`：

- `src/native/EnrichedTextInput.tsx`
- `src/native/EnrichedText.tsx`
- `src/web/adaptWebToNativeEvent.ts`

公共实例接口仍只继承 `NativeMethods`，因为公开 API 只承诺测量、聚焦等调用能力。

React Native `0.85` 同时将 Jest preset 拆分到 `@react-native/jest-preset`。本 fork 的测试配置直接使用该包，避免继续加载已经只负责报错迁移提示的 `react-native/jest-preset.js`。

`@tiptap/react@3.27.1` 将 Bubble Menu 和 Floating Menu 声明为 `^3.27.1` 的 optional dependency，可能在消费者项目中漂移到不兼容的 `3.28.x`。本 fork 将这两个扩展直接固定为 `3.27.1`，并显式提供 Floating Menu 所需的 `@floating-ui/dom@1.8.0`，确保依赖闭合且整套 Tiptap 保持同一版本。

## 移植的上游能力

PR `#641` 提供 iOS 自定义文字色与背景色支持：

- `setStyle` 原生命令和 TypeScript API。
- 当前选区的 `customStyle` 状态上报。
- iOS attributed string 颜色应用、HTML 解析和 HTML 序列化。
- `<span style="color: ...; background-color: ...">` 的标准化处理。

Android 的 `setStyle` 在该 PR 中仍为空实现。FlexiCalc 当前只在 iOS 展示颜色入口，Android 与 Web 的颜色编辑和跨端保真不属于当前 fork 的承诺范围。

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
  "react-native-enriched-html": "npm:@i-www/react-native-enriched-html@1.0.1-flexicalc.2"
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

## 同步上游

1. 记录新的上游 commit 或发布 tag。
2. 检查本文件记录的本地偏离是否已被上游实现。
3. 只保留上游仍缺失的改动。
4. 提升 `-flexicalc.N` 版本并重新构建、检查和发布。
5. 在业务项目中更新 alias 版本并重新安装依赖。
6. 重新构建 Dev Client，回归颜色、输入框、只读文本和自适应宽度。

当官方发布版同时满足颜色、自适应宽度和当前 React Native 类型要求后，应删除本地 fork 并恢复官方 npm 依赖。
