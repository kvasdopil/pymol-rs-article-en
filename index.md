# PyMOL-RS — Molecular Visualization and Vibe Coding

> English translation of the Russian interview/article from VAS3K Club project #31005.

## Tell us about yourself and the project

My name is Pavel Yakovlev. I’m a computational biologist from Saint Petersburg, working in pharma.

For people far from molecular biology, here’s some context. Proteins, DNA, and drug molecules are all 3D objects. To work with them, you need to see them: rotate them, color chains, display surfaces, and inspect how a ligand sits in a receptor pocket. That’s what molecular viewers are for, and **PyMOL** is the undisputed king among them.

If you’ve ever seen a beautiful protein image in *Nature*, a biochemistry textbook, or a pharma deck, there’s a good chance it was made in PyMOL.

PyMOL appeared in 2000 and was created by Warren DeLano, one of the people who made structural biology truly visual. Over 25 years the program accumulated a huge feature set: from simple ball-and-stick rendering to advanced ray tracing, molecular surfaces, structural alignment, support for hundreds of file formats, and a scripting language that powers thousands of lab pipelines worldwide.

DeLano died by suicide in 2009 (which, sadly, led to a lot of dark humor in structural biology). Schrödinger now develops the commercial version, while the open-source one has largely stagnated.

**PyMOL-RS** is a full rewrite of PyMOL from scratch in Rust. Same command language, same workflows familiar to thousands of scientists, but with a modern core: WebGPU instead of early-2000s OpenGL 2.x, memory safety instead of manual C memory management, and a modular crate-based architecture instead of a 25-year monolith.

And yes, it has new features too.

This project was built almost entirely with Claude Code and became my first large-scale vibe-coding experience: over 65,000 lines of code. So this is not just a “look at my cool project” post. It’s mostly about using Claude Code in a genuinely large and algorithmically complex project—and how not to repeat Warren’s fate while doing molecular visualization.

## How did the idea appear?

On Old New Year’s Eve, we had bad Peking duck and spent two weeks with acute salmonellosis. After a few days I had lost 5 kg, stopped living between bed and toilet, and realized I had free time for a pet project for the first time in years.

Jokes aside, the motivation had been building for a long time.

Installing open-source PyMOL is painful. Schrödinger does a lot to prevent binary redistribution (and to nudge users toward the commercial product). Building from source feels intentionally difficult and requires many dependencies. Homebrew formulas exist, but they pull in dozens of packages, often with version conflicts and system chaos. There are long-standing issues too, like broken Jupyter integration on Macs.

The commercial version doesn’t solve this much better. Releases are tied to bundled Python versions, updates are infrequent, and there’s still no Apple Silicon version, even though macOS Tahoe is effectively the last Intel-Mac-friendly release. The UI is another pain point. Scientists are used to suffering, but this is too much. It’s often easier for me to write script commands than click through the interface.

So yes: huge product, immense legacy, but it feels half-dead.

This is typical for scientific software in general. Researchers are used to tools that are awkward, ugly, and slow. Science culture often prioritizes results and sacrifice over great creative environments. If you saw the UI of some crystallography or molecular dynamics software, you’d understand. This is not just annoying—it slows science itself, because people waste hours fighting tools instead of thinking about biology.

