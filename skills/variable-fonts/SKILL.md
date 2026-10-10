---
name: variable-fonts
description: Use when working with variable fonts — choosing between variable and static files, using weight/width/optical-size/slant axes, setting font-variation-settings vs standard properties, animating axes, or deciding whether a variable font actually saves bytes. Also use when a variable font renders wrong or ignores its axes.
---

# Variable fonts

One file containing a continuous design space instead of discrete instances.
Powerful and frequently misused — the most common error is reaching for
`font-variation-settings` when a standard CSS property already maps to the axis.

## 1. When to invoke

- Choosing between a variable font and a set of statics.
- Using weight, width, optical size, slant, or italic axes.
- Axes not responding, or rendering incorrectly.
- Animating or transitioning a typographic axis.
- Auditing whether a variable font is actually paying for itself.

## 2. Required context

- **Which axes** the font exposes, and their **ranges** (read the font, don't
  assume — `wght 100–900` is common but not universal).
- **How many instances** you'd otherwise ship.
- **Real byte comparison**: variable file vs. the statics it replaces.
- Whether you need **continuous** values or only named instances.
- Whether the design needs **optical sizing**.

## 3. Invariant principles

- **Use standard CSS properties, not `font-variation-settings`, wherever one
  exists.** Registered axes map to real properties:

  | Axis | Standard property |
  |---|---|
  | `wght` | `font-weight` |
  | `wdth` | `font-stretch` (or `font-width`) |
  | `opsz` | `font-optical-sizing` |
  | `ital` | `font-style: italic` |
  | `slnt` | `font-style: oblique <angle>` |

  `font-variation-settings` is a low-level override. Setting it replaces the
  whole list rather than merging individual axes, so a child that sets one axis
  loses the parent's other explicit axis settings. Explicit axis values override
  the corresponding high-level font properties. Prefer standard properties when
  they express the desired control; use the low-level property for custom axes
  or values unavailable through a standard property. For example, a deliberate
  numeric `opsz` override is valid: `font-optical-sizing` only accepts `auto` or
  `none`. Document the reason, check the font's range, and preserve any other
  explicit axes when changing the list.

- **Declare the axis range in `@font-face`.** `font-weight: 100 900` on the
  `@font-face` rule tells the browser the file covers that span; omit it and the
  browser may treat the font as a single weight and synthesize faux bold.
- **A variable font is not automatically smaller.** It carries the whole design
  space. Compare real bytes.
- **Only ship the axes you use.** Instancing/subsetting can strip unused axes.

## 4. Context-dependent heuristics

**Variable vs. static — the break-even.**

| Instances needed | Usually |
|---|---|
| 1–2 | Static wins |
| 3 | Roughly even — measure |
| 4+ | Variable usually wins |
| Continuous values or animation | Variable, necessarily |

Measure, don't assume. A variable file replacing two weights often *costs* bytes.

**Optical size (`opsz`) is the underused one.** It adjusts the design for the size
it's rendered at — more contrast and tighter spacing at display sizes, sturdier
forms and looser spacing at text sizes. This is what metal type did by cutting each
size separately. Enable with `font-optical-sizing: auto` (browsers apply it from
`font-size` automatically when the axis exists). Turn it off only deliberately.

**Weight is now continuous.** You can use 450 or 530. Resist it in a design system —
the value of a scale is exclusion. Pick your named steps and tokenize them.

**Grade (`GRAD`) — the axis most people don't know they need.** Grade changes
apparent weight **without changing advance widths**, so text gets visually heavier
or lighter while occupying exactly the same space. Nothing reflows.

Two uses:

- **Dark mode.** Light-on-dark text appears optically bloated — the light strokes
  bleed into the dark ground. Drop the grade slightly in dark mode and the text
  reads at the same visual weight as its light-mode counterpart. (In print the
  relationship inverts.) This is the correct fix for "our dark mode looks heavier",
  far better than switching to a lighter *weight*, which would reflow the line.
- **Interaction states.** Because nothing reflows, grade is safe for hover and
  active states in a way `font-weight` is not.

`GRAD` is a custom axis (Roboto Flex exposes it), so it is one of the legitimate
cases for `font-variation-settings` — there is no standard property for it.

**Parametric axes** go further still, exposing thin-stroke, thick-stroke, counter
width, and similar low-level parameters. Powerful for fine-tuning a face to a
specific context; easy to produce something the type designer would not recognise.
Use sparingly and compare against the designed instances. Custom axes are
arbitrary — a type designer can expose almost any parameter of the design this
way, down to contrast, x-height, serif shape, or the length of a single glyph's
tail (Rutter cites a Q's tail as a real example); the four-character tag is
whatever the foundry chose, so read the specimen rather than guessing at what a
custom axis controls.

**Multiplexed fonts** solve the same reflow problem a different way: every weight
occupies identical horizontal space by design, so switching weight never changes
width. See `ui-component-typography`.

**Animation.** Axes are animatable, which enables genuinely new effects. Constrain
it: animating `wght` reflows text if the width changes, and any motion must respect
`prefers-reduced-motion`.

```css
@media (prefers-reduced-motion: no-preference) {
  .headline { transition: font-variation-settings 200ms ease; }
}
```

**Fallbacks.** Variable font support is universal in current browsers, but the file
still has to load. The metric-matched-fallback technique in
`font-loading-and-performance` applies unchanged.

## 5. Failure patterns

| Pattern | Cause | Fix |
|---|---|---|
| Faux bold despite a variable font | `@font-face` didn't declare the range | `font-weight: 100 900` in `@font-face` |
| Child element loses width/optical settings | `font-variation-settings` replaces the whole list | Use standard properties |
| Variable font larger than the statics it replaced | Replaced too few instances | Measure; consider statics |
| Axis has no effect | Axis not present, or outside its range | Inspect the font's actual axes |
| Text reflows during a weight animation | Weight change alters advance widths | Animate a non-reflowing property, or reserve space |
| Display text looks flabby at 72px | `opsz` unused or disabled | `font-optical-sizing: auto` |
| Design drifts to arbitrary weights | Continuous axis, no tokens | Tokenize named steps |
| Dark mode text looks heavier than light mode | Optical bloat of light-on-dark | Lower `GRAD`, not `wght` — no reflow |
| Hover state shifts the layout | `font-weight` changes advance widths | `GRAD`, or a multiplexed face |
| Parametric tuning produced something odd | Low-level axes pushed past the design space | Compare against the designed instances |
| Motion sickness complaints | Axis animation, no reduced-motion guard | Wrap in the media query |

## 6. Evaluation procedure

1. Inspect the font's **actual axes and ranges** (browser devtools, `fonttools`, or
   the foundry spec). Don't assume.
