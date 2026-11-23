---
kind: frontend_style
name: MeterSphere 前端样式体系：Arco Design + Less + Tailwind + 自定义主题
category: frontend_style
scope:
    - '**'
source_files:
    - frontend/src/main.ts
    - frontend/src/assets/style/global.less
    - frontend/src/assets/style/var.less
    - frontend/src/assets/style/arco-reset.less
    - frontend/src/assets/style/breakpoint.less
    - frontend/src/theme/default.less
    - frontend/src/theme/green.less
    - frontend/tailwind.config.mjs
    - frontend/config/plugin/arcoStyleImport.ts
    - frontend/src/config/chartTheme.ts
    - frontend/.stylelintrc.cjs
    - frontend/package.json
---

## 1. 采用的样式系统

- **UI 组件库**：`@arco-design/web-vue`（v2.56+），通过 `app.use(ArcoVue)`、`app.use(ArcoVueIcon)` 在 `src/main.ts` 全局注册。
- **主题包**：`@arco-themes/vue-metersphere-v3`，在 `main.ts` 中直接 `import '@arco-themes/vue-metersphere-v3/index.less'` 引入，作为默认主题；同时通过 Vite 插件 `@arco-plugins/vite-vue` 的 `vitePluginForArco({ theme: '@arco-themes/vue-metersphere-v3' })` 实现主题样式的按需加载（见 `config/plugin/arcoStyleImport.ts`）。
- **CSS 预处理器**：Less（`less` + `less-loader`），所有业务样式以 `.less` 编写，并通过 PostCSS (`postcss-less`) 解析。
- **原子化 CSS**：Tailwind CSS v3（`tailwind.config.mjs` 仅声明 content 路径并 `extend: {}`，未扩展任何 token），在 `global.less` 中以 `@tailwind base/components/utilities` 注入，并在组件内大量使用 `@apply` 组合原子类。
- **富文本 Markdown 样式**：额外引入 `github-markdown-css/github-markdown-light.css`。
- **图表主题**：ECharts 主题集中在 `src/config/chartTheme.ts`，由 `useThemeVars` hook 驱动。

## 2. 关键文件与位置

| 作用 | 文件 |
|---|---|
| 应用入口（注册 Arco、主题、全局样式） | `frontend/src/main.ts` |
| 全局样式入口（Tailwind 注入、基础 reset、通用类） | `frontend/src/assets/style/global.less` |
| 设计变量 & 混合函数（圆角、按钮状态 mixin、滚动条、阴影容器、输入框前后缀、表格尺寸） | `frontend/src/assets/style/var.less` |
| Arco reset 覆盖 | `frontend/src/assets/style/arco-reset.less` |
| 断点变量 | `frontend/src/assets/style/breakpoint.less` |
| 主题覆盖（默认/绿色两套） | `frontend/src/theme/default.less`, `frontend/src/theme/green.less` |
| Tailwind 配置 | `frontend/tailwind.config.mjs` |
| Arco 主题按需加载插件 | `frontend/config/plugin/arcoStyleImport.ts` |
| ECharts 主题 | `frontend/src/config/chartTheme.ts` |
| Stylelint 规则 | `frontend/.stylelintrc.cjs` |
| 构建配置（Vite base/dev/prod） | `frontend/config/vite.config.base.ts`, `vite.config.dev.ts`, `vite.config.prod.ts` |

## 3. 架构与设计约定

### 3.1 主题与 Token 体系
- 主色调通过 CSS 自定义属性 `--primary-6`、`--primary-7` 等暴露给组件层。`default.less` 将默认紫色的两个色阶覆盖为 MeterSphere 品牌紫（`#811fa3` / `#6e1a8b`）；`green.less` 提供 `.theme-green` 类，把这两个变量替换为绿色（`#aeff18` / `#8ae600`），供运行时切换。
- 颜色值统一通过 `rgb(var(--primary-X))` / `var(--color-text-*)` / `var(--danger-*)` / `var(--warning-*)` 引用 Arco 语义变量，避免硬编码十六进制。
- 圆角等布局 token 在 `var.less` 中以 Less 变量定义（`@border-radius-mini/small/medium/large`），并被多处复用。

