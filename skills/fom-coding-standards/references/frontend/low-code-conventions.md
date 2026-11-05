# ifom3-web 低代码开发规约

本规约适用于在线开发、动态表单、列表设计、表单脚本、流程表单、虚拟列表、页面布局、报表和可视化配置。低代码的字段编码、组件类型和脚本参数是设计器、运行时、后端之间的契约，修改时优先保证契约稳定。

## 1. 适用边界

- 普通表单、列表、查询、详情、审批、权限、布局和报表优先使用低代码设计器。
- 只有低代码无法表达的复杂交互、特殊布局、复杂拖拽或性能要求才引入高代码扩展。
- 仅增加字段、调整布局、增加查询条件或绑定已有接口，不新建 Vue 页面。
- 设计器配置、发布/回滚、运行时解析和 API 必须作为一条链路验证，不能只验证设计器预览。

## 2. 模型与字段契约

- 字段编码以模型配置和后端契约为准，不根据显示名称猜字段名；改名或删除前检索新建、编辑、详情、审批、导入导出和脚本事件。
- 控件类型必须来自当前运行时组件映射，不写运行时无法识别的组件名或 `jnpfKey`。
- 必填、多选、默认值、显示条件、禁用条件和数据字典值在设计器、运行时和提交前保持一致。
- 区分展示值、编码值和 ID。关联表单、弹窗选择和远端接口不回显时，依次检查数据源类型、回显配置和返回值是否一致。
- 后端大整数 ID 按字符串传递；字段默认值必须符合组件实际类型和接口约定。

## 3. 动态表单脚本

脚本运行在低代码运行时，不等同于 `ifom3-web` 的 TypeScript 页面代码。只使用运行时注入的参数和工具。

### 3.1 常用参数

- `data`：当前控件或当前行选中的数据。
- `rowIndex`：子表/虚拟列表的当前行下标。
- `formData`：当前表单数据；子表通常通过 `formData[field][rowIndex][childField]` 访问。
- `formInfo`：弹窗表单上下文，可能包含 `customChooseData.selectRow` 等父列表选择数据，使用前判空。
- `setFormData(field, value)`：设置普通字段或子表数据。
- `setShowOrHide(field, value)`、`setRequired(field, value)`、`setDisabled(field, value)`：控制运行时字段状态。
- `setFormLoading(value)`、`setListloading(value)`：控制加载状态，异步流程必须在失败和异常路径恢复。
- `refresh(value?)`、`closeForm()`、`closeCustomListPopup()`：按当前脚本上下文刷新或关闭。
- `onlineUtils.getUserInfo()`、`getDeviceInfo()`、`request()`、`route()`、`toast()`：使用平台工具，不自行读取 token 或拼接认证头。

### 3.2 脚本要求

- 异步请求使用完整的成功、失败和 finally 路径，检查响应状态及 `data` 后再回填。
- 字段联动时同时处理旧值、校验规则和提交行为；隐藏字段不能替代权限或后端校验。
- 空结果要明确清空或保留原值，不能因请求为空而残留过期数据。
- 时间计算明确单位、时区、空值、结束早于开始和精度，避免依赖浏览器的字符串解析。
- 上传 Excel 使用 `FormData` 调用 `/api/file/excelParser`；下载使用 `/api/file/Download/{type}/{fileName}` 对应的受保护流程。
- 不输出 token、密码、完整用户隐私或大对象日志；发布前删除调试日志。
- 禁止 `eval`、动态脚本注入、不可信 HTML 和未经审核的第三方接口。

## 4. 子表与虚拟列表

- 批量回填前确认模型要求的是平铺字段、嵌套字段还是子表字段。
- 每一行必须生成稳定且唯一的 `jnpfId`，不能使用可能重复的随机值或数组索引作为长期业务 key。
- 主表 change/blur 事件给子表赋值时，使用模型字段编码；单行回填使用 `rowIndex` 定位，不修改错误行。
- 选择产线、工序等外部条件后回填虚拟列表，先清理旧列表，再写入新列表并保证字段映射完整。
- 子表数据接口失败时恢复 loading，并保留用户可理解的错误提示。

## 5. 列表脚本渲染

- 自定义单元格渲染参数通常为 `text`、`record`、`index`、`column`、`onlineUtils`、`refresh`。
- 只在目标列返回 `tag` 或 `switch` 配置，其他列返回 `null`，保留默认单元格渲染。
- `switch` 状态变更必须调用后端接口，成功后刷新，失败时提示并避免界面状态与服务端不一致。
- 动态隐藏列时基于完整原始 columns 重新派生，不在已经过滤的结果上重复过滤。
- 行高亮优先使用表格配置的 `customRow`、`rowClassName` 和稳定 `rowKey`；不要把 DOM 查询和 `MutationObserver` 作为普通业务方案。

## 6. 高低代码结合契约

高代码通过 `CustomModel` 加载低代码模型：

```vue
<CustomModel
  ref="modelRef"
  modelId="materialMaster"
  :customSearchSchemas="searchSchemas"
  :customTableConfig="tableConfig"
  :initData="initData"
/> 
```

- `modelId` 可使用模型编码或 ID，优先使用稳定编码。
- `customSearchSchemas` 必须是 `FormSchema[]`；低代码列表必须存在对应查询字段，否则过滤不会生效。
- `customTableConfig` 合并 `useTable` 配置，如 `rowKey`、分页、行点击和列配置。
- `initData` 只传表单默认提交数据，不混入仅用于展示的查询参数。
- `isForm`、`showSearch`、`showOperation`、`canDragSort`、`defaultCheckFirst`、`hideBtns` 的默认值以当前源码为准。

低代码按钮与高代码交互时使用历史协议 `customClick({ funcName, data, callback })`：

- `funcName` 保持业务唯一且稳定，`data` 只传最小必要数据。
- 高代码接收后先判断方法名并校验数据，再执行允许的动作。
- 回调只用于明确的成功/失败通知，不传任意可执行代码。
- 高代码注册回调、低代码触发按钮、父级刷新和弹窗关闭必须形成闭环，详见 `high-code-conventions.md`。

## 7. 发布前检查

1. 检查模型字段、控件映射、默认值、校验、权限和接口参数。
2. 验证新建、编辑、详情、查询、删除、发布/回滚和异常场景。
3. 涉及流程时验证发起、办理、退回、待办跳转和弹窗回填。
4. 验证真实接口、空列表、空字段、重复点击、分页删除最后一条和旧配置。
5. 验证菜单关联：菜单类型为高低代码，地址为高代码路径，关联正确的低代码模型。
