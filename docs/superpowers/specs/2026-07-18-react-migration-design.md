# Migrate portfolio site to React

## Why

The site is a single static `docs/index.html` (482 lines) plus `docs/style.css` (715
lines), with a chunk of vanilla JS at the bottom of the HTML handling the typed-text
effect, scroll-spy nav highlighting, and scroll-triggered reveal animations. The HTML
also repeats near-identical markup nine times for skill bars, four times for project
cards, three times for service cards, and three times for contact cards. Moving to
React componentizes the page along its existing section boundaries and turns the
repeated markup into data-driven components.

## Scope

In scope: rebuilding the current site's markup, styling, and behavior as a React +
TypeScript app. Out of scope: any visual redesign, new content, or new features — this
is a like-for-like architecture change.

## Repo layout

The React app is scaffolded at the repo root with Vite (`react-ts` template):

```
/
  index.html          # Vite entry, replaces docs/index.html
  package.json
  vite.config.ts
  tsconfig.json
  netlify.toml         # build = "npm run build", publish = "dist"
  public/
    profile.png         # renamed from "Untitled design.png" (About section photo)
    hero.png             # renamed from "Untitled design (2).png" (Home section image)
  src/
    main.tsx
    App.tsx
    App.css              # ported from docs/style.css, imported once in main.tsx
    components/
      Header.tsx
      Home.tsx
      About.tsx
      Skills.tsx
      Services.tsx
      Projects.tsx
      Contact.tsx
      SkillBar.tsx
      RadialBar.tsx
      ServiceCard.tsx
      ProjectCard.tsx
      ContactCard.tsx
    data/
      skills.ts           # technical + professional skill data
      services.ts
      projects.ts
      contacts.ts
    hooks/
      useScrollSpy.ts
      useSectionReveal.ts
    App.test.tsx / component .test.tsx files colocated per component
```

`docs/index.html`, `docs/style.css`, `docs/temp`, and the two original images are
removed once the new app is verified working. `docs/superpowers/` (specs/plans) is
untouched — it's not part of the deployed site.

## Components and data flow

Each of the seven page sections becomes a component, matching the current HTML
section boundaries 1:1: `Header` (logo + nav), `Home`, `About`, `Skills`, `Services`,
`Projects`, `Contact`. `App.tsx` renders them in order, same as the current document.

Repeated markup becomes a typed data array rendered through a small presentational
component:

- `data/skills.ts` exports `technicalSkills: { name, colorHex, iconClass, percent }[]`
  and `professionalSkills: { name, percent }[]`. `Skills.tsx` maps these through
  `SkillBar` and `RadialBar`.
- `data/services.ts` and `data/projects.ts` and `data/contacts.ts` follow the same
  pattern, feeding `ServiceCard`, `ProjectCard`, `ContactCard`.

This removes the copy-pasted block-per-item structure in the current HTML — adding a
skill or project becomes a one-line data entry instead of a duplicated markup block.
No further abstraction (no generic `Card` primitive, no design-system layer) since
nothing else repeats enough to justify it.

## Interaction and animation

- `useSectionReveal(ref)`: a hook wrapping `IntersectionObserver` (threshold 0.3).
  Runs the section's reveal animation once when it scrolls into view (mirrors the
  current one-shot observer-per-section behavior) and returns a `replay()` callback,
  wired to the section's `onClick`, matching today's click-to-replay behavior on the
  skills/services/projects/contact boxes.
- `useScrollSpy(sectionIds)`: mirrors the current scroll-listener logic, tracking
  which section is currently in view and returning its id so `Header` can apply the
  `active` class to the matching nav link. Clicking a nav link still does a smooth
  `scrollIntoView` and sets the active link immediately, same as today.
- The typed.js effect moves into a `useEffect` in `Home.tsx`: constructs `new
  Typed(...)` on mount against a ref (not a class selector, since refs are the React
  way to target a DOM node) and calls `.destroy()` on unmount.

## Dependencies

- `typed.js` becomes an npm dependency (already used today, just via CDN) —
  `import Typed from 'typed.js'`.
- Google Fonts and boxicons stay as CDN `<link>` tags in `index.html`'s `<head>` —
  they're icon/font stylesheets, not JS, so there's no reason to pull them into the
  npm dependency graph.
- No new runtime dependency beyond `typed.js`, `react`, and `react-dom`.

## Styling

`docs/style.css` is ported to `src/App.css` largely unchanged and imported once in
`main.tsx`, so all existing class names and selectors keep working against the new
JSX structure. No CSS Modules split — this keeps the diff focused on structure, not a
rewrite of every selector, and carries zero visual-regression risk since the
stylesheet itself doesn't change.

## Testing

Vitest + React Testing Library. One smoke test per section component asserting it
renders its expected heading/content (e.g. `Projects` renders all four project
titles and their GitHub links, `Contact` renders phone/email/LinkedIn entries), plus
a test for `useScrollSpy`'s active-section selection logic. No animation/visual
testing — jsdom doesn't do real layout or `IntersectionObserver` timing, so
`useSectionReveal` is exercised only insofar as component smoke tests mount it
without throwing.

## Deploy

`netlify.toml` is added with `command = "npm run build"` and `publish = "dist"`.
There's no `netlify.toml` in the repo today, so the Netlify site is presumably
configured via its dashboard with publish directory `docs` — the user should confirm
the dashboard settings after this change (a committed `netlify.toml` normally
overrides dashboard build settings, but this needs verifying against the actual
Netlify project, which isn't accessible from this repo).

## Migration order

1. Scaffold Vite app, dependencies, `netlify.toml`.
2. Port `style.css` to `App.css`, move + rename images into `public/`.
3. Build data files, then components bottom-up (leaf cards/bars first, then
   sections, then `App.tsx`), porting one HTML section at a time.
4. Add hooks (`useScrollSpy`, `useSectionReveal`), wire into `Header` and the
   animated sections.
5. Write component + hook tests.
6. Run `npm run build` and `npm run dev`, visually compare against the current
   `docs/index.html` in a browser.
7. Remove `docs/index.html`, `docs/style.css`, `docs/temp`, and the two original
   images.
