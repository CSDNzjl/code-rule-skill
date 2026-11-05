# ifom3-web 高代码开发规约

本规约适用于 `ifom3-web` 中的 Vue 页面、组件、hooks、API、状态、路由和样式，也适用于承载低代码模型的高低结合页面。高代码只实现业务差异，基础能力优先复用项目现有封装。

## 1. 目录与命名

- 页面放在 `src/views/<业务模块>/`，业务 API 放在 `src/api/<业务域>/`，公共组件放在 `src/components/<Component>/`，公共 hooks 放在 `src/hooks/`。
- 新目录优先小写短横线；历史目录如 `workFlow`、`machineEquipment` 不做无关迁移。
- Vue 组件文件使用 PascalCase；页面入口通常是 `index.vue`；hook 使用 `useXxx.ts`。
- API 方法使用 `getXxx`、`createXxx`、`updateXxx`、`deleteXxx`、`exportXxx`、`publishXxx` 等可检索命名。
- 布尔变量使用 `is`、`has`、`can`、`should` 开头；优先 named exports 和 `interface`。

## 2. Vue 与 TypeScript

- 新代码使用 `<script setup lang="ts">`，区块顺序为 `template`、`script`、`style`；不新增 Options API。
- 优先使用 `interface` 描述 props、emits、请求参数、响应、表格行和公开方法；避免 `any`，外部动态数据使用 `unknown` 并在边界收窄。
- 纯函数使用 `function` 声明。能用 `computed` 表达的派生值不使用 watch 回写；watch 必须处理重复触发、旧请求和清理。
- 模板不写请求、复杂转换和副作用；`v-for` 使用稳定唯一 key，不用数组索引作为可变列表 key。
- 卸载时清理定时器、事件监听、WebSocket、MutationObserver 和全局回调；异步回写前确认组件仍有效。

## 3. 平台组件

- 优先复用 `BasicForm`、`BasicTable`、`BasicModal`、`BasicPopup`、`BasicTree`、`useForm`、`useTable` 和项目已有 hooks。
- 标准列表使用 `BasicForm` + `useForm`、`BasicTable` + `useTable`；分页、选择、刷新、滚动和高度重算使用 hook 暴露的方法。
- 表单使用 Schema；异步选项通过 `updateSchema` 更新。操作列优先使用 `TableAction`/`ActionItem`。
- 常规弹窗使用 `BasicModal` + `useModalInner`，抽屉/弹出页面使用 `BasicPopup` + `usePopupInner`。
- 自定义单元格优先具名 slot；只有需要动态 VNode 时才用 render 回调，复杂单元格拆为局部组件。
- 样式优先使用已有类、Windi/Tailwind 和 Less 变量；避免全局选择器、深层穿透和覆盖第三方内部 DOM。

## 4. API、状态与异步

- `src/api` 只负责 URL 常量、HTTP 方法、参数和返回类型；页面不直接调用 axios，不重复定义拦截器或基础 URL。
- 统一使用 `defHttp`，写法参考 `src/utils/http/axios` 和同域 API 文件；不要假设返回值是 Axios 原始响应。
- URL 前缀集中在 API 文件的 `Api` 常量中；路径参数、查询参数、请求体按后端契约传递。
- 调用低代码数据接口统一走 `POST /api/system/DataInterface/invokeByEnCode`：请求体 `id` 传接口编码（enCode）、`map` 传业务参数。示例见 `src/api/craft/processNodeAnalysis.ts` 与 `src/api/craft/product.js` 的 `dataInterfaceApi`。
- loading、空数据、错误、旧请求覆盖和重复提交由页面或业务 hook 负责，并在 `finally` 恢复状态。
- 破坏性操作使用 `useMessage` 确认；成功后清理选择项并刷新列表。
- 单页状态留在页面或 hook，跨页面共享状态才进入 Pinia。
- 雪花 ID 等大整数按字符串处理；文件上传使用 `FormData`，不要手工设置 multipart boundary。

## 5. 表单、表格和交互

