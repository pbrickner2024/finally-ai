# Review Findings

## 1. High: `Stop` hook can recursively spawn review runs and rewrite the repo on every stop

- File: `.claude/settings.json:7`
- The new `Stop` hook runs `codex exec "Review changes since last commit and write results to a file named planning/REVIEW.md"` whenever a Claude session stops.
- That child `codex exec` invocation will itself stop when it finishes, which gives it the same chance to trigger the hook again. Even if the host avoids literal infinite recursion, this still creates repeated review runs and repeated rewrites of `planning/REVIEW.md` from a single stop event.
- This is also not scoped to review-related work. Any unrelated Claude task that ends will dirty the repo by regenerating `planning/REVIEW.md`, which is a workflow regression.

## 2. Medium: Quick start no longer creates `.env`, so the documented Docker command fails as written

- File: `README.md:29`
- The old quick start at least attempted a concrete setup step before `docker run --env-file .env ...`. The new version only leaves a commented example line and never creates `.env`.
- If a user follows the snippet verbatim, Docker will fail because `.env` does not exist. The docs need either a real file-creation step or an inline here-doc / copyable example block that clearly instructs the user to create the file before running Docker.

## Notes

- I did not find other material code-level issues in the remaining diff. The rest of the changes are documentation or local tooling additions.
