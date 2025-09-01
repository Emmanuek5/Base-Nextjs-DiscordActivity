## TicTakTo — Discord Embedded App (Next.js 15)

[![Next.js](https://img.shields.io/badge/Next.js-15-black?logo=nextdotjs)](https://nextjs.org)
[![React](https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=black)](https://react.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-5-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind-3.4-38B2AC?logo=tailwindcss&logoColor=white)](https://tailwindcss.com/)

A modern Discord Embedded App (Activity) scaffold built with Next.js App Router, React 19, TypeScript, and Tailwind CSS. It integrates the official `@discord/embedded-app-sdk` via a typed React context to authenticate users and access guild/channel/user data inside Discord.

### Highlights
- **Discord SDK integration**: Ready-to-use `DiscordProvider` with auth flow and state (user, guild, channel).
- **App Router + TypeScript**: Next.js 15 with strict TS, path aliases (`@/*`).
- **Tailwind CSS**: Utility-first styling wired up in `app/globals.css` and `tailwind.config.ts`.
- **Turbopack dev**: Fast local development via `next dev --turbopack`.

## Tech Stack
- **Framework**: Next.js 15 (App Router)
- **UI**: React 19, Tailwind CSS
- **Lang/Tools**: TypeScript 5, ESLint (Next config)
- **Discord**: `@discord/embedded-app-sdk` (via `contexts/DiscordContext.tsx`)

## Project Structure
```text
workspace/
  app/
    layout.tsx          # Root layout (Geist fonts, globals)
    page.tsx            # Starter page
    globals.css         # Tailwind layers, CSS variables
  contexts/
    DiscordContext.tsx  # Discord SDK auth + user/guild/channel context
  public/               # Static assets (icons, svgs)
  next.config.ts        # Next.js config
  tailwind.config.ts    # Tailwind content/theme
  tsconfig.json         # TS config with path alias @/*
  eslint.config.mjs     # ESLint config (flat)
  package.json          # Scripts & deps
```

## Prerequisites
- Node.js 18.18+ or 20+
- A Discord Application with the **Embedded App** feature enabled

## Discord Setup (once)
1. Create an application in the Discord Developer Portal.
2. Enable the **Embedded App** feature.
3. Add OAuth2 Redirect URI(s) you will use for the token exchange (e.g., your deployed URL).
4. Note your **Client ID** and **Client Secret**.

## Environment Variables
Create a `.env.local` in the project root:

```env
# Used on the client (safe to expose)
NEXT_PUBLIC_DISCORD_CLIENT_ID=YOUR_DISCORD_CLIENT_ID

# Used only by the server-side token proxy
DISCORD_CLIENT_ID=YOUR_DISCORD_CLIENT_ID
DISCORD_CLIENT_SECRET=YOUR_DISCORD_CLIENT_SECRET
DISCORD_REDIRECT_URI=https://YOUR_DOMAIN/.proxy/api/token
```

Notes:
- `NEXT_PUBLIC_DISCORD_CLIENT_ID` is required by the client-only `DiscordProvider`.
- The app expects a server endpoint at `/.proxy/api/token` to exchange the OAuth2 code for an access token (see below).

## Token Proxy (OAuth2 Code Exchange)
Implement a minimal server route to exchange the `authorize` code for a token. In a Next.js App Router app, you can create:

```tsx
// app/.proxy/api/token/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function POST(request: NextRequest) {
  const { code } = await request.json();
  if (!code) return NextResponse.json({ error: "Missing code" }, { status: 400 });

  const body = new URLSearchParams({
    client_id: process.env.DISCORD_CLIENT_ID!,
    client_secret: process.env.DISCORD_CLIENT_SECRET!,
    grant_type: "authorization_code",
    code,
    redirect_uri: process.env.DISCORD_REDIRECT_URI!,
  });

  const res = await fetch("https://discord.com/api/oauth2/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body,
  });

  if (!res.ok) {
    const text = await res.text();
    return NextResponse.json({ error: text }, { status: 500 });
  }

  const json = await res.json();
  return NextResponse.json(json);
}
```

The `contexts/DiscordContext.tsx` calls this endpoint at `/.proxy/api/token` to complete the OAuth flow and then fetches user/guild/channel data using the returned `access_token`.

## Wiring the Provider
Wrap your app with the `DiscordProvider` and pass your `NEXT_PUBLIC_DISCORD_CLIENT_ID`:

```tsx
// app/providers.tsx
"use client";
import { DiscordProvider } from "@/contexts/DiscordContext";

export default function Providers({ children }: { children: React.ReactNode }) {
  return (
    <DiscordProvider clientId={process.env.NEXT_PUBLIC_DISCORD_CLIENT_ID!}>
      {children}
    </DiscordProvider>
  );
}
```

Then include it in your root layout:

```tsx
// app/layout.tsx
import Providers from "./providers";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

## Getting Started (Local)
```bash
pnpm install     # or npm install / yarn / bun install
pnpm dev         # runs Next.js with Turbopack
```

- App runs at `http://localhost:3000`.
- The Discord SDK requires the app to run inside Discord to fully initialize; outside the Discord client you may see initialization/auth errors. This is expected.

## Scripts
- `dev`: Start development server (Turbopack)
- `build`: Production build
- `start`: Start production server
- `lint`: Run ESLint

## Deployment
- Set the environment variables in your hosting provider (Vercel recommended).
- Ensure `DISCORD_REDIRECT_URI` exactly matches the value configured in the Discord Developer Portal.
- Serve the token proxy route at `/.proxy/api/token` over HTTPS.

## Troubleshooting
- **Failed to initialize Discord SDK**: Ensure the app is running as an Embedded App inside Discord and is served over HTTPS.
- **Authentication failed**: Confirm the token proxy route is reachable, returns JSON, and client/secret/redirect URI match Discord settings.
- **CORS/401 issues**: The token exchange must be server-side; never expose your client secret to the browser.

## License
MIT
