# Keycloak Admin Setup

To make the passkey extension work, Keycloak admins need to install the provider JAR and configure the target realm and clients.

Minimum supported Keycloak version: `26.5`.

## 1. Install The Provider

1. Download or build the plugin JAR.
2. Copy it into Keycloak's providers directory:

```bash
cp custom-passkey-*.jar keycloak/providers/
```

3. If Keycloak runs in Docker, mount the providers directory into the container:

```yaml
services:
  keycloak:
    volumes:
      - ./keycloak/providers:/opt/keycloak/providers:ro
```

4. Restart Keycloak so the provider is loaded.

## 2. Configure Passwordless WebAuthn

In the target realm open the Passkey settings: `Authentication` -> `Policies` -> `WebAuthn Passwordless Policy` and configure the following settings:

1. `Enable Passkeys` -> `ON`.
2. Set the Passwordless Relying Party ID to the largest common application host. So if the apps are running at `app1.foo.example.com` and `app2.foo.example.com`, that would be `foo.example.com`. Use `localhost` for local development.
3. Set `Require discoverable credential` to `Yes`.
4. Set `User verification requirement` to `Required`.

## 3. Configure Each Client

For every OIDC client that should use passkey login:

1. Add the application origin to `Web Origins`, for example `https://app.example.com`.
2. Add the application callback URLs to `Redirect URIs`.
3. Add the silent check-sso page to `Redirect URIs`, for example `https://app.example.com/silent-check-sso.html`.

After this setup, the extension is available under:

```text
/realms/{realm}/passkey/{clientId}/challenge
/realms/{realm}/passkey/{clientId}/save
/realms/{realm}/passkey/{clientId}/authenticate
```
