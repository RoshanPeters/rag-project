# Learning log

## Phase 0 — Foundations (14–23 Nov 2026)

### What I built
*Describe in 3–5 bullet points what exists now. Imagine explaining it to a friend who knows Python but not git or CI.*

### What I tried
*The main steps, briefly: tools, the /health endpoint, the test, ruff, GitHub, CI, branch protection, the PR flow, the break-it experiments.*

### What went wrong and how I fixed it
*For each problem: what happened, how you found out, what the cause was, and how you fixed it.*

Reminders of real problems from this phase:
- *CI failed in "Set up job": `action/checkout` typo, and `astral-sh/setup-uv@v10` didn't exist (tags vs commit hashes).*
- *pytest's deprecation warning: `httpx` → `httpx2`.*
- *A break exercise left `== 202` in the test, so tests were red until it was changed back.*
- *Formatting issues in `main.py` that `ruff check` didn't flag but `ruff format --check` did.*
- *Your Gmail address would have been public in commits and `pyproject.toml`, so you switched to the noreply email before the first commit.*

### What surprised me
*At least two things. Something that worked differently from what you expected, or a concept that suddenly clicked (for example, branches being just labels, or CI starting from an empty machine).*

### What I'd do differently
*If you started Phase 0 again tomorrow, what would you change?*

### Checkpoint answers

**1. Dependency vs dev dependency: why does it matter for a Docker image later?**

*Your answer. Hint: which packages does the API need when it's running in production, and which are only for you while developing? What would extra packages cost in an image?*

**2. Why does CI install from scratch instead of reusing my machine's setup?**

*Your answer.*

**3. `uses:` vs `run:` in a workflow step?**

*Your answer, with one example of each from your own `ci.yml`.*

**4. Why `ruff format --check` in CI instead of `ruff format`?**

*Your answer.*

**5. Why might a load balancer need `/health`?**

*Your answer. Think about what you saw in Part C's "dead server" exercise.*

### Questions I still have
*Anything that's still fuzzy. Write it down even if it feels basic.*