- 提交顺序为：校验 → loading/禁用重复提交 → API → 成功提示与关闭 → 通知父列表刷新 → finally 恢复。
- 弹窗打开时初始化，关闭或切换记录时清理旧表单值、校验、选择项和异步状态。
- 破坏性操作处理无选择、部分失败、空结果、分页删除最后一条和接口失败。
- 表格必须设置稳定 `rowKey`。行点击选中使用 `rowSelection` 和 `setProps`，不修改第三方内部状态。
- 动态列基于不可变的完整 columns 派生；状态颜色保持统一语义，不能作为唯一状态表达。
- 路由跳转使用 Vue Router；优先使用 query/params 或现有 store，避免手工拼接不透明参数。
- **启停开关布局**：`是否启用` 等单独 Switch 不要独占表单中部一行造成大片留白；有备注时放在备注下方，或与其它短字段（如「是否默认」）同一行并排。

## 6. 高低代码结合

使用 `CustomModel` 时必须同时阅读 `low-code-conventions.md`。高代码负责页面布局、复杂交互、弹窗和特殊 API，低代码负责模型字段、标准表单/列表和平台配置。

```vue
<CustomModel
  ref="modelRef"
  modelId="planProductionOrder"
  :showSearch="true"
  :showOperation="true"
  :customSearchSchemas="searchSchemas"
  :customTableConfig="tableConfig"
  :initData="initData"
/> 
```

常用实例方法：

- `getFormRef()`：获取当前低代码表单引用；表单未渲染时可能为空。
- `getFormData()` / `setFormData(field, value)`：读取或设置表单数据。
- `getTableRef()`：获取低代码列表引用，可调用 `reload`、`setProps`、`getSelectRows` 等已有表格方法。
- `setFieldsValue(data)`：设置低代码列表搜索表单的值。
- `getLeftTreeRef()`：获取左侧树引用，仅树表格模式可用。
- `comSet(field, prop, value, isButton)`：改变模型组件配置；优先使用语义明确的现有方法。

### 6.1 `customClick` 生命周期

低代码按钮可以调用 `customClick({ funcName, data, callback })`，高代码页面接收后执行复杂逻辑：

```ts
const currentHandler = window.customClick;
const handler = (opts: CustomClickOptions) => {
  if (opts.funcName !== 'processInformation') return;
  // 校验 opts.data 后执行弹窗、请求或刷新
};

onMounted(() => {
  window.customClick = handler;
});

onUnmounted(() => {
  if (window.customClick === handler) window.customClick = currentHandler;
});
```

- 为 `CustomClickOptions` 定义最小类型，至少区分 `funcName`、`data` 和可选 `callback`。
- 只响应允许的方法名，校验数据结构，不执行传入字符串代码。
- 新功能优先使用 props/emits、注入的 emitter 或已有弹窗协议；不要扩大 `window` 全局 API。
- 高代码关闭弹窗或刷新父列表时使用 typed emits 或已有事件，例如 `reloadParentList`，不要直接操作父级 DOM。

## 7. 权限、安全与响应式

- 使用项目已有权限指令、`usePermission` 和菜单配置；前端隐藏不能替代后端鉴权。
- 高低代码菜单需配置正确的页面地址、模型关联和权限，否则页面可打开但按钮/字段权限可能失效。
- token 由请求封装管理，不写死、不输出日志；低代码脚本通过运行时工具获取用户信息。
- 用户输入、富文本和 Markdown 使用项目已有清洗方案，禁止直接拼接不可信 HTML。
- 页面必须考虑窄屏、滚动、亮色/暗色主题、加载、空数据和错误状态。

## 8. 验证清单

在前端工程目录、已 `pnpm bootstrap` 安装依赖后执行。验证以本次改动范围为准，全仓库扫描不作为卡点：

1. 类型检查：`npx vue-tsc --noEmit --skipLibCheck`，输出过滤到改动文件所在目录，改动文件零错误即通过；存量模块既有错误不属于本次交付范围。
2. 行为改动补同目录 `*.spec.ts`，用 `npx vitest run <spec>` 只跑相关用例。
3. 只对本次改动文件执行 `npx eslint <文件>`、`npx stylelint <文件>`（命中 ignore 清单的文件跳过）；提交时 lint-staged 会自动补充格式化。
4. 仅当涉及路由、动态导入、构建配置或模块联邦时执行 `pnpm build`，必要时执行 `pnpm build:federation`。
5. 手工验证权限账号、弹窗打开/关闭、父列表刷新、重复点击、接口失败、真实低代码模型和窄屏布局。
