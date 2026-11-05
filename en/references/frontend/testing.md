# Frontend testing

This document covers how to test components and pages. See [Testing strategy](../general/testing-strategy.md) for test layers, [TypeScript testing](../languages/typescript/testing.md) for test frameworks, test doubles, and async assertions, and [Accessibility](accessibility.md) for accessibility requirements.

## Component tests

### Test behavior users can see [Recommended]

Verify components through rendered output and user actions. Do not assert on internal state, instance methods, or CSS class names.

Avoid:

```ts
expect(wrapper.vm.isOpen).toBe(true);
```

Prefer:

```ts
await user.click(screen.getByRole('button', { name: 'Open menu' }));
expect(screen.getByRole('menu')).toBeVisible();
```

**Why:** Tests tied to implementation details fail in large numbers during refactors, yet miss what actually affects users.

**Source:** Testing Library Docs

### Query elements from the user's perspective [Recommended]

Priority order: `getByRole`, `getByLabelText`, `getByPlaceholderText`, `getByText`, `getByDisplayValue`, then `getByAltText`, `getByTitle`, and `getByTestId` only as a last resort. Usually pair `getByRole` with `name`. When `getByRole` cannot find an element, first check the UI for accessibility problems.

**Tooling:** `eslint-plugin-testing-library`

**Source:** Testing Library Docs

### Simulate interaction with `user-event` [Recommended]

Call `userEvent.setup()` before rendering. `fireEvent` dispatches a single DOM event. Use it only when `user-event` cannot do the job.

**Why:** `user-event` simulates the full interaction, including focus and keyboard behavior.

**Source:** Testing Library Docs

### Wait for async content with `findBy` [Recommended]

Use `findBy*` to wait for elements to appear and `waitFor` for other conditions. Do not write fixed-duration waits.

```ts
expect(await screen.findByText('Saved successfully')).toBeVisible();
```

**Source:** Testing Library Docs

### Write few large snapshots [Recommended]

Use snapshots only for small, stable output. Large snapshots are easily updated without review and lose their protective value.

**Source:** Storybook Docs

## MSW

Intercept HTTP dependencies with MSW. See [TypeScript testing](../languages/typescript/testing.md). The same handlers work for Node tests, browser development, and Storybook.

### Baseline handlers describe only the success path [Recommended]

Put initial handlers in a shared file, describing only success scenarios. Override error and edge scenarios inside tests with `server.use()`, and call `server.resetHandlers()` in `afterEach`.

```ts
beforeAll(() => server.listen({ onUnhandledFrame: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('shows a retry option when loading fails', async () => {
  server.use(http.get('/api/orders', () => new HttpResponse(null, { status: 500 })));
  render(<OrderList />);
  expect(await screen.findByRole('button', { name: 'Retry' })).toBeVisible();
});
```

**Source:** MSW Docs

### Treat unhandled requests as errors [Recommended]

Undeclared requests must fail the test. The default `warn` strategy only prints a warning, and the request still goes out. `onUnhandledFrame` requires MSW 3 or later. Check the docs for your version for the option name.

**Why:** Once tests reach the real network, results are no longer deterministic.

**Source:** MSW Docs

## E2E

### Cover only core user paths [Recommended]

Cover core paths such as login and ordering with Playwright or Cypress. Leave edge cases to component tests and integration tests.

### Use locators and auto-waiting assertions [Recommended]

Use user-facing locators such as `getByRole`. Do not depend on CSS structure. Use auto-retrying assertions. Do not use `page.waitForTimeout()`.

Avoid:

```ts
await page.waitForTimeout(2000);
expect(await page.locator('.toast > span').isVisible()).toBe(true);
```

Prefer:

```ts
await expect(page.getByText('Saved successfully')).toBeVisible();
```

**Why:** Fixed waits are inherently flaky, and manual assertions check immediately without waiting.

**Tooling:** `eslint-plugin-playwright`

**Source:** Playwright Docs

### Isolate every test [Recommended]

Each test has its own storage, cookies, and data, and does not depend on the execution order of other tests. Log in once with a setup project and reuse the saved state. State files contain sensitive cookies, so add them to `.gitignore`. External sites and third-party APIs are out of your control, so return fixed responses with `page.route`.

**Source:** Playwright Docs

## Visual regression

### Generate and compare baselines in a fixed environment [Recommended]

Run baselines and comparisons on the same OS, browser version, and settings, usually in a CI container. Do not generate them on development machines.

**Why:** Rendering varies with system, fonts, and hardware, so different environments produce many false positives.

**Source:** Playwright Docs

### Eliminate unstable content [Recommended]

Use fixed data, and hide changing areas such as times and animations with `stylePath`. Relax `maxDiffPixels` and `threshold` only for known rendering jitter, not to mask differences. Keep baseline changes in the same change as the code.

**Source:** Playwright Docs

## Accessibility testing

### Put axe checks in tests [Recommended]

Run axe in E2E tests or Storybook, scoped to the target WCAG tags, including both Level A and Level AA. Fail the test on violations. axe skips hidden regions, so open dialogs and menus and run it again.

```ts
const results = await new AxeBuilder({ page })
  .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa'])
  .analyze();
expect(results.violations).toEqual([]);
```

**Source:** Playwright Docs, axe-core Docs

### Manually confirm axe results and scope exemptions [Recommended]

On average, axe automatically finds only about 57% of WCAG issues. Manually confirm items with an "incomplete" result. See [Accessibility](accessibility.md) for manual checks. Keep exemptions for known issues narrow: `exclude()` turns off all rules for an element and its children, and `disableRules()` turns off rules for the whole scan.

**Source:** axe-core Docs, Playwright Docs

## Storybook

Write one Story per state. See [components.md](components.md). Stories are also render tests and can be reused by Vitest and Playwright.

### Write interaction tests with `play` functions [Recommended]

Queries in `play` follow the priority above. `await` all `userEvent` and `expect` calls. In Vite projects, run Stories in CI with the Vitest plugin.

```ts
export const Submitted: Story = {
  play: async ({ canvas, userEvent }) => {
    await userEvent.click(canvas.getByRole('button', { name: 'Submit' }));
    await expect(canvas.getByText('Submitted successfully')).toBeVisible();
  },
};
```

**Source:** Storybook Docs

### Set accessibility checks to fail [Recommended]

Set `parameters.a11y.test` to `'error'` in `.storybook/preview`, and set Stories not yet fixed to `'todo'` individually.

**Source:** Storybook Docs
