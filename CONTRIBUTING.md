# Contributing to FSE Extensions

This repository holds stack-specific implementations of FlowState Engineering. The methodology lives in [fse-core](https://github.com/Joticle/fse-core). Extensions translate that methodology into the rules, patterns, and traps of a specific stack.

If you haven't read the core methodology, start there. An extension that doesn't align with the core isn't an extension — it's a different methodology wearing the same name.

## Two kinds of contribution

There are two ways to contribute, and they have different bars.

**1. New stack extension.** You're claiming an open stack and producing the four required files for it. This is a substantial contribution. The bar is high because once you submit it and it's accepted, you become the maintainer for that stack.

**2. Improvement to an existing stack.** You're fixing a typo, clarifying a rule, adding a pattern, or correcting an anti-pattern in a stack folder that already has a maintainer. The bar is whatever the stack maintainer sets, plus the universal rules below.

## Universal rules — apply to every contribution

These come from the core methodology and are not negotiable in this repo:

- Tone is plain. No marketing language, no hype, no aspirational claims.
- Every rule has a reason. If you can't explain why a rule is in `standing-orders.md`, it shouldn't be there.
- Real-world only. Patterns must come from projects that actually shipped, not from theory or documentation reading.
- No tool worship. Extensions describe stacks, not editors or AI assistants. If your extension only works because of a specific tool, it's not an extension.
- Apache 2.0. By submitting, you're contributing under that license.

## Bar for a new stack extension

To claim an open stack, your pull request must include all four files, populated:

**`README.md`** — what this stack covers, who you are, what production work this is grounded in. You don't have to share anything proprietary, but the reviewer needs to believe you've actually shipped in this stack.

**`standing-orders.md`** — the absolute rules for this stack. Every entry must include the rule itself and the failure mode that justified adding it. Rules without consequences are guidelines, and guidelines belong in `patterns.md`.

**`patterns.md`** — patterns you've used repeatedly that work. Each pattern needs a name, the situation it applies to, the implementation sketch, and what it replaces. "Just use library X" is not a pattern.

**`anti-patterns.md`** — things that look reasonable in this stack but consistently break. Each anti-pattern needs the trap, why it's tempting, what actually goes wrong, and the correct alternative.

The dotnet folder is the reference. Match its structure and depth. If your extension is half the length of dotnet's, it's probably not done yet.

## Review process for new stacks

1. Open a draft PR early. You don't have to wait until all four files are perfect — partial work is fine for review feedback.
2. The maintainers of fse-core will review for tone, structure, and consistency with the methodology.
3. At least one developer who works in that stack but is not you should review for technical accuracy. If we don't have one, we'll find one before merging.
4. After merge, you're listed as the maintainer in the stacks table in the root README.

## What disqualifies a new stack submission

- Patterns that don't reflect real production work
- Standing orders without consequences attached
- Tone that drifts into marketing, evangelism, or stack rivalry
- Copy-pasted content from other documentation
- A stack that doesn't actually need its own extension because the core already handles it cleanly

## Bar for improvements to existing stacks

- Typos, broken links, formatting fixes: open a PR directly
- Wording clarifications, small examples: open a PR directly, ping the maintainer for review
- New patterns, new anti-patterns, changes to standing orders: open an issue first, discuss with the maintainer, then PR
- Changes that would alter the meaning of a standing order: significant discussion required, fse-core maintainers will weigh in

## Maintainer responsibilities

If you become a stack maintainer:

- Respond to issues on your stack within a reasonable window (no hard SLA, but visible activity)
- Review pull requests against your folder
- Keep the four files in sync with each other
- Update content when the underlying stack changes in ways that affect FSE adoption

You are not on the hook forever. If you need to step down, open an issue and we'll find a successor.

## Code of Conduct

By participating, you agree to the [Code of Conduct](./CODE_OF_CONDUCT.md). Reports go to conduct@joticle.com.

## Questions

Open an issue using the methodology question template in fse-core if it's about how the methodology applies. Use the extension request or extension bug templates here if it's about this repo.
