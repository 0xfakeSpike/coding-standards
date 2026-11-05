# 前端表单

本文规定表单的校验、状态、错误展示和提交。表单元素的语义见 [html.md](html.md)，无障碍要求见 [accessibility.md](accessibility.md)，通用的受控与非受控组件见 [components.md](components.md)。

## 校验时机

前端校验只为体验，可以被绕过，后端规则见 [input-and-injection.md](../backend/security/input-and-injection.md)。

### 前后端共享校验规则 【推荐】

共用 Zod Schema 或从 OpenAPI 生成，类型用 `z.infer` 推导，避免规则写两份后逐渐不一致。用于服务端入参的 Schema 要满足 [input-and-injection.md](../backend/security/input-and-injection.md) 的要求，比如拒绝未知字段、字符串写长度上限。

**示例：**

```ts
const profileSchema = z.strictObject({
  name: z.string().min(1).max(100),
  email: z.email().max(254),
});
type ProfileForm = z.infer<typeof profileSchema>;

const form = useForm<ProfileForm>({
  resolver: zodResolver(profileSchema),
  mode: 'onTouched',
  defaultValues: { name: '', email: '' },
});
```

**来源：** Zod 文档、React Hook Form 文档

### 用户离开字段后才显示错误 【推荐】

用户还没操作时不提示错误，字段出错后随输入更新。React Hook Form 用 `mode: 'onTouched'`。CSS 用 `:user-invalid`，`:invalid` 在交互前就会匹配。

**来源：** web.dev 表单校验、React Hook Form 文档

### 跨字段规则写在 Schema 中 【推荐】

比如日期范围，用 `.refine()` 校验，并通过 `path` 把错误指向具体字段。校验函数返回假值表示失败，不抛异常。异步校验要用 `parseAsync`。

**来源：** Zod 文档

## 受控策略

### 表单状态交给表单库 【推荐】

React 用 React Hook Form，Vue 用 VeeValidate。原生输入用 `register`，第三方受控组件用 `Controller`。编辑中的表单不放进全局 Store，提交后再更新，否则每次按键都有开销却没有收益。

**例外：** 所见即所得的实时预览可以放入 Store。

**来源：** Redux 风格指南、React Hook Form 文档

### React Hook Form 的字段都提供默认值 【推荐】

每个字段都写 `defaultValues`，空字段写 `''`。`defaultValues` 只在首次渲染时读取，数据后到时用 `values`，不在 `useEffect` 里调用 `reset`。

**原因：** `isDirty` 依赖与默认值比较，缺失时字段值为 `undefined`，脏状态也不准确。

**示例：**

```ts
const form = useForm<ProfileForm>({ defaultValues: EMPTY_PROFILE, values: data });
```

**来源：** React Hook Form 文档

## 错误展示

### 错误展示在字段附近，提交失败不清空输入 【推荐】

错误文案说明如何修复，不只写“格式错误”。说明文字用 `aria-describedby` 与控件关联，颜色的要求见 [accessibility.md](accessibility.md)。

### 服务端字段错误映射回字段 【推荐】

把 422 返回的字段错误逐个写入表单，第一个出错的字段获得焦点。`toApiError` 把后端 `errors` 中的 `pointer` 转成表单路径，比如 `#/items/0/sku` 转成 `items.0.sku`。不属于具体字段的错误写入 `root`，展示在表单顶部。`setError` 不是校验规则，不能阻止下一次提交。

**原因：** 只弹一条通用提示时，用户不知道要改哪里。

**示例：**

```ts
Object.entries(error.fieldErrors).forEach(([name, message], index) => {
  form.setError(name, { type: 'server', message }, { shouldFocus: index === 0 });
});
```

**来源：** React Hook Form 文档

### Server Action 表单用 `useActionState` 【推荐】

Next.js 中，Action 把字段错误作为返回值，由 `useActionState` 接收并展示，`isPending` 用于提交中状态。Action 的要求见 [nextjs.md](nextjs.md)。

**来源：** Next.js 文档、React 文档

## 防重复提交

### 提交期间阻止重复触发 【必须】

请求期间禁用提交按钮并显示进行中状态。原生 `disabled` 会让按钮失去焦点，提交后把焦点移到结果提示，或改用 `aria-disabled` 并在提交函数中拦截。按钮禁用只是体验，关键操作由后端保证幂等，见 [rest.md](../backend/api/rest.md)，前端的幂等键见 [data-fetching.md](data-fetching.md)。

**原因：** 双击和网络慢时的重复点击会产生重复订单。

**示例：**

```tsx
<button type="submit" disabled={form.formState.isSubmitting}>
  {form.formState.isSubmitting ? '保存中' : '保存'}
</button>
```

**来源：** React Hook Form 文档、web.dev 登录表单最佳实践

### 不因表单未填完而禁用提交按钮 【推荐】

允许点击，再指出缺少的内容。否则用户会以为页面失效。

**来源：** web.dev 登录表单最佳实践

### 表单内的普通按钮写 `type="button"` 【必须】

按钮默认类型是 `submit`，“显示密码”这类按钮会直接提交表单。

**来源：** web.dev 登录表单最佳实践

## 复杂表单

### 动态列表使用表单库的数组 API 【必须】

使用表单库时走它的数组 API，React Hook Form 用 `useFieldArray`，列表 key 用库生成的 `field.id`，不用下标，否则增删后输入值会错位。`append` 等方法传入完整对象，不在同一处理函数中连续堆叠操作。

**来源：** React Hook Form 文档

### 明确条件字段隐藏后的取值 【推荐】

React Hook Form 默认保留已卸载字段的值，它们不参与内置校验，但仍在提交数据中。需要移除时设置 `shouldUnregister: true`，该选项不能与 `useFieldArray` 同用。后端拒绝未知字段，隐藏字段提交前要移除，或由 Schema 明确允许。用 `FormData` 收集时，禁用的字段不会被包含。

**来源：** React Hook Form 文档、MDN

### 分步表单共用一个表单实例 【推荐】

所有步骤共用一份状态和一份 Schema，切换步骤时校验当前步骤的字段，最终提交时校验全部字段。

### 大表单缩小状态订阅范围 【推荐】

把 `useFormState` 放在提交按钮这类叶子组件中，并在渲染前解构需要的字段。

**来源：** React Hook Form 文档

## 草稿

### 拦截离开的写法 【推荐】

需要拦截时，应用内跳转，React Router 用 `useBlocker` 并自行提供确认界面，Vue Router 用 `onBeforeRouteLeave`。关闭和刷新页面用 `beforeunload`，只在有未保存修改时注册，保存后移除。提交成功后解除拦截。

**来源：** MDN、React Router 文档、Vue Router 文档

### 本地草稿的写法 【推荐】

需要本地草稿时，防抖保存，并在 `visibilitychange` 时保存，移动端的 `beforeunload` 不可靠。key 包含用户和表单标识，避免共用设备上读到他人草稿。提交成功后清除，恢复前按 Schema 校验，并告知用户内容来自草稿。草稿按字段白名单保存，密码、证件号以及姓名、手机号、地址这类个人信息不写入本地。存储规则见 [state-management.md](state-management.md)。

**来源：** MDN
