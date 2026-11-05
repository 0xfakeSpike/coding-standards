# 前端测试

本文讲组件和页面的测试方法。测试分层见 [测试策略](../general/testing-strategy.md)，测试框架、替身和异步断言见 [TypeScript 测试](../languages/typescript/testing.md)，无障碍要求见 [无障碍](accessibility.md)。

## 组件测试

### 测试用户能看到的行为 【推荐】

通过渲染结果和用户操作验证组件，不断言组件内部状态、实例方法或 CSS 类名。

反例：

```ts
expect(wrapper.vm.isOpen).toBe(true);
```

正例：

```ts
await user.click(screen.getByRole('button', { name: '打开菜单' }));
expect(screen.getByRole('menu')).toBeVisible();
```

**原因：** 依赖实现细节的测试在重构时大量失败，却发现不了用户受到的影响。

**来源：** Testing Library 文档

### 按用户视角查询元素 【推荐】

优先级依次为：`getByRole`、`getByLabelText`、`getByPlaceholderText`、`getByText`、`getByDisplayValue`，然后是 `getByAltText`、`getByTitle`，最后才用 `getByTestId`。`getByRole` 一般配合 `name` 使用。用 `getByRole` 找不到元素时，先检查界面的无障碍问题。

**工具：** `eslint-plugin-testing-library`

**来源：** Testing Library 文档

### 用 `user-event` 模拟交互 【推荐】

在渲染前调用 `userEvent.setup()`。`fireEvent` 只派发单个 DOM 事件，只在 `user-event` 做不到时使用。

**原因：** `user-event` 模拟完整的交互过程，包括焦点和键盘行为。

**来源：** Testing Library 文档

### 异步出现的内容用 `findBy` 等待 【推荐】

等待元素出现用 `findBy*`，其他条件用 `waitFor`，不写固定时长的等待。

```ts
expect(await screen.findByText('保存成功')).toBeVisible();
```

**来源：** Testing Library 文档

### 少写大快照 【推荐】

快照只用于小而稳定的输出。大快照容易被不加检查地更新，失去保护作用。

**来源：** Storybook 文档

## MSW

HTTP 依赖用 MSW 拦截，见 [TypeScript 测试](../languages/typescript/testing.md)。同一组处理器可用于 Node 测试、浏览器开发和 Storybook。

### 基线处理器只描述成功路径 【推荐】

初始处理器放在共享文件中，只描述成功场景。错误和边界场景在测试内用 `server.use()` 覆盖，并在 `afterEach` 中调用 `server.resetHandlers()`。

```ts
beforeAll(() => server.listen({ onUnhandledFrame: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('加载失败时显示重试入口', async () => {
  server.use(http.get('/api/orders', () => new HttpResponse(null, { status: 500 })));
  render(<OrderList />);
  expect(await screen.findByRole('button', { name: '重试' })).toBeVisible();
});
```

**来源：** MSW 文档

### 未处理的请求按错误处理 【推荐】

未声明的请求让测试失败。默认策略 `warn` 只打印警告，请求仍会发出。`onUnhandledFrame` 需要 MSW 3 及以上，选项名以所用版本文档为准。

**原因：** 测试访问真实网络后，结果不再确定。

**来源：** MSW 文档

## E2E

### 只覆盖核心用户路径 【推荐】

用 Playwright 或 Cypress 覆盖登录、下单这类核心路径。边界情况放到组件测试和集成测试。

### 用定位器和自动等待的断言 【推荐】

用 `getByRole` 这类面向用户的定位器，不依赖 CSS 结构。用会自动重试的断言。不用 `page.waitForTimeout()`。

反例：

```ts
await page.waitForTimeout(2000);
expect(await page.locator('.toast > span').isVisible()).toBe(true);
```

正例：

```ts
await expect(page.getByText('保存成功')).toBeVisible();
```

**原因：** 固定时长的等待天生不稳定，手动断言不等待就立即判断。

**工具：** `eslint-plugin-playwright`

**来源：** Playwright 文档

### 每个测试相互隔离 【推荐】

每个测试有独立的存储、Cookie 和数据，不依赖其他测试的执行顺序。登录用 setup 项目执行一次，复用保存的状态。状态文件含敏感 Cookie，要加入 `.gitignore`。外部站点和第三方接口不受控，用 `page.route` 返回固定响应。

**来源：** Playwright 文档

## 视觉回归

### 在固定环境生成和比对基线 【推荐】

基线和比对在同一操作系统、浏览器版本和设置下运行，一般放在 CI 的容器中，不用开发机生成。

**原因：** 渲染结果随系统、字体和硬件变化，环境不同会产生大量误报。

**来源：** Playwright 文档

### 消除不稳定内容 【推荐】

用固定数据，用 `stylePath` 隐藏时间、动画这类变化区域。`maxDiffPixels` 和 `threshold` 只为已知的渲染抖动放宽，不用来掩盖差异。基线变化和代码放在同一个变更中。

**来源：** Playwright 文档

## 无障碍测试

### 把 axe 检查放进测试 【推荐】

在 E2E 或 Storybook 中运行 axe，并限定到目标 WCAG 标签，A 级和 AA 级都写。有违规时测试失败。axe 跳过隐藏区域，弹窗和菜单要打开后再跑一次。

```ts
const results = await new AxeBuilder({ page })
  .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa'])
  .analyze();
expect(results.violations).toEqual([]);
```

**来源：** Playwright 文档、axe-core 文档

### axe 结果的人工确认和豁免 【推荐】

axe 平均只能自动发现约 57% 的 WCAG 问题，结果为“不完整”的项要人工确认，手工检查见 [无障碍](accessibility.md)。已知问题的豁免范围要小：`exclude()` 会对元素及其子元素关闭所有规则，`disableRules()` 会在本次扫描中关闭规则。

**来源：** axe-core 文档、Playwright 文档

## Storybook

每种状态写一个 Story，见 [components.md](components.md)。Story 同时是渲染测试，可被 Vitest、Playwright 复用。

### 用 `play` 函数写交互测试 【推荐】

`play` 中的查询遵循上面的优先级，`userEvent` 和 `expect` 调用都要 `await`。Vite 项目用 Vitest 插件在 CI 中运行 Story。

```ts
export const Submitted: Story = {
  play: async ({ canvas, userEvent }) => {
    await userEvent.click(canvas.getByRole('button', { name: '提交' }));
    await expect(canvas.getByText('提交成功')).toBeVisible();
  },
};
```

**来源：** Storybook 文档

### 无障碍检查设为失败级别 【推荐】

在 `.storybook/preview` 中设置 `parameters.a11y.test` 为 `'error'`，还未修复的 Story 单独设为 `'todo'`。

**来源：** Storybook 文档
