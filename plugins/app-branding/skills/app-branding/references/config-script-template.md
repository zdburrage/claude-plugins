# Config Script Template

Reference implementation for `script/config.ts`. Adapt the env var prefix and color picker based on Phase 1 detection results.

## Core Helpers

These helpers are used by both the config and reset scripts. Include them in each file (no shared module needed — keeps scripts standalone).

### setEnvVar

Handles three cases: active var, commented var, and missing var.

```typescript
function setEnvVar(content: string, key: string, value: string): string {
  const activeRegex = new RegExp(`^${key}=.*$`, "m");
  const commentedRegex = new RegExp(`^#\\s*${key}=.*$`, "m");

  if (activeRegex.test(content)) {
    return content.replace(activeRegex, `${key}=${value}`);
  }
  if (commentedRegex.test(content)) {
    return content.replace(commentedRegex, `${key}=${value}`);
  }
  return content.trimEnd() + `\n${key}=${value}\n`;
}
```

### HTML Entity Decoding

For LinkedIn OG tag content:

```typescript
function decodeHtmlEntities(s: string): string {
  return s
    .replace(/&amp;/g, "&")
    .replace(/&lt;/g, "<")
    .replace(/&gt;/g, ">")
    .replace(/&quot;/g, '"')
    .replace(/&#39;/g, "'");
}
```

### LinkedIn OG Fetch

```typescript
async function fetchLinkedInMeta(url: string) {
  try {
    const res = await fetch(url, {
      headers: {
        "User-Agent":
          "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36",
      },
      redirect: "follow",
    });
    const html = await res.text();

    const rawTitle = html.match(
      /<meta[^>]*property="og:title"[^>]*content="([^"]*)"[^>]*>/i
    )?.[1];
    const rawImage = html.match(
      /<meta[^>]*property="og:image"[^>]*content="([^"]*)"[^>]*>/i
    )?.[1];

    const ogTitle = rawTitle ? decodeHtmlEntities(rawTitle) : undefined;
    const ogImage = rawImage ? decodeHtmlEntities(rawImage) : undefined;

    return { title: ogTitle, image: ogImage };
  } catch {
    console.log("  Could not fetch page. Continuing with manual input.");
    return {};
  }
}
```

### Readline Helpers

```typescript
import * as readline from "readline";

function createRl() {
  return readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });
}

function ask(rl: readline.Interface, question: string): Promise<string> {
  return new Promise((resolve) => rl.question(question, resolve));
}
```

### ANSI Formatting

```typescript
const RESET = "\x1b[0m";
const DIM = "\x1b[2m";
const BOLD = "\x1b[1m";
```

## Full Script Structure

The config script follows this interactive flow:

```typescript
#!/usr/bin/env npx tsx

import * as fs from "fs";
import * as path from "path";
import * as readline from "readline";

// ── Constants ────────────────────────────────────────────────────────
// ANSI codes, color definitions (adapt per UI library — see radix-colors.md)
// ENV_PREFIX set based on detected framework

const ENV_PREFIX = "NEXT_PUBLIC_"; // ← adapt per framework
const ENV_PATH = path.resolve(process.cwd(), ".env");

// ── Helpers ──────────────────────────────────────────────────────────
// setEnvVar, decodeHtmlEntities, fetchLinkedInMeta, createRl, ask
// (paste from above)

// ── .env file handling ───────────────────────────────────────────────

function readOrCreateEnv(): string {
  if (!fs.existsSync(ENV_PATH)) {
    fs.writeFileSync(ENV_PATH, "# App branding configuration\n");
    console.log("  Created .env file");
  }
  return fs.readFileSync(ENV_PATH, "utf-8");
}

// ── Main ─────────────────────────────────────────────────────────────

