# Prettier & pre-commit — quick invocations
<!-- verified: 2026-08 -->

## Format specific files with Prettier (without a save-on-format editor setup)
```
npx prettier --write <FILE_PATH>
```
Example:
```
npx prettier --write src/components/form/OrderForm.tsx
npx prettier --write src/types/schedule.ts
```
**Why it bites:** `npx prettier --write .` reformats the whole repo, including generated or vendored files if they're not in `.prettierignore`. Target specific files or globs when only a handful of files actually changed.

## Run all pre-commit hooks against the whole repo (not just staged files)
```
pre-commit run --all-files
```
**Why it bites:** a normal `git commit` only runs hooks against staged files. If a hook config or hook version changed, files nobody touched can now fail — `--all-files` is how that surfaces before CI (or a teammate) finds it instead.
