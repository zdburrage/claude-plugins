# Theme Resolution Patterns

Per-framework patterns for `lib/branding.ts` (or `src/lib/branding.ts`). The branding lib exports a typed `getBrandingConfig()` function and a `BrandingConfig` type.

## Shared Type

All variants export this type:

```typescript
export interface BrandingConfig {
  appName: string;
  accentColor: string;
  logoUrl: string | null;
  appDescription: string;
}
```

## Next.js + Radix Themes

Env vars accessed via `process.env.NEXT_PUBLIC_*` (inlined at build time by Next.js).

Color is validated against the Radix color union type.

```typescript
const radixAccentColors = [
  "amber", "blue", "bronze", "brown", "crimson", "cyan", "gold", "grass",
  "gray", "green", "indigo", "iris", "jade", "lime", "mint", "orange",
  "pink", "plum", "purple", "red", "ruby", "sky", "teal", "tomato",
  "violet", "yellow",
] as const;

type RadixAccentColor = (typeof radixAccentColors)[number];

export interface BrandingConfig {
  appName: string;
  accentColor: RadixAccentColor;
  logoUrl: string | null;
  appDescription: string;
}

export function getBrandingConfig(): BrandingConfig {
  const rawColor = process.env.NEXT_PUBLIC_ACCENT_COLOR || "blue";
  const accentColor: RadixAccentColor = radixAccentColors.includes(
    rawColor as RadixAccentColor
  )
    ? (rawColor as RadixAccentColor)
    : "blue";

  return {
    appName: process.env.NEXT_PUBLIC_APP_NAME || "My App",
    accentColor,
    logoUrl: process.env.NEXT_PUBLIC_LOGO_URL || null,
    appDescription: process.env.NEXT_PUBLIC_APP_DESCRIPTION || "",
  };
}
```

### Wiring into layout

```typescript
// app/layout.tsx
import { Theme } from "@radix-ui/themes";
import { getBrandingConfig } from "@/lib/branding";

const branding = getBrandingConfig();

export const metadata = {
  title: branding.appName,
  description: branding.appDescription,
};

// In the JSX:
<Theme accentColor={branding.accentColor}>
  {children}
</Theme>
```

## Next.js + Tailwind / shadcn

Color stored as HSL string. For shadcn, override the `--primary` CSS variable.

```typescript
export interface BrandingConfig {
  appName: string;
  accentColor: string;
  logoUrl: string | null;
  appDescription: string;
}

export function getBrandingConfig(): BrandingConfig {
  return {
    appName: process.env.NEXT_PUBLIC_APP_NAME || "My App",
    accentColor: process.env.NEXT_PUBLIC_ACCENT_COLOR || "",
    logoUrl: process.env.NEXT_PUBLIC_LOGO_URL || null,
    appDescription: process.env.NEXT_PUBLIC_APP_DESCRIPTION || "",
  };
}
```

### Wiring into layout

For shadcn projects, inject as CSS variable override on `<body>` or in a `<style>` tag:

```typescript
// app/layout.tsx
import { getBrandingConfig } from "@/lib/branding";

const branding = getBrandingConfig();

export const metadata = {
  title: branding.appName,
  description: branding.appDescription,
};

// In the JSX — only inject style if accent is set:
<body style={branding.accentColor ? { "--primary": branding.accentColor } as React.CSSProperties : undefined}>
  {children}
</body>
```

For plain Tailwind (no shadcn), you can inject `--accent-color` and reference it in Tailwind config or component styles.

## Vite + React

Env vars accessed via `import.meta.env.VITE_*`.

```typescript
export interface BrandingConfig {
  appName: string;
  accentColor: string;
  logoUrl: string | null;
  appDescription: string;
}

export function getBrandingConfig(): BrandingConfig {
  return {
    appName: import.meta.env.VITE_APP_NAME || "My App",
    accentColor: import.meta.env.VITE_ACCENT_COLOR || "",
    logoUrl: import.meta.env.VITE_LOGO_URL || null,
    appDescription: import.meta.env.VITE_APP_DESCRIPTION || "",
  };
}
```

### Wiring into App

Vite apps typically don't have a `layout.tsx` — wire into the root `App.tsx` or `main.tsx`:

```typescript
// src/App.tsx
import { getBrandingConfig } from "./lib/branding";

const branding = getBrandingConfig();

// Set document title
document.title = branding.appName;

// Inject CSS custom property for accent color
if (branding.accentColor) {
  document.documentElement.style.setProperty("--accent-color", branding.accentColor);
}
```

If using Radix with Vite, use the same `<Theme accentColor={...}>` pattern as Next.js.

## CRA (Create React App)

Env vars accessed via `process.env.REACT_APP_*`.

```typescript
export interface BrandingConfig {
  appName: string;
  accentColor: string;
  logoUrl: string | null;
  appDescription: string;
}

export function getBrandingConfig(): BrandingConfig {
  return {
    appName: process.env.REACT_APP_APP_NAME || "My App",
    accentColor: process.env.REACT_APP_ACCENT_COLOR || "",
    logoUrl: process.env.REACT_APP_LOGO_URL || null,
    appDescription: process.env.REACT_APP_APP_DESCRIPTION || "",
  };
}
```

## Fallback (No Framework Detected)

Plain `process.env.*` with CSS custom properties:

```typescript
export interface BrandingConfig {
  appName: string;
  accentColor: string;
  logoUrl: string | null;
  appDescription: string;
}

export function getBrandingConfig(): BrandingConfig {
  return {
    appName: process.env.APP_NAME || "My App",
    accentColor: process.env.ACCENT_COLOR || "",
    logoUrl: process.env.LOGO_URL || null,
    appDescription: process.env.APP_DESCRIPTION || "",
  };
}
```

## Nav / Header Wiring Pattern

When an existing nav/header component is found, wire the logo and app name:

```typescript
// In existing nav component — add these lines:
import { getBrandingConfig } from "@/lib/branding"; // adjust path

const branding = getBrandingConfig();

// Replace or augment the brand area:
{branding.logoUrl ? (
  <img src={branding.logoUrl} alt={branding.appName} style={{ height: 32 }} />
) : (
  <ExistingIcon /> // keep whatever icon was already there
)}
<span>{branding.appName}</span>
```

Do NOT create a nav component if one doesn't exist. Only wire into existing components.
