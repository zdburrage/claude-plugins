# Radix Colors Reference

26 Radix Theme accent color names with ANSI 256 terminal preview codes. Used by the config script color picker when the project uses `@radix-ui/themes`.

## Color Names

```typescript
const RADIX_COLORS = [
  "amber", "blue", "bronze", "brown", "crimson", "cyan",
  "gold", "grass", "gray", "green", "indigo", "iris",
  "jade", "lime", "mint", "orange", "pink", "plum",
  "purple", "red", "ruby", "sky", "teal", "tomato",
  "violet", "yellow",
] as const;

type RadixAccentColor = (typeof RADIX_COLORS)[number];
```

## ANSI 256 Terminal Preview Codes

Map each color to an approximate ANSI 256 code for terminal display:

```typescript
const COLOR_ANSI: Record<string, string> = {
  amber:   "\x1b[38;5;214m",
  blue:    "\x1b[38;5;33m",
  bronze:  "\x1b[38;5;137m",
  brown:   "\x1b[38;5;130m",
  crimson: "\x1b[38;5;197m",
  cyan:    "\x1b[38;5;37m",
  gold:    "\x1b[38;5;178m",
  grass:   "\x1b[38;5;34m",
  gray:    "\x1b[38;5;245m",
  green:   "\x1b[38;5;28m",
  indigo:  "\x1b[38;5;62m",
  iris:    "\x1b[38;5;99m",
  jade:    "\x1b[38;5;36m",
  lime:    "\x1b[38;5;154m",
  mint:    "\x1b[38;5;121m",
  orange:  "\x1b[38;5;208m",
  pink:    "\x1b[38;5;205m",
  plum:    "\x1b[38;5;133m",
  purple:  "\x1b[38;5;129m",
  red:     "\x1b[38;5;196m",
  ruby:    "\x1b[38;5;161m",
  sky:     "\x1b[38;5;117m",
  teal:    "\x1b[38;5;30m",
  tomato:  "\x1b[38;5;202m",
  violet:  "\x1b[38;5;135m",
  yellow:  "\x1b[38;5;220m",
};
const RESET = "\x1b[0m";
```

## Grid Display Pattern

Show colors in a 4-column grid with number + swatch + name:

```typescript
const colsPerRow = 4;
console.log("\nAccent colors:");
for (let i = 0; i < RADIX_COLORS.length; i += colsPerRow) {
  const row = RADIX_COLORS.slice(i, i + colsPerRow)
    .map((c, j) => {
      const num = String(i + j + 1).padStart(2, " ");
      const ansi = COLOR_ANSI[c] || "";
      return `  ${num}) ${ansi}██${RESET} ${c.padEnd(10)}`;
    })
    .join("");
  console.log(row);
}
```

Output looks like:

```
Accent colors:
   1) ██ amber       2) ██ blue        3) ██ bronze      4) ██ brown
   5) ██ crimson     6) ██ cyan        7) ██ gold        8) ██ grass
   9) ██ gray       10) ██ green      11) ██ indigo     12) ██ iris
  13) ██ jade       14) ██ lime       15) ██ mint       16) ██ orange
  17) ██ pink       18) ██ plum       19) ██ purple     20) ██ red
  21) ██ ruby       22) ██ sky        23) ██ teal       24) ██ tomato
  25) ██ violet     26) ██ yellow
```

## Input Handling

Accept either a number (1-26) or a color name:

```typescript
const colorInput = await ask(
  rl,
  `\nPick a color [1-${RADIX_COLORS.length}] or name (default: blue): `
);
let accentColor = "blue";
const colorNum = parseInt(colorInput, 10);
if (colorNum >= 1 && colorNum <= RADIX_COLORS.length) {
  accentColor = RADIX_COLORS[colorNum - 1];
} else if (
  colorInput.trim() &&
  RADIX_COLORS.includes(colorInput.trim().toLowerCase() as RadixAccentColor)
) {
  accentColor = colorInput.trim().toLowerCase();
}
console.log(
  `  Selected: ${COLOR_ANSI[accentColor]}██${RESET} ${accentColor}`
);
```

## Validation in Theme Resolution

Use the same array for runtime validation in `lib/branding.ts`:

```typescript
export const radixAccentColors = [
  "amber", "blue", "bronze", "brown", "crimson", "cyan", "gold", "grass",
  "gray", "green", "indigo", "iris", "jade", "lime", "mint", "orange",
  "pink", "plum", "purple", "red", "ruby", "sky", "teal", "tomato",
  "violet", "yellow",
] as const;

export type RadixAccentColor = (typeof radixAccentColors)[number];

// In getBrandingConfig():
const rawColor = process.env.NEXT_PUBLIC_ACCENT_COLOR || "blue";
const accentColor: RadixAccentColor = radixAccentColors.includes(
  rawColor as RadixAccentColor
)
  ? (rawColor as RadixAccentColor)
  : "blue";
```