I had experimented with AI + PyMOL before. A few years ago I wrote a plugin called [pymol-gpt](https://github.com/zmactep/pymol-gpt), embedding ChatGPT into PyMOL. But eventually a bigger idea emerged: what if I rewrite everything the way I want it?

## What went into the prototype and how long did it take?

About 1.5 months from first commit to v0.1.0.

Team: me + Claude Code (+ Google Gemini at planning stage).

I started by discussing original PyMOL architecture with Gemini for several days. We reviewed source code, I asked questions, and Gemini helped shape decisions. The result was a ~100 KB architecture document describing target crates, dependencies, rendering design, and library choices. I edited and extended it manually, and it became the foundation for development.

At early stages, I fed this document to Claude Code and created a dedicated prompt per module. For example:

> “Here is the overall architecture (the 100 KB doc). Now implement `pymol-select`, an atom selection language parser. It must support these operators, behave like this, and return these structures.”

Claude wrote a lot, very fast, and generally well.

Later, when the skeleton was stable, I relied more on Claude’s planning mode—but still planned in detail before assigning tasks. A good prompt is not “make it nice,” but a mini-spec with explicit buttons, commands, separators, and expected behavior.

The more precise the prompt, the closer the output to what you want. It feels counterintuitive (more prompt work), but one detailed prompt usually saves five “no, I meant something else” iterations.

After 1.5 months, the output was:

- 13 independent crates
- 65k+ lines of code
- PyMOL-compatible selection language with 95+ keywords
- Parsers for 7 file formats
- GPU rendering with impostor shaders
- Ray tracing
- Structural alignment
- Python API via PyO3

## Tech stack and why

**Rust + wgpu (WebGPU) + egui + PyO3**

But honestly, the most important “technology” was Claude Code.

### Why Rust

1. Memory safety without garbage collection (critical for millions of atoms)
2. `cargo build` cross-platform convenience with far less dependency pain

One binary, predictable behavior—that’s exactly what original PyMOL lacks.

### Why wgpu

Modern graphics pipeline. Spheres are rendered via GPU impostors (quad + shader), not heavy triangle meshes from 2003. Faster and prettier.

### Why egui

Fast GUI with low boilerplate.

### Why PyO3

Python bindings for the familiar `from pymol_rs import cmd` workflow.

### How Claude Code was used

Workflow loop:

1. I define task
2. Claude writes code
3. I review
4. I patch manually or describe problems in prompts
5. Repeat

I kept a folder with “problem prompts” and fed it back regularly as examples of recurring mistakes and fixes.

One caveat: context fills quickly. 200k disappears in 2–3 serious requests.

For large planning and structured prompt drafting, I used Gemini.

A striking case: molecular surface generation. Claude noticed original PyMOL’s approach was suboptimal. I found papers with newer methods, shared them, and together we implemented a solution around **1000x faster** than original PyMOL in measured cases. Then we parallelized with Rayon, and full-protein surfaces went from “grab coffee” to “blink and done.”

Another case: Kabsch and CE structural alignment algorithms (SVD, optimal rotation, dynamic programming). Claude produced working Rust implementations of both on first attempt.

Then came visual quality. A colleague (Stas), famous for clear molecular visuals, preferred ChimeraX aesthetics over PyMOL. I inspected internals and found fundamentally different lighting models, not just setting differences. So PyMOL-RS now includes two lighting models:

- `classic` (PyMOL-like)
- `skripkin` (inspired by ChimeraX and Stas Skripkin’s preferences)

Switchable with one setting.

A big win was fast feedback: colleagues posted pain points in chat and got fixes the same evening. When “I want this” to “done” is hours instead of months, everything changes.

## Launch and first users

I published on GitHub, posted on Hacker News (Show HN), and announced in my Telegram channel.

HN feedback was useful: one experienced user found subtle SDF parser bugs and raised a fair licensing concern that some algorithms (notably DSS) looked too close to original PyMOL code. That prompted me to rewrite parts from C-style carryover into idiomatic Rust.

User base is still small, but the most important users are in our own computational biology department.

This is a niche product for structural biologists. But in this niche, a new molecular viewer is a once-in-a-decade event. Even a modest community would already make me happy.

## Biggest unexpected challenges

### 1) Secondary structure rendering (hardest)

For ~1.5 weeks I struggled with cartoon rendering of protein secondary structures (alpha helices, beta sheets, loops).

This is less algorithmic and more geometric/visual:

- proper polygon generation
- smooth transitions
- correct normal orientation
- tapering at beta-sheet arrow ends

You can’t fully describe these transitions in words. Claude kept generating compilable code, but visuals looked terrible.

What worked: I drew geometry by hand on paper, annotated transitions and normals, photographed sketches, and then described them in text. Only then did we achieve proper output.

**Lesson:** some things are faster to sketch than to explain. Shader and visual geometry work still demands strong human visual reasoning.

### 2) Architectural entropy

Claude writes quickly, but has little architectural taste by default. Left unchecked, it duplicates constants, introduces pointless wrappers, and places hacks in odd modules.

After each major feature I needed a cleanup/refactor session.

Typical prompt:

> “Review these files. Logic X is duplicated, constants Y are defined in three places, structure Z is pointless. Clean it up.”

Resulting dynamic: one hour to ship a feature, two hours to clean it. Still faster than fully manual coding, but not “instant magic.”

### 3) Cost

I started with Cursor + Claude Opus and burned through a $200 subscription in four days.

Then switched to Claude Code Max ($200/month), which I still use.

Large projects consume huge context and many iterations; pure token billing would have been financially painful.

## Money spent, money earned, monetization

- Claude Max: ~$200/month
- Cursor burn: ~$200 in four days
- Total so far: roughly **$600**

Revenue: **$0**

Project is BSD 3-Clause, fully open source.

Classic monetization is not planned; this is for the scientific community.

However, modular architecture enables reuse:

- Need a PDB parser? Use `pymol-io` + `pymol-mol` without GUI baggage.
- Need atom selection language in your own pipeline? Use `pymol-select`.

Original PyMOL is monolithic; this is not.

## Future plans

Near-term roadmap:

- electron density maps (`isomesh` / `isosurface`)
- video export
- object grouping
- stronger `.pse` session compatibility for seamless migration
- better Python integration (possibly bindings for more languages)

Longer-term dream:

- WASM build + browser runtime
- Jupyter plugin or standalone web service

Rust + wgpu make this realistic.

That would finally solve “installing PyMOL is an adventure.”

Open link → load structure → inspect/process → render movie → present. No installs, no dependency chaos.

Philosophically, I want to show that scientific software doesn’t have to be ugly and hostile. Tools should respect researchers’ time. Better tools produce better thinking, and better thinking produces better science.

## What help is needed from the community?

I write about reducing pain, but I’m still a product of this ecosystem.

I’m not a UI/UX specialist (and haven’t been a full-time software developer for years). I built a strong base for what I think can become a better molecular visualization platform, but I need design/UI input.

So practical comments on usability and visual quality are highly welcome.

If you’re a structural biologist and want to test it, the repo is open:

- [github.com/zmactep/pymol-rs](https://github.com/zmactep/pymol-rs)

## Advice for people following a similar path

Most important point: AI coding assistants are **not** “press a button, get a project.”

They multiply your existing expertise.

Claude Code doesn’t know by itself how to render molecular surfaces correctly or stitch cartoon geometry beautifully. But if you know the domain and can specify requirements, it can generate thousands of quality lines in hours instead of weeks.

Counterintuitive consequence: vibe coding works best when you deeply understand the domain. It’s not a tool for doing what you can’t do—it’s a tool for doing what you can do, much faster.

Practical advice:

1. **Invest in planning.** My 100 KB architecture document was arguably the most valuable artifact.
2. **Write detailed prompts.** Short prompts often cost more in retries and tokens.
3. **Schedule refactoring.** AI writes fast, not always clean. Architectural discipline is your job.
4. **Use different models for different jobs.** Gemini for planning, Claude Code for implementation/refactoring.
5. **Draw on paper when needed.** If the model doesn’t get it, prompt quality is usually the bottleneck.

And finally:

> Don’t eat suspicious Peking duck.
>
> Then again, without that duck there might never have been PyMOL-RS.