async function main() {
  const rl = createRl();
  console.log(`\n${BOLD}🎨 App Branding Configuration${RESET}\n`);

  let appName = "";
  let logoUrl = "";

  // Step 1: LinkedIn URL (optional)
  const linkedInUrl = await ask(
    rl,
    "LinkedIn company URL (optional, press Enter to skip): "
  );

  if (linkedInUrl.trim()) {
    console.log("  Fetching company info...");
    const meta = await fetchLinkedInMeta(linkedInUrl.trim());
    if (meta.title) {
      appName = meta.title.replace(/ \|.*$/, "").trim();
      console.log(`  Found: ${appName}`);
    }
    if (meta.image) {
      logoUrl = meta.image;
      console.log(`  Logo: ${logoUrl.slice(0, 60)}...`);
    }
  }

  // Step 2: Company / App name
  const namePrompt = appName
    ? `App name [${appName}]: `
    : "App name: ";
  const nameInput = await ask(rl, namePrompt);
  if (nameInput.trim()) appName = nameInput.trim();

  // Step 3: Logo URL
  const logoPrompt = logoUrl
    ? `Logo URL [${logoUrl.slice(0, 50)}...]: `
    : "Logo URL (optional, press Enter to skip): ";
  const logoInput = await ask(rl, logoPrompt);
  if (logoInput.trim()) logoUrl = logoInput.trim();

  // Step 4: Accent color
  // ← Insert color picker here (Radix grid / Tailwind presets / freeform)
  // See radix-colors.md or adapt for Tailwind
  const accentColor = "blue"; // ← replace with picker result

  // Step 5: App description
  const descInput = await ask(rl, "App description (optional): ");
  const appDescription = descInput.trim();

  // Step 6: Write to .env
  let env = readOrCreateEnv();
  env = setEnvVar(env, `${ENV_PREFIX}APP_NAME`, appName);
  env = setEnvVar(env, `${ENV_PREFIX}ACCENT_COLOR`, accentColor);
  env = setEnvVar(env, `${ENV_PREFIX}LOGO_URL`, logoUrl);
  env = setEnvVar(env, `${ENV_PREFIX}APP_DESCRIPTION`, appDescription);
  fs.writeFileSync(ENV_PATH, env);

  // Summary
  console.log(`\n${BOLD}✅ Branding configured${RESET}\n`);
  console.log(`  Name:   ${appName || DIM + "(not set)" + RESET}`);
  console.log(`  Color:  ${accentColor}`);
  console.log(`  Logo:   ${logoUrl || DIM + "(not set)" + RESET}`);
  console.log(`  Desc:   ${appDescription || DIM + "(not set)" + RESET}`);
  console.log(`\n  Restart the dev server to see changes.\n`);

  rl.close();
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

## Reset Script Structure

```typescript
#!/usr/bin/env npx tsx

import * as fs from "fs";
import * as path from "path";

const ENV_PREFIX = "NEXT_PUBLIC_"; // ← adapt per framework
const ENV_PATH = path.resolve(process.cwd(), ".env");

function setEnvVar(content: string, key: string, value: string): string {
  const regex = new RegExp(`^${key}=.*$`, "m");
  if (regex.test(content)) {
    return content.replace(regex, `${key}=${value}`);
  }
  return content.trimEnd() + `\n${key}=${value}\n`;
}

if (!fs.existsSync(ENV_PATH)) {
  console.log("No .env file found. Nothing to reset.");
  process.exit(0);
}

let env = fs.readFileSync(ENV_PATH, "utf-8");
env = setEnvVar(env, `${ENV_PREFIX}APP_NAME`, "");
env = setEnvVar(env, `${ENV_PREFIX}ACCENT_COLOR`, "");
env = setEnvVar(env, `${ENV_PREFIX}LOGO_URL`, "");
env = setEnvVar(env, `${ENV_PREFIX}APP_DESCRIPTION`, "");
fs.writeFileSync(ENV_PATH, env);

console.log("\nBranding reset to defaults.");
console.log("Restart the dev server to see changes.\n");
```

## Adaptation Notes

When generating the actual script for a project:

1. **Replace `ENV_PREFIX`** with the detected framework prefix
2. **Replace the color picker section** with the appropriate implementation:
   - Radix: use the grid from `radix-colors.md`
   - Tailwind: offer preset HSL values
   - None: accept freeform CSS color
3. **Adjust `.env` path** if the project uses `.env.local` as primary (e.g., Next.js projects often do)
4. **Use `.js` extension** if the project doesn't use TypeScript (and adjust imports to `require`)