2. Count the instances the design genuinely uses.
3. **Measure both**: variable file vs. sum of the statics, after subsetting.
4. Confirm `@font-face` declares each axis range.
5. Confirm standard properties are used for registered axes; `font-variation-settings`
   is used only where those properties cannot express the requested control
   (custom axes or a deliberate numeric `opsz` override, for example).
6. Check inheritance: set an axis on a parent, verify a child that changes one axis
   retains the rest.
7. If `opsz` exists, verify `font-optical-sizing: auto` and compare display vs. text
   rendering.
8. Confirm no faux bold/italic is being synthesized.
9. If animating, verify `prefers-reduced-motion` is respected and no reflow occurs.

## 7. Output format

```
Font: <name>
Axes available: <wght 100–900, wdth 75–125, opsz 14–72, …>
Axes used: <list> — via <standard property | font-variation-settings + reason>
@font-face ranges declared: <yes/no>
Instances replaced: <n> — variable <n>KB vs statics <n>KB → <win|loss>
Optical sizing: <auto|off> — because <reason>
Tokenized weights: <list>
Faux synthesis: <none|present>
Reduced-motion respected: <yes/n-a>
```

## 8. Examples

**Design system shipping Inter at 400/500/600/700 as four static files.**

> Four instances is past the break-even, and the variable file measures smaller
> than the four statics after subsetting, so switch. Declare
> `font-weight: 100 900` in `@font-face` so the browser knows the span and stops
> synthesizing. Use `font-weight` — not `font-variation-settings` — so the cascade
> and `font-synthesis` keep working. Tokenize exactly the four existing steps
> despite the axis now being continuous; the scale's value is that 530 is
> unavailable.

**Headline set in a family with an `opsz` axis, looking weak at 64px.**

> The optical size axis isn't engaged, so the display setting is using text-optimised
> forms — too little contrast and too much spacing for 64px. `font-optical-sizing:
> auto` lets the browser drive `opsz` from the rendered size; the headline picks up
> the higher-contrast, tighter-spaced cut it was designed to have. No markup change,
> no extra bytes.

## 9. Counterexamples

- ❌ Unnecessarily setting `font-variation-settings: "wght" 700` when
  `font-weight: 700` expresses the same intent. — Adds a low-level override and
  complicates inheritance of the other explicit axes.
- ❌ "Variable fonts are always smaller." — They carry the full design space; two
  weights are often cheaper as statics.
- ❌ Omitting `font-weight: 100 900` from `@font-face`. — Invites faux bold.
- ❌ "We have a continuous weight axis, so let designers pick any value." — Destroys
  the system's constraint.
- ❌ Animating `wght` on body text without checking reflow or reduced-motion.
- ❌ "It has an `opsz` axis so it handles sizes automatically." — Only if optical
  sizing is enabled and not overridden.

## 10. Source citations

- **Current standards take precedence here** — variable fonts postdate most of the
  corpus. MDN —
  [variable fonts guide](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_fonts/Variable_fonts_guide),
  [`font-variation-settings`](https://developer.mozilla.org/en-US/docs/Web/CSS/font-variation-settings),
  [`font-optical-sizing`](https://developer.mozilla.org/en-US/docs/Web/CSS/font-optical-sizing).
  [web.dev — optimize web fonts](https://web.dev/learn/performance/optimize-web-fonts).
- Stocks, *Universal Principles of Typography* (2024) — the most current book in the
  corpus on variable fonts. §29 multiple widths; §30 masters and interpolation
  (weight and width masters combined by vector addition); §31 multiplexed typefaces;
  §84–85 variable fonts and axes; **§86 grade** — "grade, unlike weight (except in
  multiplexed fonts), doesn't affect the overall width of the glyphs", and "on
  screens, light-on-dark text will appear slightly bloated, so, if possible, switch
  to a lighter grade when designing for Dark Mode. The opposite is true in print";
  §87 parametric axes; §88 the inheritance problem; §27 avoid faux styles.
- Brown, *Flexible Typesetting* — designing typographic systems that adapt rather
  than fixed instances; the conceptual case for continuous ranges.
- Rutter, *Web Typography* — OpenType foundations that variable fonts extend;
  custom axes as arbitrary designer-defined parameters (contrast, x-height, serif
  shape, individual glyph details) reachable only through
  `font-variation-settings`. **His browser-support and format-string details
  (`format('woff-variations')`, multiple repeated `@font-face` rules for
  fallback, an early-2018 viability estimate) are from 2017 and superseded —
  variable font support is now universal; don't carry those specifics forward.**
