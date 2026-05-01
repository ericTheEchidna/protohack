Protohack / nethack-protohack
GitHub Actions CI — replace azure-pipelines

## Phase N4 — CI

Replace `azure-pipelines.yml` with a GitHub Actions workflow that builds
nethack-protohack using the Meson build (Phase N1) on Linux, macOS, and
Windows. Prerequisite: Phase N1 complete. Can run in parallel with N2/N3.

### Context

The repo has `azure-pipelines.yml` from upstream NetHack. It uses the
old Makefile build system and is not useful once Meson is in place.
GitHub Actions is the natural target since the repo is on GitHub.

The CI matrix should validate that the Meson build stays green across all
three supported platforms on every push to the main branch and on every
PR.

### Tasks

- [ ] Create `.github/workflows/build.yml`
- [ ] Define build matrix:
  ```
  os: [ubuntu-latest, macos-latest, windows-latest]
  ```
- [ ] Each matrix job:
  - [ ] Check out repo with `submodules: recursive` (needed for Lua)
  - [ ] Install Meson and Ninja via pip or system package manager
  - [ ] Linux: `sudo apt-get install -y libncurses-dev libao-dev`
  - [ ] macOS: `brew install meson ninja ncurses`
  - [ ] Windows: install Meson via pip; MSVC available via
        `actions/setup-msvc` or use the pre-installed VS toolchain
  - [ ] `meson setup builddir`
  - [ ] `ninja -C builddir`
  - [ ] Smoke test: `./builddir/nethack --version` or equivalent
        non-interactive flag to confirm the binary runs
- [ ] Set the workflow to trigger on:
  - `push` to `NetHack-3.7` (main branch)
  - `pull_request` targeting `NetHack-3.7`
- [ ] Add a status badge to `README` or `CLAUDE.md`
- [ ] Delete or move `azure-pipelines.yml` to `outdated/` once the
      Actions workflow is green on all three platforms

### Complexity notes

- Windows MSVC builds require the `cl.exe` compiler to be on PATH.
  GitHub's `windows-latest` runner has VS installed; use
  `actions/setup-msvc` or the built-in `Developer Command Prompt` step
  pattern to activate it before running Meson.
- MinGW is an alternative to MSVC on Windows and simpler to set up in CI,
  but MSVC is the more realistic Windows target. Start with MSVC; add
  MinGW as a second matrix entry only if there is a concrete need.
- The smoke test must be non-interactive — nethack's normal startup
  requires a terminal. Use a `--version` flag or equivalent if NetHack
  3.7 supports it; otherwise use `echo | ./nethack` to send EOF
  immediately and check the exit code is not a crash (signal/SIGSEGV).
- Do not cache the Meson build directory between runs — NetHack's
  generated sources (`pm.h`, `onames.h`, `tilemap.c`) must regenerate
  cleanly from scratch to catch any custom target ordering issues.

### Done when

- GitHub Actions workflow runs green on all three platforms (Linux,
  macOS, Windows) on a push to the main branch
- Each job completes in under 15 minutes (cold build)
- `azure-pipelines.yml` is removed or moved to `outdated/`
- A failing build on any platform blocks PR merge (branch protection
  rule — document this as a manual step in GitHub repo settings)
