# UI review checklist

Run each item on every affected page, at **1440×900** and **390×844**, in **light and dark** theme
when the site has both. Take a screenshot for anything that fails, and one "after" screenshot per
page per viewport regardless.

## Layout

- [ ] No elements overlap or cover each other (sticky headers, floating buttons, popovers, toasts,
      dropdowns, modals over content).
- [ ] No horizontal page scroll at 390 px wide (check `document.documentElement.scrollWidth >
      clientWidth`).
- [ ] Text is not cut off, clipped, or overflowing its box; long words / long numbers / long
      titles wrap or ellipsize deliberately.
- [ ] Images and SVGs load (no broken image icons), keep their aspect ratio, and are readable in
      both themes (no black-on-black / white-on-white).
- [ ] Spacing and alignment match neighboring, unchanged parts of the page.

## Interaction

- [ ] Every new or changed control can actually be clicked — not covered by another element
      (check with `document.elementFromPoint(x, y)` at the control's center if in doubt).
- [ ] Clicking does what the spec says; the resulting state is visible.
- [ ] Hover / focus / active / disabled states exist and are visible.
- [ ] Keyboard: Tab reaches the control, focus ring is visible, Enter / Space / Esc behave.
- [ ] Forms: validation messages appear next to the right field; submitting twice does not
      double-submit.

## States

- [ ] Loading state (throttle the network if needed).
- [ ] Empty state (no data).
- [ ] Error state (failed request) — message is shown, page does not go blank.
- [ ] Logged-out vs logged-in, ordinary user vs admin, if the change depends on who is viewing.

## Health

- [ ] No new errors or warnings in the browser console.
- [ ] No failed network requests caused by the change.
- [ ] Reloading the page keeps the state it should keep (URL params, saved settings).
