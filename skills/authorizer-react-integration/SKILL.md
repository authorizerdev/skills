---
name: authorizer-react-integration
description: Add login, signup, logout and protected routes to a React or Next.js app with @authorizerdev/authorizer-react (AuthorizerProvider, Authorizer component, useAuthorizer hook). Use when the user says "add login to my React app", "add Authorizer to Next.js", "protect a route", "get the access token in React", "show the user's email", "logout button", "authorizer-react", or wants a frontend to talk to an Authorizer server. Not for backend token validation (see authorizer-backend-auth).
---

# Add Authorizer login to React / Next.js

Prerequisite: a running Authorizer and its `client_id` (see `authorizer-local-setup`).
The app's origin must be allowed by the server: start it with
`--allowed-origins=<app origin>,<authorizer url>` (e.g.
`--allowed-origins=http://localhost:3000,http://localhost:8080`). With the server default
`*`, cross-origin POSTs from the browser are rejected with `403 csrf_validation_failed`.

## 1. Install

```bash
npm install @authorizerdev/authorizer-react
```

Package exports (v2.x): `AuthorizerProvider`, `useAuthorizer`, `Authorizer`,
`AuthorizerSignup`, `AuthorizerBasicAuthLogin`, `AuthorizerMagicLinkLogin`,
`AuthorizerForgotPassword`, `AuthorizerSocialLogin`, `AuthorizerResetPassword`,
`AuthorizerVerifyOtp`, `AuthorizerTOTPScanner`, `AuthorizerPasskeyLogin`,
`AuthorizerPasskeyRegister`, `AuthorizerMFASetup`. Stylesheet:
`@authorizerdev/authorizer-react/styles.css`. Do not invent other exports.

## 2. Wrap the app in the provider (plain React / Vite)

```jsx
import { AuthorizerProvider } from '@authorizerdev/authorizer-react';
import '@authorizerdev/authorizer-react/styles.css';

export default function Root({ children }) {
  return (
    <AuthorizerProvider
      config={{
        authorizerURL: 'http://localhost:8080',
        redirectURL: window.location.origin,
        clientID: 'YOUR_CLIENT_ID', // value of --client-id
      }}
    >
      {children}
    </AuthorizerProvider>
  );
}
```

`config` takes `authorizerURL`, `redirectURL`, `clientID`. An optional
`onStateChangeCallback={async ({ user, token }) => {}}` fires on every auth-state change.
The provider loads feature flags from the server and refreshes the access token before
it expires.

## 3. Render login and gate content

```jsx
import { useAuthorizer, Authorizer } from '@authorizerdev/authorizer-react';

function RequireAuth({ children }) {
  const { token, loading } = useAuthorizer();
  if (loading) return <h1>Loading…</h1>;   // restoring session: don't flash the login form
  if (!token) return <Authorizer />;        // full login/signup/social/MFA flow
  return children;
}

function Profile() {
  const { user, logout } = useAuthorizer();
  return user ? <div>{user.email} <button onClick={logout}>Log out</button></div> : null;
}
```

`useAuthorizer()` returns `user`, `token` (`access_token`, `id_token`, `refresh_token`),
`loading`, `config`, `logout`, `setUser`, `setToken`, `setLoading`, `setAuthData`, and
`authorizerRef` (the underlying `@authorizerdev/authorizer-js` instance).
`<Authorizer onLogin={(data) => …} roles={['user']} />` accepts a login callback and a
role restriction. MFA screens render automatically during login.

## 4. Call your backend with the token

```jsx
const { token } = useAuthorizer();
await fetch('https://api.example.com/things', {
  headers: { Authorization: token ? `Bearer ${token.access_token}` : '' },
});
```

The backend must validate it — see `authorizer-backend-auth`.

## 5. Next.js

`authorizer-react` is client-side only. App Router:

```tsx
// app/providers.tsx
'use client';

import { AuthorizerProvider } from '@authorizerdev/authorizer-react';
import '@authorizerdev/authorizer-react/styles.css';

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <AuthorizerProvider
      config={{
        authorizerURL: process.env.NEXT_PUBLIC_AUTHORIZER_URL!,
        redirectURL: typeof window !== 'undefined' ? window.location.origin : '',
        clientID: process.env.NEXT_PUBLIC_AUTHORIZER_CLIENT_ID!,
      }}
    >
      {children}
    </AuthorizerProvider>
  );
}
```

```tsx
// app/layout.tsx
import { Providers } from './providers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body><Providers>{children}</Providers></body>
    </html>
  );
}
```

Mark every component that calls `useAuthorizer()` or renders an Authorizer component
with `'use client'`. Pages Router: wrap `<Component {...pageProps} />` in
`AuthorizerProvider` inside `pages/_app.tsx` with the same config. For SSR that needs the
user, validate the session cookie server-side with `validate_session` instead of relying
on client context.

Only `NEXT_PUBLIC_AUTHORIZER_URL` and `NEXT_PUBLIC_AUTHORIZER_CLIENT_ID` belong in the
browser. Never expose `client_secret` or the admin secret to frontend code.

## 6. Reset-password route (optional)

```jsx
import { AuthorizerResetPassword } from '@authorizerdev/authorizer-react';

export default function ResetPasswordPage() {
  return <AuthorizerResetPassword onReset={() => (window.location.href = '/')} />;
}
```

## Verify

1. `curl -s -o /dev/null -w '%{http_code}\n' -X POST "$AUTHORIZER_URL/graphql" -H 'Content-Type: application/json' -H 'Origin: http://localhost:3000' -d '{"query":"{ meta { version } }"}'`
   prints `200` (a `403` means your app origin is missing from `--allowed-origins`).
2. Start the app, sign up through `<Authorizer />`, and confirm the profile renders
   `user.email`; reload — the session restores (`loading` → `token` set) without a login.
3. In DevTools → Network, requests to your API carry `Authorization: Bearer eyJ…`.
4. Click logout; `token` becomes `null` and the login form returns.

Docs: https://docs.authorizer.dev/sdks/authorizer-react/integration
