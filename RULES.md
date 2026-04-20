# Review Quick Reference

Distilled from `docs/guidelines/`. Load this instead of the 4 individual guideline docs.

---

## File naming

| Type | Convention | Key rule |
|---|---|---|
| Components | `PascalCase.tsx` | Folder name must match component name exactly |
| Screens | `PascalCase.tsx` | **No `Screen` suffix** — the folder communicates that |
| Hooks | `kebab-case.ts` | Always starts with `use-` |
| Contexts | `kebab-case.ts` | **No `Context` suffix** on the file |
| Stores | `kebab-case.ts` | **No `Store` suffix** on the file |
| GQL queries | `kebab-case.ts` | No `.queries` suffix; avoid `get` prefix |
| GQL mutations | `kebab-case.ts` | Always starts with a verb |
| Barrels | `index.ts` | Exact — no variation |
| Tests | `<exact-name>.test.tsx/.ts` | Must match the file being tested; `.test` singular not `.tests` |
| Auxiliary files | `ComponentName.suffix.ts` | Suffix: types, styles, hooks, schema, a11y, constants, error, skeleton |
| Extensions | `.ts` / `.tsx` | `.tsx` only when the file contains JSX |

---

## Folder structure

- Feature screens are **flat**: `features/<name>/screens/ScreenName.tsx` — no subfolders inside `screens/`
- Sub-feature folders are only allowed under `components/`, never under `screens/`, `hooks/`, or `utils/`
- A sub-feature **cannot** have a sub-feature
- `index.ts` barrel is **required** at the top level of every feature subfolder (`screens/`, `hooks/`, `components/`, etc.)
- `index.ts` barrel is **required** at the top level of every global component folder
- Sub-feature folders use `kebab-case`; component folders inside them use `PascalCase`
- Import from the feature barrel when possible: `import { X } from '#/feature/screens'`

---

## Member naming

| Type | Convention | Notes |
|---|---|---|
| Variables, functions | `camelCase` | |
| Constants (scalar) | `UPPER_SNAKE_CASE` | |
| Constant objects | `PascalCase` | Members inside use `UPPER_SNAKE_CASE` |
| Types / interfaces | `PascalCase` | Members inside use `camelCase` |
| GQL documents | `UPPER_SNAKE_CASE` | |
| String literal values | `snake_case` | e.g., enum-like values |
| Component names | `PascalCase` | |
| Test IDs | `kebab-case` | |
| React keys | `kebab-case` | |

---

## Component structure

- All components typed as `React.FC<Props>` — not bare arrow functions
- `ComponentName/` folder → `ComponentName.tsx` (main), `index.ts` (barrel)
- Auxiliary files use dot-suffix pattern: `.types.ts`, `.styles.ts`, `.hooks.ts`, `.schema.ts`, `.a11y.ts`, `.constants.ts`, `.error.tsx`, `.skeleton.tsx`
- Storybook + barrel only required for global UI kit components (atomic design); feature components: optional
- Use `FF` prefix only when a same-named component already exists in another library (e.g., `FFButton` because RN has `Button`)
