# Music Assistant frontend — VolMod

A fork of the [Music Assistant frontend](https://github.com/music-assistant/frontend) that makes the
behaviour of the player volume slider configurable.

Upstream ships a single, fixed volume interaction: tap or drag anywhere on the volume bar and the volume
jumps to that position, in 2% increments, with haptic feedback on touch devices. That works well on a
desktop with a mouse, but on a phone — where the volume bar is a thin strip inside the now-playing
sheet — a slightly misplaced thumb can send a speaker from a background listening level to full volume
in a single tap. There is currently no way to change this.

This fork does not replace that behaviour. It adds an alternative alongside it and exposes two
independent settings so each user can pick the interaction that suits their devices.

## What it adds

Two new settings under **Settings → Frontend → Volume control**:

| Setting                     | Options                                                 | Default  |
| --------------------------- | ------------------------------------------------------- | -------- |
| **Volume slider behaviour** | Absolute (jump to position) / Relative (drag to adjust) | Absolute |
| **Volume haptic feedback**  | On / Off                                                | On       |

Every default reproduces current upstream behaviour exactly, so an existing install that never opens the
settings page sees no change whatsoever.

### Volume slider behaviour

- **Absolute** — unchanged upstream behaviour. The tap or drag position maps directly onto the 0–100
  range, so the volume follows wherever you put your finger or pointer.
- **Relative** — the volume adjusts by the _distance_ you drag, starting from the volume the drag began
  at, at half the pointer's speed. Where you first touch the bar is irrelevant; only the movement counts.
  This makes the bar behave like a jog control rather than a position control: a mistimed tap can no
  longer jump the volume, and fine adjustments are practical even on a narrow mobile slider. Relative
  mode also enables click-and-drag adjustment with a mouse, which reads as a natural extension of the
  same gesture on desktop.

In relative mode a short tap (under 5 px of travel) is still treated as a tap, so the group-player
expand action on the volume row keeps working.

### Volume haptic feedback

Turns off the vibration that fires while dragging the volume slider. Upstream vibrates on touch-start
and again on every step crossed, which some users find noisy — a single drag can produce dozens of
pulses. The toggle gates every vibration from the volume control at a single point and has no effect
on devices without a vibration motor.

### On volume step size

An earlier version of this fork also exposed the volume step size as a frontend preference. That is
gone: the step is server-owned. Music Assistant's `players` core config has a `volume_step` entry
(server PR [#5571](https://github.com/music-assistant/server/pull/5571)) which drives the volume
up/down commands for every client, and it already renders under **Settings → System → Players**. The
step used while dragging the slider is a different quantity and stays at upstream's 2%.

## Why these are settings rather than a change in default

Volume interaction is a matter of hardware and habit, not correctness. A mouse user on a wide desktop
slider is well served by absolute positioning; a phone user reaching for a kitchen speaker is usually
better served by relative dragging. The same is true of haptics. Rather than trading one group's
experience for another's, both are exposed as preferences with upstream's current behaviour as the
default.

The two settings are also deliberately orthogonal — relative mode implies nothing about haptics — so
users can combine them freely instead of choosing between bundled presets.

## Implementation notes

- The changes are confined to `src/layouts/default/PlayerOSD/PlayerVolume.vue`,
  `src/views/settings/FrontendConfig.vue` and `src/translations/en.json`. No shared component,
  composable or API surface is modified.
- Settings are stored as **per-user server preferences**, not per-device `localStorage`, so a user's
  volume preferences follow them across every browser and installed PWA signed in to the same account.
  They are read through the existing `useUserPreferences()` composable as reactive computed refs.
- Both interaction paths coexist in the component. In absolute mode every relative-mode handler returns
  early and the stock slider is untouched, including its keyboard focus behaviour on desktop.
- Because both preferences are read reactively, saving them takes effect immediately, so the settings
  page skips its full-page reload for them.
- New settings are surfaced through the existing frontend-owned config-entry mechanism, with
  `en.json` labels, descriptions and option titles; other locales fall back to English until translated.

## Upstream compatibility

The fork tracks upstream directly:

- `main` is a pristine mirror of `music-assistant/frontend`.
- `volmod` carries the volume change as a single commit on top of a few fork-only commits (this notice,
  the screenshot, a `.gitignore` entry for local scratch files), rebased onto upstream `main`. The
  volume commit is the one proposed upstream, so `volmod` and the pull request stay in agreement.

Type checking (`vue-tsc`), linting (`oxlint`, `eslint`), formatting (`prettier`) and the full unit test
suite pass unmodified.

## Building

The build is unchanged from upstream:

```sh
pnpm install
pnpm build
```

`pnpm build` writes an importable Python package to `./music_assistant_frontend/`, which is what a
Music Assistant server serves as its UI.
