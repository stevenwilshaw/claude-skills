# Storybook (for `src/components`)

## Where it lives: inside `apps/web`, not its own workspace

Storybook has to render the components with the same Vite config, Tailwind setup, aliases (`@/`) and React version as the app. A separate `apps/storybook` workspace would need to duplicate or cross-import all of that and keep dependency versions in lockstep, for no benefit - the components are not a published package. Keeping `.storybook/` in `apps/web` gives one dependency tree and stories that sit next to the components they document. (If the user later wants a publishable design-system package, extract `src/components` into `packages/ui` and move Storybook there - mention this option in the wrap-up.)

## Install

Run in `apps/web`: `npm i -D storybook @storybook/react-vite @storybook/addon-docs @storybook/addon-a11y` (all same major version - install `@latest` for each and confirm they match). `npx storybook@latest init` is an acceptable alternative if you then fix the stories glob below and delete the example stories it generates.

Scripts in `apps/web/package.json`:
```json
"storybook": "storybook dev -p 6006",
"build-storybook": "storybook build"
```
Add a root script `"storybook": "npm run storybook -w apps/web"`.

## `apps/web/.storybook/main.ts`

```ts
import type { StorybookConfig } from '@storybook/react-vite';

const config: StorybookConfig = {
  framework: '@storybook/react-vite',
  // Only the shared components - screens are tested with Vitest instead.
  stories: ['../src/components/**/*.stories.@(ts|tsx)'],
  addons: ['@storybook/addon-docs', '@storybook/addon-a11y'],
  viteFinal: (viteConfig) => ({
    ...viteConfig,
    // Storybook reuses vite.config.ts; drop the TanStack Router code generator, which is app-only.
    plugins: (viteConfig.plugins ?? []).flat().filter((p) => !(p && 'name' in p && /tanstack.*router/i.test(p.name))),
  }),
};
export default config;
```
If the plugin filter is not needed (Storybook runs fine with the router plugin present) leave it out. Check by running it.

`apps/web/.storybook/preview.ts`
```ts
import type { Preview } from '@storybook/react-vite';
import '../src/index.css'; // Tailwind

const preview: Preview = { parameters: { layout: 'centered' } };
export default preview;
```

## Stories

Co-locate: `components/inputs/TextInput.stories.tsx`, `SelectInput.stories.tsx`, `Button.stories.tsx`, `Popover.stories.tsx`.
```tsx
import type { Meta, StoryObj } from '@storybook/react-vite';
import { TextInput } from './TextInput';

const meta = { title: 'Inputs/TextInput', component: TextInput, tags: ['autodocs'] } satisfies Meta<typeof TextInput>;
export default meta;
type Story = StoryObj<typeof meta>;

export const Default: Story = { args: { placeholder: 'Type here' } };
export const Invalid: Story = { args: { invalid: true, defaultValue: 'oops' } };
export const Disabled: Story = { args: { disabled: true, defaultValue: 'Read only' } };
```
Generic components need an instantiation expression (`component: SelectInput<Option>`), and controlled ones (`ImageInput`) need a small `useState` host in `render`. The Field components need form context: write their stories with a `render` that mounts a small `useAppForm` host (same trick as their Vitest test).

Add `storybook-static` to `.gitignore` and `.dockerignore`. Verify with `npm run build-storybook -w apps/web` (it must succeed), and also confirm `npm run build` for the app does not pick up the stories (they are excluded from the app bundle because nothing imports them; keep `*.stories.tsx` out of `tsconfig` excludes only if they typecheck - they should be typechecked).
