# Application Developer Setup

This guide covers only what must be configured in the client application. It assumes a Keycloak admin has already installed the provider and configured the realm/client.

## 1. Client Configuration

Configure your app with the Keycloak base URL, realm, client id, and the WebAuthn RP ID from your Keycloak admin:

```js
const keycloakConfig = {
  url: 'https://keycloak.example.com',
  realm: 'my-realm',
  clientId: 'my-client'
};

const passkeyRpId = 'example.com';
```

The `passkeyRpId` must match the Passwordless WebAuthn relying party ID configured in Keycloak.

## 2. Silent Check SSO

Add a silent check-sso callback page to your app, for example `public/silent-check-sso.html`:

```html
<!doctype html>
<html lang="en">
  <body>
    <script>parent.postMessage(location.href, location.origin);</script>
  </body>
</html>
```

Initialize `keycloak-js` with `check-sso`:

```js
const checkSsoOptions = {
  onLoad: 'check-sso',
  pkceMethod: 'S256',
  silentCheckSsoRedirectUri: `${window.location.origin}/silent-check-sso.html`,
  silentCheckSsoFallback: false
};
```

## 3. Passkey Endpoints

Build endpoint URLs with the client id in the path:

```js
function passkeyUrl(path) {
  return `${keycloakConfig.url}/realms/${encodeURIComponent(keycloakConfig.realm)}/passkey/${encodeURIComponent(keycloakConfig.clientId)}/${path}`;
}
```

Always fetch a fresh challenge before registering or authenticating:

```js
async function checkPasskeyExtensionHealth() {
  const res = await fetch(passkeyUrl('health'), { credentials: 'include' });
  const body = await res.json().catch(() => ({}));
  if (!res.ok || body.available !== true) {
    throw new Error(body.error || 'Passkey extension is not available');
  }
}

async function getChallenge() {
  const res = await fetch(passkeyUrl('challenge'), { credentials: 'include' });
  const body = await res.json().catch(() => ({}));
  if (!res.ok) throw new Error(body.error || 'Failed to get challenge');
  return body.challenge;
}
```

## 4. Encoding

WebAuthn returns binary `ArrayBuffer` values. Encode them as base64url before sending JSON:

```js
function toBase64Url(buffer) {
  const bytes = new Uint8Array(buffer);
  let binary = '';
  for (const b of bytes) binary += String.fromCharCode(b);
  return btoa(binary).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/g, '');
}

function fromBase64Url(value) {
  const base64 = value.replace(/-/g, '+').replace(/_/g, '/');
  const padded = base64 + '='.repeat((4 - (base64.length % 4)) % 4);
  return Uint8Array.from(atob(padded), (c) => c.charCodeAt(0));
}
```

## 5. Register A Passkey

Registration requires a logged-in user and a bearer token. Use the Keycloak subject (`sub`) as the WebAuthn user id:

```js
const challenge = await getChallenge();
const userId = keycloak.tokenParsed?.sub;
if (!userId) throw new Error('Access token subject is required');

const username = keycloak.tokenParsed?.preferred_username || userId;
const displayName = keycloak.tokenParsed?.name || username;
const userIdBytes = new TextEncoder().encode(userId);
if (userIdBytes.length > 64) {
  throw new Error('User id is too long for WebAuthn user.id');
}

const credential = await navigator.credentials.create({
  publicKey: {
    challenge: fromBase64Url(challenge),
    rp: { name: 'My App', id: passkeyRpId },
    user: {
      id: userIdBytes,
      name: username,
      displayName
    },
    pubKeyCredParams: [{ type: 'public-key', alg: -7 }],
    authenticatorSelection: { residentKey: 'required', userVerification: 'required' },
    attestation: 'none'
  }
});

const platform =
  navigator.userAgentData?.platform ||
  navigator.platform ||
  'Unknown Platform';

const browser =
  navigator.userAgentData?.brands?.find((b) => b.brand && b.brand !== 'Not_A Brand')?.brand ||
  (navigator.userAgent.includes('Firefox') ? 'Firefox' :
   navigator.userAgent.includes('Edg') ? 'Edge' :
   navigator.userAgent.includes('Chrome') ? 'Chrome' :
   navigator.userAgent.includes('Safari') ? 'Safari' :
   'Unknown Browser');

const deviceName = `${platform} - ${browser}`;

await fetch(passkeyUrl('save'), {
  method: 'POST',
  credentials: 'include',
  headers: {
    'Content-Type': 'application/json',
    Authorization: `Bearer ${keycloak.token}`
  },
  body: JSON.stringify({
    deviceName,
    credentialId: toBase64Url(credential.rawId),
    clientDataJSON: toBase64Url(credential.response.clientDataJSON),
    attestationObject: toBase64Url(credential.response.attestationObject),
    challenge
  })
});
```

The example uses `alg: -7` (`ES256`). Keep `pubKeyCredParams` aligned with the algorithms allowed in the Keycloak Passwordless WebAuthn policy.

## 6. Login With A Passkey

Passkey login uses a discoverable credential, so do not set `allowCredentials`:

```js
const challenge = await getChallenge();
const assertion = await navigator.credentials.get({
  publicKey: {
    challenge: fromBase64Url(challenge),
    userVerification: 'required'
  }
});

if (!assertion.response.userHandle) {
  throw new Error('Passkey did not return a user handle');
}

const res = await fetch(passkeyUrl('authenticate'), {
  method: 'POST',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json', Accept: 'application/json' },
  body: JSON.stringify({
    credentialId: toBase64Url(assertion.rawId),
    userHandle: toBase64Url(assertion.response.userHandle),
    clientDataJSON: toBase64Url(assertion.response.clientDataJSON),
    authenticatorData: toBase64Url(assertion.response.authenticatorData),
    signature: toBase64Url(assertion.response.signature),
    challenge
  })
});

if (res.status !== 204) {
  throw new Error(`Passkey auth failed: ${res.status}`);
}
```

After a successful passkey login, run `keycloak.init(checkSsoOptions)` again so `keycloak-js` collects fresh tokens from the new Keycloak browser session.

## Important Client Rules

- Use `credentials: 'include'` for all passkey endpoint calls.
- Send `Authorization: Bearer <token>` only for `/save`.
- Use `credentialId`, not `rawId`, in the JSON payload.
- Send `userHandle` during `/authenticate`.
- Keep the RP ID, app origin, and Keycloak client configuration aligned.
