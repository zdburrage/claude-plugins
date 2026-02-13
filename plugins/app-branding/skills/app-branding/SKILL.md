---
name: app-branding
description: Scaffold a standalone branding system into any React project
---

# App Branding Skill

Implement a standalone branding system into the current project. The system lets users run `npm run brand` to interactively configure company name, logo, accent color, and description — no Claude dependency at runtime.

## Trigger Conditions

Activate this skill when:

- User explicitly asks: "add the auto brander", "implement branding", "brand this for [company]", "set up branding"
- User is creating a demo app: "create a demo app", "help me build a demo application", "fire up a new demo", "scaffold a new demo"
- Any demo app creation context — branding is table stakes for demo apps and should be included automatically

When triggered during demo app creation, include branding as part of the setup without asking. For standalone invocation, proceed directly.

## Workflow

Execute these four phases in order. Report findings after Phase 1 before continuing.

### Phase 1: Detect

Read `package.json` and scan the project to determine:

1. **Framework**: Next.js (`next` in dependencies) / Vite (`vite`) / CRA (`react-scripts`) / plain React
2. **UI library**: Radix Themes (`@radix-ui/themes`) / Tailwind (`tailwindcss`) / shadcn (`@shadcn/ui` or `components/ui` dir) / none
3. **Env var prefix**: Based on framework:
   - Next.js → `NEXT_PUBLIC_`
   - Vite → `VITE_`
   - CRA → `REACT_APP_`
   - Other → empty string
4. **Source directory**: `src/` if it exists, otherwise root
5. **Existing .env file**: Check for `.env`, `.env.local`, `.env.example`
6. **TypeScript vs JavaScript**: Check for `tsconfig.json`

Report findings to the user in a brief summary before proceeding:

```
Detected: Next.js + Radix Themes (TypeScript)
Env prefix: NEXT_PUBLIC_
Source dir: src/
```

### Phase 2: Scaffold

Create three files. Use the reference documents in `references/` for exact implementation patterns.

#### 2a. `script/config.ts` (or `script/config.js` for JS projects)

Interactive branding wizard. Reference: `references/config-script-template.md`

Key behaviors:
- Runs standalone via `npx tsx script/config.ts` (no framework dependency)
- Prompts: LinkedIn URL (optional) → company name → logo URL → accent color → app description → write to .env
- LinkedIn fetch: scrapes `og:title` and `og:image` from company page, pre-fills name + logo
- Color picker: adapts to detected UI library (see below)
- Env writing: uses `setEnvVar()` helper that handles active, commented, and missing vars
- Creates `.env` if it doesn't exist (unlike se-demo-connect which requires .env.example)

**Color picker adaptation:**
- **Radix Themes**: Show 26 Radix color names with ANSI terminal preview. Reference: `references/radix-colors.md`
- **Tailwind/shadcn**: Show preset HSL values (slate, zinc, red, orange, green, blue, violet, pink). Accept any valid CSS color.
- **No UI library**: Accept any valid CSS color string (hex, rgb, hsl)

**Env vars written** (4 total, using detected prefix):
- `{PREFIX}APP_NAME` — company/app display name
- `{PREFIX}ACCENT_COLOR` — color value (Radix name, HSL, or hex depending on UI lib)
- `{PREFIX}LOGO_URL` — logo image URL
- `{PREFIX}APP_DESCRIPTION` — short app description

#### 2b. `script/reset-branding.ts` (or `.js`)

Simple script that resets the 4 branding env vars to empty strings. Same `setEnvVar()` helper pattern.

#### 2c. `lib/branding.ts` (or `src/lib/branding.ts` based on source dir)

Theme resolution layer. Reference: `references/theme-resolution.md`

Exports:
- `getBrandingConfig(): BrandingConfig` — reads env vars, returns typed config with safe defaults
- `BrandingConfig` type: `{ appName: string; accentColor: string; logoUrl: string | null; appDescription: string }`

The implementation varies by framework — see `references/theme-resolution.md` for exact patterns.

### Phase 3: Wire

Make minimal edits to existing project files. Do NOT create new components — only wire into what exists.

#### 3a. Layout / Theme Provider

Find the root layout file (e.g., `app/layout.tsx`, `src/App.tsx`, `src/main.tsx`).

- Import `getBrandingConfig` from the branding lib
- Call it at the top of the file: `const branding = getBrandingConfig()`
- Wire `accentColor`:
  - **Radix**: Pass to `<Theme accentColor={branding.accentColor}>`
  - **Tailwind/shadcn**: Inject as CSS variable via `style` prop or `globals.css`
  - **None**: Set `--accent-color` CSS custom property on `<html>` or `<body>`
- Wire `appName` into page metadata/title

#### 3b. Nav / Header Component (if one exists)

Search for an existing nav/header component. If found:
- Wire `logoUrl`: render `<img>` when set, fallback icon when not
- Wire `appName`: display in nav brand area

If no nav component exists, skip this step. Do NOT create a nav component.

#### 3c. package.json Scripts

Add two scripts:
```json
{
  "brand": "npx tsx script/config.ts",
  "brand:reset": "npx tsx script/reset-branding.ts"
}
```

#### 3d. Dependencies

Check if `tsx` is in `devDependencies`. If not, add it:
```bash
npm install --save-dev tsx
```

### Phase 4: Verify

1. Run the project build (`npm run build`, `next build`, `vite build`, etc.)
2. If the build fails, read the error and fix it
3. Report success with a summary:

```
Branding system installed:
- npm run brand      → interactive config wizard
- npm run brand:reset → reset to defaults
- Config: script/config.ts
- Theme:  lib/branding.ts

Run `npm run brand` to configure branding for a prospect.
```

## Constraints

- **Standalone runtime**: The config script must run without Claude. It's a normal Node.js script.
- **4 env vars only**: `APP_NAME`, `ACCENT_COLOR`, `LOGO_URL`, `APP_DESCRIPTION` (with appropriate prefix). No personas, no multi-app routing.
- **Minimal footprint**: Three new files + minimal edits to existing files. No new components, no new directories beyond `script/`.
- **No framework lock-in**: The branding lib abstracts framework differences. The config script is framework-agnostic.
- **Graceful defaults**: Everything works without running `npm run brand` — defaults to generic name, blue/default color, no logo.
- **Create .env if missing**: Unlike se-demo-connect, don't require `.env.example`. Create `.env` if it doesn't exist, append vars if file exists but vars are missing.
