# Extract a subdirectory into its own repo with history intact
<!-- verified: 2026-08 -->

```
git subtree split --prefix=<SUBDIR> -b <NEW_BRANCH>
```
Example:
```
git subtree split --prefix=packages/shared-ui -b split-shared-ui
```
This rewrites history so paths outside `<SUBDIR>` never existed — unlike copying the files out, which loses all history for them. Push that branch to a new, empty repo as its `main`.

## The trap right after splitting
Deleting the old subdirectory from the original repo also deletes whatever `.gitignore` lived inside it. If that `.gitignore` was the only thing ignoring a `node_modules/` (or `dist/`, `.env`, etc.) inside that folder, it becomes unignored — and the next `git add -A` anywhere in the repo can stage tens of thousands of files before anyone notices.

**Why it bites:** the failure shows up as a `git status` that looks like the whole repo suddenly changed, minutes after a totally unrelated split operation — nothing points back to the deleted `.gitignore` as the cause. Before deleting the old directory, add its patterns to the root `.gitignore`:
```
echo "<SUBDIR>/node_modules/" >> .gitignore
```
Example:
```
echo "packages/shared-ui/node_modules/" >> .gitignore
```
