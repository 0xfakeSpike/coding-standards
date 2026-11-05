# Frontend forms

This document covers form validation, state, error display, and submission. For form element semantics, see [html.md](html.md). For accessibility requirements, see [accessibility.md](accessibility.md). For controlled and uncontrolled components in general, see [components.md](components.md).

## Validation timing

Frontend validation is for user experience only and can be bypassed. For backend rules, see [input-and-injection.md](../backend/security/input-and-injection.md).

### Share validation rules between frontend and backend [Recommended]

Share a Zod Schema or generate from OpenAPI, and infer types with `z.infer`, so that two copies of the rules do not drift apart. Schemas used for server-side input must meet [input-and-injection.md](../backend/security/input-and-injection.md), such as rejecting unknown fields and setting length limits on strings.

**Example:**

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

**Source:** Zod Docs, React Hook Form Docs

### Show errors only after the user leaves a field [Recommended]

Do not show errors before the user acts. Once a field has an error, update it as the user types. In React Hook Form, use `mode: 'onTouched'`. In CSS, use `:user-invalid`, because `:invalid` matches before interaction.

**Source:** web.dev: Form Validation, React Hook Form Docs

### Put cross-field rules in the Schema [Recommended]

For date ranges, for example, validate with `.refine()` and point the error at a specific field with `path`. The validation function returns a falsy value to signal failure and does not throw. Async validation needs `parseAsync`.

**Source:** Zod Docs

## Control strategy

### Leave form state to a form library [Recommended]

React uses React Hook Form, and Vue uses VeeValidate. Native inputs use `register`, and third-party controlled components use `Controller`. Do not put in-progress forms in a global Store. Update it after submit. Otherwise every keystroke has a cost with no benefit.

**Exceptions:** Live WYSIWYG previews may go in the Store.

**Source:** Redux Style Guide, React Hook Form Docs

### Give every React Hook Form field a default value [Recommended]

Write `defaultValues` for every field, with `''` for empty fields. `defaultValues` is read only on first render. When data arrives later, use `values`. Do not call `reset` in `useEffect`.

**Why:** `isDirty` relies on comparison with default values. When they are missing, the field value is `undefined` and the dirty state is inaccurate.

**Example:**

```ts
const form = useForm<ProfileForm>({ defaultValues: EMPTY_PROFILE, values: data });
```

**Source:** React Hook Form Docs

## Error display

### Show errors near the field, and keep input after a failed submit [Recommended]

Error text says how to fix the problem, not just "Invalid format". Associate help text with the control through `aria-describedby`. For color requirements, see [accessibility.md](accessibility.md).

### Map server field errors back to fields [Recommended]

Write the field errors from a 422 into the form one by one, and focus the first field with an error. `toApiError` converts `pointer` in the backend's `errors` to a form path, such as `#/items/0/sku` to `items.0.sku`. Errors that do not belong to a specific field go into `root` and display at the top of the form. `setError` is not a validation rule and does not block the next submit.

**Why:** With only one generic message, users do not know what to change.

**Example:**

```ts
Object.entries(error.fieldErrors).forEach(([name, message], index) => {
  form.setError(name, { type: 'server', message }, { shouldFocus: index === 0 });
});
```

**Source:** React Hook Form Docs

### Use `useActionState` for Server Action forms [Recommended]

In Next.js, an Action returns field errors as its return value, `useActionState` receives and displays them, and `isPending` drives the submitting state. For Action requirements, see [nextjs.md](nextjs.md).

**Source:** Next.js Docs, React Docs

## Duplicate submit prevention

### Block repeat triggers during submit [Required]

During the request, disable the submit button and show an in-progress state. Native `disabled` makes the button lose focus, so after submit move focus to the result message, or use `aria-disabled` and block in the submit function. Disabling the button is only user experience. For critical operations, the backend guarantees idempotency. See [rest.md](../backend/api/rest.md). For the frontend idempotency key, see [data-fetching.md](data-fetching.md).

**Why:** Double clicks and repeated clicks on slow networks create duplicate orders.

**Example:**

```tsx
<button type="submit" disabled={form.formState.isSubmitting}>
  {form.formState.isSubmitting ? 'Saving' : 'Save'}
</button>
```

**Source:** React Hook Form Docs, web.dev: Sign-In Form Best Practices

### Do not disable the submit button because a form is incomplete [Recommended]

Allow the click, then point out what is missing. Otherwise users think the page is broken.

**Source:** web.dev: Sign-In Form Best Practices

### Write `type="button"` on plain buttons in forms [Required]

The default button type is `submit`, so a button such as "Show password" submits the form directly.

**Source:** web.dev: Sign-In Form Best Practices

## Complex forms

### Use the form library's array API for dynamic lists [Required]

When using a form library, go through its array API. React Hook Form uses `useFieldArray`. For list keys, use the library-generated `field.id`, not the index, or input values get misplaced after insertion and deletion. Pass complete objects to methods such as `append`, and do not stack operations back to back in the same handler.

**Source:** React Hook Form Docs

### Define the values of hidden conditional fields [Recommended]

React Hook Form keeps the values of unmounted fields by default. They skip built-in validation but stay in the submitted data. To remove them, set `shouldUnregister: true`. This option cannot be used with `useFieldArray`. The backend rejects unknown fields, so remove hidden fields before submit, or have the Schema explicitly allow them. When collecting with `FormData`, disabled fields are not included.

**Source:** React Hook Form Docs, MDN

### Share one form instance across steps [Recommended]

All steps share one state and one Schema. Validate the current step's fields when switching steps, and validate all fields on final submit.

### Narrow state subscriptions in large forms [Recommended]

Put `useFormState` in leaf components such as the submit button, and destructure the needed fields before render.

**Source:** React Hook Form Docs

## Drafts

### How to block leaving [Recommended]

When blocking is needed: for in-app navigation, React Router uses `useBlocker` with your own confirmation UI, and Vue Router uses `onBeforeRouteLeave`. For closing and refreshing the page, use `beforeunload`, registered only when there are unsaved changes and removed after save. Remove the block after a successful submit.

**Source:** MDN, React Router Docs, Vue Router Docs

### How to save local drafts [Recommended]

When local drafts are needed, save with debounce and also on `visibilitychange`, because `beforeunload` is unreliable on mobile. The key includes the user and form identifiers, to avoid reading someone else's draft on a shared device. Clear the draft after a successful submit. Before restoring, validate it against the Schema and tell the user the content comes from a draft. Save drafts by a field allowlist. Do not write passwords, ID numbers, or personal information such as names, phone numbers, and addresses locally. For storage rules, see [state-management.md](state-management.md).

**Source:** MDN
