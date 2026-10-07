RPCS3 – Starhawk fix fork
=========================

This is a fork of [RPCS3](https://github.com/RPCS3/rpcs3) that fixes the broken lighting in **Starhawk** (2012). It follows upstream RPCS3 and only changes how the renderer handles a color target and a depth buffer that share the same memory.

The fork is based on upstream RPCS3 commit `925fbb260`. Everything below the line further down is the original RPCS3 README.

## The problem

In Starhawk, lit surfaces, characters and the sky render black or go missing ([RPCS3/rpcs3#11877](https://github.com/RPCS3/rpcs3/issues/11877)).

Starhawk uses a deferred renderer. It first draws a depth-only pass (Z-prepass), then draws its G-buffer into the **same memory** as that depth buffer. During the G-buffer pass, depth testing is on and depth writing is off, so the game tests against the prepass depth while writing color over it.

Upstream RPCS3 can only treat a block of memory as either color or depth. When depth testing is enabled it keeps the depth buffer and silently drops the color writes. Upstream added a manual workaround, *Framebuffer Aliasing Heuristic Bias → Prefer Color* ([RPCS3/rpcs3#18644](https://github.com/RPCS3/rpcs3/pull/18644)). It keeps the color writes but skips the depth test for those draws.

## The fix

When a draw writes color into memory that is also bound as the depth buffer, and only tests depth/stencil without writing it, this fork binds **both** views of that memory:

- The depth test runs against the depth data that was already there, such as the Z-prepass.
- The color writes land normally, and the color data becomes the newest owner of that memory. Later passes that read the memory as a texture see the G-buffer.

The fix does not depend on the game's region or release, and it needs no per-game setting or patch. It applies to any game that renders this way.

### Settings

The behavior is controlled by **Framebuffer Aliasing Heuristic Bias** on the **Debug** tab of the settings dialog. The Debug tab is hidden by default; you only need it to change away from Auto. To show it, close RPCS3 and set `showDebugTab=true` in the `[Meta]` section of `GuiConfigs/CurrentSettings.ini`. That file is in the RPCS3 folder on Windows and in `~/.config/rpcs3` on Linux.

| Option | Behavior |
|---|---|
| **Auto** (default, recommended) | Binds both the color and depth views when a draw writes color and only tests depth/stencil. All other cases use the previous depth-biased behavior. |
| Prefer Color | Same as upstream: keeps only the color target when color is written and depth/stencil is not. The depth test is skipped for those draws. |
| Prefer Depth | The previous upstream Auto behavior, unchanged. Use it if a game renders worse with this fork than with upstream RPCS3. |

If you set Prefer Color for Starhawk with an earlier build, set it back to **Auto**.

## Downloads

This fork's builds are made with GitHub Actions and are not published as releases.

1. Open the [Actions tab](https://github.com/grimmchickendead-glitch/rpcs3-SH-Fix/actions/workflows/rpcs3.yml) and select the most recent successful **Build RPCS3** run for the branch you want.
2. Download the build for your system from the **Artifacts** section at the bottom of the run page. You must be signed in to GitHub. Artifacts expire after 90 days.
   - Windows: **RPCS3 for Windows (MSVC)** (the same kind of build as official releases)
   - Linux: **RPCS3 for Linux (X64, clang)**
   - macOS: **RPCS3 for Mac (Apple Silicon)** or **(Intel)**
3. To make a new build, choose **Run workflow** on the same page and pick the branch.

On Windows, copy the `dev_flash`, `dev_hdd0` and `config` folders from your existing RPCS3 folder into the new one to keep your firmware, games and settings. On Linux and macOS these are kept in your user configuration folder and are shared automatically.

## Changes compared to upstream

All changes are in the RSX (GPU) emulation, plus one settings tooltip.

| Area | Files | Change |
|---|---|---|
| Framebuffer setup | `rpcs3/Emu/RSX/RSXThread.cpp`, `RSXThread.h` | Decides whether to keep depth, color, or both for each color target that shares the depth buffer's address. Re-evaluates when the game toggles depth, stencil or color writes between draws without rebinding its surfaces. |
| Surface cache | `rpcs3/Emu/RSX/Common/surface_store.h` | Lets a color surface and a depth surface live at the same address while both are bound: neither evicts the other, and data is inherited from whichever is newer. Address lookups return the newest view. Bound surfaces are protected from cleanup. |
| Texture cache | `rpcs3/Emu/RSX/Common/texture_cache.h` | When the shared memory is sampled during the pass, a depth-format read gets the depth view and a color read gets the color view. |
| Vulkan / OpenGL | `rpcs3/Emu/RSX/VK/VKGSRender.cpp`, `rpcs3/Emu/RSX/GL/GLRenderTargets.cpp` | Pass the new state to the surface cache. The read-only depth view does not lock or flush memory, because the color target owns that range. |
| UI | `rpcs3/rpcs3qt/tooltips.h` | Describes the three options. |

The full history is in `git log 925fbb260..` on the fix branch. It starts with an earlier per-title workaround that forced Prefer Color for Starhawk; the proper fix replaced it.

## Status and testing

- **Not yet confirmed in game.** No one has played Starhawk on these builds yet. Reports are welcome, both good and bad.
- **Compile check:** every changed source file was compile-checked locally.
- **Builds:** GitHub Actions builds the project for Windows, Linux, macOS and FreeBSD. The status of the latest build is on the Actions tab.
- **Surface cache test harness:** a standalone harness, not included in this repository, ran the real surface cache code with mock GPU surfaces through these scenarios:
  - Starhawk-style frames: depth clear, Z-prepass, G-buffer pass with both views bound, lighting pass.
  - Blits into the shared memory.
  - Skipped draws.
  - Several color targets on the same address.
  - Oversized texture reads.

  The harness also showed that the previous behavior loses the Z-prepass depth and shows stale data for the lighting pass.
- **Other games:** the change affects any game that renders color into its bound depth buffer while only testing depth. It has not been tested on other titles. The original heuristic (2017) was tuned with Tales of Vesperia, God of War HD and Assassin's Creed among others. If a game looks wrong only on this fork, try **Prefer Depth** and please report it.
- **Known limitation:** the depth view may be reloaded from memory that already contains the color output in the middle of a pass. This only happens when *Read Depth Buffer* and *Write Color Buffers* are both enabled. Neither is on by default.

**AI disclosure:** the code changes in this fork were researched and written with the help of an AI assistant (Claude). They were checked with the compile checks, test harness, CI builds and adversarial code reviews described above, not by playing the game. Upstream RPCS3 requires contributors to disclose AI involvement and to fully understand and test their changes; see *AI Use* below. Anyone submitting this work upstream must do that themselves first.

## Reporting problems

Open an issue on this repository and include:
- the game and its title ID
- your GPU, driver and renderer (Vulkan or OpenGL)
- the value of *Framebuffer Aliasing Heuristic Bias*
- a screenshot
- the `RPCS3.log` file from a run that shows the problem

The log contains a warning starting with `Framebuffer at ... has aliasing color/depth targets` whenever this code path is used.

---

RPCS3
=====

[![GitHub Actions](https://img.shields.io/github/actions/workflow/status/RPCS3/rpcs3/rpcs3.yml?branch=master&logo=github&label=Actions)](https://github.com/RPCS3/rpcs3/actions/workflows/rpcs3.yml)
[![RPCS3 Discord Server](https://img.shields.io/discord/272035812277878785?color=5865F2&label=RPCS3%20Discord&logo=discord&logoColor=white)](https://discord.gg/rpcs3)

The world's first free and open-source PlayStation 3 emulator/debugger, written in C++ for Windows, Linux, macOS and FreeBSD.

You can find some basic information on our [**website**](https://rpcs3.net/). Game info is being populated on the [**Wiki**](https://wiki.rpcs3.net/).
For discussion about this emulator, PS3 emulation, and game compatibility reports, please visit our [**forums**](https://forums.rpcs3.net) and our [**Discord server**](https://discord.gg/RPCS3).

[**Support the Lead Developers on Patreon**](https://rpcs3.net/patreon)

## Contributing

If you want to help the project but do not code, the best way to help out is to test games and make bug reports. See:
* [Quickstart](https://rpcs3.net/quickstart)

If you want to contribute as a developer, please take a look at the following pages:

* [Coding Style](https://github.com/RPCS3/rpcs3/wiki/Coding-Style)
* [Developer Information](https://github.com/RPCS3/rpcs3/wiki/Developer-Information)

You should also contact any of the developers in the forums or in the Discord server to learn more about the current state of the emulator.

### AI Use

Use of AI tools for research and reverse engineering purposes is permitted. However, contributors are expected to fully own and understand all code they submit. Any communication with the team — including code, code comments, and GitHub comments — must come from the human contributor, not an AI agent acting autonomously.

We have unfortunately seen a rise in untested and unverified AI-generated slop being submitted to this project. This wastes maintainer time and, in worse cases, such changes get merged and break functionality for all users. Repeated violations will result in a ban from the repository. Please be respectful of everyone's time.

**Pull requests opened by AI agents or automated tools must include a disclosure in the PR description** stating the scope of AI involvement — which parts were AI-generated and what human testing or review was performed prior to submission. PRs that omit this disclosure may be closed without review.

If you are unsure about your work, open a discussion issue to talk it through with the team, or reach out to a maintainer on [Discord](https://discord.gg/RPCS3).

## Building

See [BUILDING.md](BUILDING.md) for more information about how to setup an environment to build RPCS3.

## Running

Check our friendly [quickstart](https://rpcs3.net/quickstart) guide to make sure your computer meets the minimum system requirements to run RPCS3.

Don't forget to have your graphics driver up to date and to install the [Visual C++ Redistributable Packages for Visual Studio 2022](https://aka.ms/vs/17/release/VC_redist.x64.exe) if you are a Windows user.

## License

Most files are licensed under the terms of GNU GPL-2.0-only License; see LICENSE file for details. Some files may be licensed differently; check appropriate file headers for details.