### 3.2 样式组织方式
- 全局基础样式集中在 `assets/style/`：`global.less` 负责 Tailwind 注入、`html/body` 基础排版、通用工具类（`.general-card`、`.split-line`、`.sub-text`、`.one-line-text`、`.ms-tooltip-white/red`、`.ms-form-table-required-button`、`.selected-row-class`、无权限 SVG 明亮/暗黑变体等）。
- 可复用的 Less Mixin 集中在 `var.less`：
  - 按钮状态 mixin：`.btn-outline-primary-*`、`.btn-base-primary-*`、`.btn-text-primary-*` 以及 danger/warning/secondary 系列，统一 hover/active/disabled 三态。
  - 滚动条：`.ms-scroll-bar()` 封装统一的 6px 半透明滚动条，hover 时显示 thumb。
  - 容器阴影：`.ms-container--shadow-y()` 通过伪元素实现顶部/底部 inset 阴影。
  - 输入框组合：`.ms-input-group--append()` / `--prepend()` 处理 Arco Input/Selct 后缀/前缀拼接时的边框与 z-index。
  - 表格：`.ms-table--special-small()` 压缩单元格 padding。
- 业务组件样式遵循按组件目录存放（如 `components/business/ms-comment/style.less`），由 `global.less` 动态 import。

### 3.3 响应式策略
- 断点变量定义在 `breakpoint.less`，组件内通过 Tailwind 的 `@screen` / `responsive` at-rule 与 `@apply` 组合使用（Stylelint 已允许这些 at-rule）。
- 没有独立的移动端适配方案，主要依赖 Tailwind 原子类与 Arco 组件自身的响应式行为。

### 3.4 图标与资源
- 自研图标通过 `SvgIcon` 组件 + `virtual:svg-icons-register` 从 `assets/svg/icons/` 与 `assets/svg/*.svg` 自动注册。
- 字体图标通过 `assets/icon-font/` 下的 iconfont 多格式文件（woff2/woff/ttf/svg/js/css）引入。
- 字体族在 `global.less` 中指定为 `'Helvetica Neue', Arial, 'PingFang SC', 'Source Han Serif'`。

## 4. 规范与约束

- **代码风格强制**：
  - Stylelint 通过 `extends` 继承 `stylelint-config-standard`、`stylelint-config-prettier`、`stylelint-config-html/vue`、`stylelint-config-recommended-less`，并使用 `stylelint-less`、`stylelint-order` 插件。
  - 缩进固定为 2 空格（`indentation: 2`）。
  - 字符串引号强制单引号（`string-quotes: 'single'`）。
  - 十六进制颜色强制小写且长格式（`color-hex-case: 'lower'`、`color-hex-length: 'long'`）。
  - 属性顺序通过 `order/properties-order` 严格排序（position → display → margin/padding → width/height → font → border → color → background → opacity → filter → list-style → outline → visibility → box-shadow → text-shadow → resize → transition）。
  - Vue SFC 内联样式使用 `postcss-html` 语法解析。
  - `:deep`、`v-deep`、`:global` 伪类被显式忽略校验。
  - Tailwind 相关 at-rule（`apply`、`tailwind`、`variants`、`responsive`、`screen`）被忽略未知 at-rule 检查。
- **提交钩子强制**：`package.json` 的 `lint-staged` 对 `*.vue` 执行 `stylelint --fix`，对 `*.{less,css,scss,sass}` 使用 `postcss-less` 语法执行 `stylelint --fix`，再经 Prettier 格式化。
- **构建期按需样式**：`@arco-plugins/vite-vue` 的 `vitePluginForArco` 根据使用的组件按需拉取对应样式，减少打包体积。
- **浏览器兼容**：`browserslist` 排除 IE 11，目标为 `>0.5%` 和最近两个版本；Safari 富文本兼容通过 `utils/polyfill.ts` 中的 `Object.hasOwn` polyfill 单独处理。
- **Tailwind 未扩展 token**：`tailwind.config.mjs` 的 `theme.extend` 为空，项目不新增 Tailwind 自定义色板或间距，而是依赖 Arco 语义变量 + Less mixin 的组合。

## 5. 总结

MeterSphere 前端样式体系以 **Arco Design Vue 组件库 + 自定义 Arco 主题包** 为基础，用 **Less** 承载业务样式与可复用 mixin，用 **Tailwind CSS** 提供原子化布局能力，并通过 **CSS 自定义属性** 暴露主题 token（主色、危险色、警告色等）实现亮/暗与品牌色切换。样式质量由 **Stylelint + Prettier + lint-staged** 在提交阶段强制约束，构建期通过 Arco Vite 插件按需加载主题样式。