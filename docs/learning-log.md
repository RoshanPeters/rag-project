# Learning log

## Phase 0 — Foundations (14–27 Nov 2026)

### What I built

- A Python 3.12 project managed by **uv**: direct dependencies in `pyproject.toml`, exact versions of everything pinned in `uv.lock`.
- A tiny **FastAPI** app with one endpoint, `GET /health`, returning `{"status": "ok"}`, run locally with **uvicorn**.
- An automated **pytest** test that calls `/health` in memory with `TestClient` and checks both the status code and the body.
- **Ruff** for linting (probable bugs) and formatting (layout).
- A public **GitHub** repo with a **GitHub Actions** CI workflow (`checks` job) that runs `uv sync --locked`, `ruff check`, `ruff format --check` and `pytest` on every push and pull request to `main`.
- A **ruleset** protecting `main`: no deletion, no force push, changes only through a pull request, and the `checks` job must pass before merging.

### What I tried

1. Installed the tools: Homebrew, uv, gh, git.
2. Created the project with `uv init --package`, added FastAPI/uvicorn as dependencies and pytest/ruff/httpx as dev dependencies.
3. Wrote `/health`, ran it with `uv run uvicorn rag.main:app --reload`, called it with `curl -i`, explored the auto-generated `/docs` page.
4. Wrote a test with `TestClient` and ran it with `uv run pytest`.
5. Ran `ruff check`, `ruff format --check` and `ruff format`, and saw that the linter and formatter catch different things.
6. Made the first commit, logged in with `gh auth login`, created the repo with `gh repo create --public --source=. --push`.
7. Wrote `.github/workflows/ci.yml` and watched it run with `gh run watch`.
8. Created a ruleset on `main` and proved a direct `git push` to `main` is rejected (`GH013`).
9. Practised the daily flow with PR #1 (CI badge in the README): branch → commit → push → PR → CI → squash merge → pull.
10. Broke things on purpose: PR #2 (failing test) and PR #3 (bad formatting). Both were blocked from merging, fixed on the same branch, then closed.

### What went wrong and how I fixed it

- **First CI run failed in "Set up job" after 2 seconds.**
  Found out with `gh run view <id>`, which showed the annotation: `Unable to resolve action action/checkout` and `unable to find version v10`.
  Two causes: a typo (`action/checkout` instead of `actions/checkout`), and `astral-sh/setup-uv` doesn't publish a moving `v10` tag, only exact versions.
  Fix: corrected the owner, and pinned setup-uv to the commit hash of v10.3.3 with a `# v10.3.3` comment. Lesson: GitHub resolves every `uses:` action *before* running any step, so a bad action name fails the whole job immediately.
- **Deprecation warning in pytest** (`Using httpx with starlette.testclient is deprecated; install httpx2`).
  Fix: `uv remove --dev httpx` then `uv add --dev httpx2`. Lesson: fix warnings early, before an upgrade turns them into errors.
- **Tests were red because a break exercise was left in** (`== 202` instead of `== 200`).
  Fix: changed it back. Lesson: always undo a break exercise before moving on.
- **`ruff check` said "All checks passed!" but the code was badly formatted** (single quotes, missing spaces, one blank line instead of two).
  `ruff format --check` caught it; `ruff format` fixed it. Lesson: lint and format are separate checks, and CI needs both.
- **My personal email would have been public** in every commit and in `pyproject.toml`.
  Fix: turned on GitHub email privacy, switched `git config --global user.email` to the noreply address *before* the first commit, and updated `pyproject.toml`.

### What surprised me

- **CI starts from a completely empty machine.** It doesn't even have my code until `actions/checkout` downloads it, and it doesn't use my `.venv` at all. That's exactly why it catches "works on my machine" problems.
- **A branch is just a label pointing at a commit**, not a copy of the project. Switching branches made the README badge line disappear and reappear in VS Code.
- **The protection lives on GitHub, not on my laptop.** I can still commit to `main` locally; it's only the push that gets rejected.
- **When one CI step fails, the later steps are skipped.** In PR #3 the formatting failure meant the tests never ran at all.

### What I'd do differently

- Check exact action versions (or tags) on GitHub before writing `uses:` lines, instead of assuming a major-version tag exists.
- Set up the noreply email at the same time as git `user.name`/`user.email` in Part A, not just before the first commit.
- Install the Ruff VS Code extension with format-on-save from the start, so formatting is never a CI failure.
- Run the three CI commands locally before every commit, as a habit.

### Checkpoint answers

**1. Dependency vs dev dependency: why does it matter for a Docker image later?**

A dependency (`fastapi`, `uvicorn`) is needed for the app to *run*. A dev dependency (`pytest`, `ruff`, `httpx2`) is only needed while *developing and checking* the code. The production Docker image should install only the runtime dependencies (for example `uv sync --no-dev`). Leaving dev tools out makes the image smaller (faster to build, push and start), and reduces the attack surface: every extra package is more code that could have a vulnerability, and none of it is used in production anyway.

**2. Why does CI install from scratch instead of reusing my machine's setup?**

My laptop has history: packages installed by hand, leftover files, a `.venv` built over many commands. Code can pass locally *because of* something that only exists on my machine. CI starts on a fresh, empty runner and builds everything only from what's committed in git (`uv.lock`, `.python-version`, the code). If it passes there, it passes because of what's in the repo. That catches forgotten files, packages I imported but never added, and a `uv.lock` that doesn't match `pyproject.toml`. It's also the same process every time, for everyone.

**3. `uses:` vs `run:` in a workflow step?**

`run:` executes a shell command on the runner, exactly like typing it in a terminal, for example `run: uv run pytest`. `uses:` runs a published, reusable Action that someone else wrote and versioned on GitHub, for example `uses: actions/checkout@v7` (downloads my repo onto the runner) or `uses: astral-sh/setup-uv@<hash> # v10.3.3` (installs uv with caching). Use `uses:` for common setup that's already solved, and `run:` for my own commands.

**4. Why `ruff format --check` in CI instead of `ruff format`?**

CI's job is to *check*, not to change code. The runner is thrown away after the job, so if `ruff format` rewrote files there, nobody would ever see the fixes, and `main` would still contain the unformatted code. `--check` only reports and exits with code 1 if anything would change, which fails the job and blocks the PR. The fix then happens on my laptop (`ruff format .`), gets committed and goes through review like any other change, exactly as in PR #3.

**5. Why might a load balancer need `/health`?**

A load balancer spreads traffic across several copies (instances) of the app. It needs a cheap, fast way to ask each one "are you alive and able to serve requests?" It calls `/health` every few seconds. A `200` means healthy, so keep sending traffic. A non-200, a timeout, or "connection refused" (what I saw when I stopped uvicorn in Part C) means unhealthy, so stop sending traffic there, and possibly restart or replace the instance. This lets the system route around broken instances automatically. The same idea is used by Docker health checks (Phase 4) and AWS ALB / ECS (Phase 5).

### Questions I still have

- Should `/health` also check the database once Postgres exists in Phase 1, or should it stay as simple as possible? (Difference between a "liveness" and a "readiness" check?)
- When should I pin GitHub Actions to commit hashes instead of tags, and how do I keep them updated (Dependabot)?
- How do I split CI into separate jobs so a formatting failure doesn't hide test results?
