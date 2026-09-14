# acb-handsontable

A fork of Handsontable 6.2.2, carrying one behavioural change, consumed by `acb-app` as a
git dependency.

Part of the ACADS-BSG estate. `foundation/architecture-map.md` in the activation record has
the wider picture; this file is about working here.

## What this is

Handsontable is the data grid library behind the calculation grids in Hyena+ and CAMEL+.
This repository is upstream version 6.2.2 with a local fix applied and the build output
committed, so `acb-app` can install it straight from git.

Nearly everything here is upstream code that nobody at ACADS wrote. Two things are not, and
they are the whole reason the repository exists.

## The fork is one line

The entire behavioural difference between this and upstream 6.2.2 is in
`src/editors/autocompleteEditor.js`, inside `onBeforeKeyDown`:

```js
let timeOffset = 20;   // upstream 6.2.2 has 0
```

It delays the autocomplete suggestion refresh by 20 milliseconds after a printable key,
backspace, delete or insert. Everything else in the diff between the `v6.2.2` and
`v6.2.2-fix` tags is build output and deleted test bundles.

There is no commit message, issue or comment explaining which bug that fixed. If you are
about to change it, find out what it was for first, because the symptom it addressed will
come back and it will look like a new bug in the grid rather than a regression here.

The other local addition is `webpack.config.js`, a small shim that reads the build config
from `.config/<NODE_ENV>` and passes a test path pattern through. Upstream does not ship it.

## Why the built output is committed

`commonjs/`, `es/` and `dist/` are in git: roughly 800 files of build output. That is
deliberate rather than sloppy.

`acb-app` installs this as `git+https://github.com/…/acb-handsontable.git#v6.2.2-fix`. npm
does not run a build step when installing from a git URL, so `main` and `module` in
`package.json` have to point at files that are already there. Remove them and `acb-app`
stops building.

The consequence: **a source change is not live until the build output is regenerated and
committed alongside it.** Editing `src/` alone changes nothing for the consumer.

## How to build and test

```bash
npm install
npm run build          # regenerates commonjs/, es/, dist/ and the language files
npm run lint
npm test
```

Run the build after any change under `src/`, and commit what it produces. Check the diff
before committing: the build touches hundreds of files and it is easy to bury a real change
inside it.

## The gate, and what it actually checks

**Nothing runs automatically.** There is no CircleCI configuration here, `.travis.yml` is
upstream's and is dead, and the default branch carries no required status check.

The rules on `main` are a pull request with one approving review from someone other than the
author, no force pushes, no deletion. That review is the entire gate.

Note the branch is `main`, not `master`. This is the only repository in the estate where
that is true, and the organisation ruleset targets the default branch rather than a literal
name, so it is covered.

## Golden rules

These four apply everywhere in the estate.

**The gate is sacred.** A change reaches `main` through a pull request with one approving
review from someone other than the author. There is no bypass. Here that review is the only
check that happens at all.

**The calculation core is out of bounds.** `acb-backend-exe` and `acb-windows-controller`
are not yours to change. Nothing here touches them, though the grids this library draws are
where calculation inputs are typed.

**Customer data never leaves its class.** See the data rules below. This repository is
public, which changes what that sentence means here.

**Production is released by a git tag.** This library ships by tag too: `acb-app` pins
`#v6.2.2-fix`, so a change reaches production only when a new tag exists and the consumer is
repointed at it.

## Data rules

This repository holds no customer data and no credentials, and it never should. What makes
it different from every other repository in the estate is simpler and more dangerous.

**It is public.** Anyone on the internet can read it, including its history.

| Class | What it is in this repository | What an agent may do |
| :-- | :-- | :-- |
| **Open** | Everything currently here: upstream Handsontable, the one line fix, the build output | Read and change through the gate |
| **Internal** | Nothing today, and nothing should be added | Do not bring Internal material into a public repository |
| **Confidential** | Nothing. No customer data belongs here, including in a test fixture or a demo grid | Never add |
| **Restricted** | Nothing. No secret, token or key belongs here under any circumstance | Never add |

Two consequences follow, and both are worse here than elsewhere.

**A secret committed here is public immediately.** In a private repository a mistaken commit
is an incident to clean up. Here it is a disclosure, and removing it later does not undo it.

**Secret scanning and push protection are currently off on this repository**, checked on
14 September 2026. They are on everywhere else in the estate. This is the one repository
where a committed credential would be readable by anyone and nothing would raise an alert,
which is the worst of both. Until that changes, the only thing standing between a mistake
and a disclosure is whoever reviews the pull request.

**A test fixture with real data is published.** Grid fixtures are the natural place to paste
a real spreadsheet while reproducing a bug. Do not. Invent the numbers.

### In practice

- **Change freely:** nothing needs changing here routinely. This is a pinned dependency.
- **Change with care:** `src/editors/autocompleteEditor.js`, because of the fork line above,
  and anything under `.config/` or `webpack.config.js` that affects the build.
- **Change and rebuild:** any change under `src/` needs `npm run build` and the regenerated
  output committed with it, or the consumer sees nothing.
- **Never add:** secrets, tokens, keys, customer data, or anything Internal. The repository
  is public.
- **Never read:** `.env` and its variants, key and certificate files, database dumps. None
  exist here, and the deny rules keep it that way.
- **Never touch:** the calculation core, and infrastructure state.
- **At the line, stop and ask.** The exception route is a recorded tool approval, not a
  workaround.

## Sharp edges

- **The estate uses two different Handsontables.** `acb-app` installs this fork by git URL.
  `acb-camel` installs `"handsontable": "^6.2.2"` from the public npm registry, which is
  upstream without the fork. So the 20 millisecond delay exists in Hyena+ and not in CAMEL+,
  and a grid bug reported against one may not reproduce in the other.
- **The git dependency in `acb-app` points at the old organisation.**
  `git+https://github.com/focallabs/acb-handsontable.git#v6.2.2-fix` resolves today only
  through GitHub's redirect after the transfer. It is a build time dependency, so if that
  redirect is ever removed, every build of `acb-app` fails. Worth repointing at
  `ACADS-BSG-PTY-LTD` as its own change.
- **Upstream is many major versions ahead.** Handsontable 6.2.2 dates from 2018 and is no
  longer supported. An upgrade means reapplying the fork line and rebuilding, and it is a
  project rather than a chore.
- **The package still calls itself `handsontable` at version `6.2.2`.** It is not that
  package. Anyone reading `package.json` alone would conclude this is upstream.
- **`.travis.yml` and `.github/` are upstream's** and neither runs anything for ACADS.
- **The licence is MIT**, inherited from upstream, which is why forking and republishing it
  is permitted at all.

## Reference

- `src/editors/autocompleteEditor.js`: the one line that makes this a fork
- `webpack.config.js`: the local build shim, not upstream
- `commonjs/`, `es/`, `dist/`: committed build output the consumer loads
- `package.json`: `main` and `module` point into the committed output
- Tags `v6.2.2` and `v6.2.2-fix`: the diff between them is the whole local change

## Learned rules

Corrections that would otherwise be made twice. Add one whenever something goes wrong in a
way that would repeat: what happened, and the rule that prevents it.

```
### <short name>
What went wrong: <one sentence>
Rule: <what to do instead>
Added: <date>
```

Seeded empty on purpose. A file that never grows is a file nobody is reading.
