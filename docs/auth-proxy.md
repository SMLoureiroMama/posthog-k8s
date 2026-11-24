# Trusted-header login

Status: **in progress**. The router and NetworkPolicy side is in the chart; the login itself needs
the overlay image, which is not published yet. Until then `authProxy.enabled` changes what the
router sends and nobody is logged in by it.

| Piece | State |
|---|---|
| Router marks `app` requests to `web` with a shared secret; `ingest` strips the identity headers | in the chart |
| NetworkPolicy: only `appIngressFrom` reaches the `app` listener | in the chart |
| `authProxy.*` values and their schema | in the chart |
| Overlay image with the login backend, used by `web` | planned |
| `authProxy.groups` reconciliation, `authProxy.passwordLogin` | planned, with the overlay image |

## Goal

Users who reach the `app` listener through an authenticating proxy (Teleport Application Access,
oauth2-proxy, Pomerium, Authelia, Cloudflare Access) should be logged into PostHog automatically,
and their organization memberships should follow the groups the proxy asserts:

```
X-Forwarded-Email:  jane@example.com
X-Forwarded-User:   jane
X-Forwarded-Groups: platform-admins,acme-developers
```

```yaml
authProxy:
  enabled: true
  groups:
    platform-admins: {organizations: ["*"], level: admin}
    acme-developers: {organizations: [acme], level: member}

networkPolicy:
  enabled: true
  appIngressFrom:
    - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: oauth2-proxy}}
```

- Accounts are matched by email (`authProxy.emailHeader`). **The proxy must only assert verified
  addresses**: whoever can make it send `jane@example.com` is Jane.
- `platform-admins` → admin in every organization; `acme-developers` → member of the organization
  with slug `acme`. Organizations are referenced by slug or ID (names are not unique) and must
  exist; the login never creates one.
- A user without any mapped group gets 403 and no account.
- Groups are authoritative for member and admin memberships. Owner is never granted or removed.
- A user left without any organization is deactivated (`is_active=False`) and reactivated when a
  group grants access again. Reconciliation runs when the session's (email, groups) changes, so a
  user removed from the identity provider is only deactivated if they show up again.
- `authProxy.passwordLogin: false` turns off PostHog's own login, signup, password reset and
  invite acceptance, leaving the proxy as the only way in.

## Why an overlay image

PostHog's self-hosted edition has no trusted-header or OIDC login backend; SAML is a paid,
Cloud-only feature. Django itself ships `RemoteUserMiddleware`/`RemoteUserBackend`, so the plan
is a small module plus a settings patch, delivered as an overlay image
`ghcr.io/enricofranke/posthog-k8s/posthog-web:<upstream-sha>` built `FROM posthog/posthog@<digest>`
and pinned by digest like the other images. Only the `web` service uses it. A mounted module
instead of an image does not work without replacing upstream's ASGI entry point, which sets
`DJANGO_SETTINGS_MODULE` unconditionally.

## Trust model

`web` cannot tell from the identity headers alone where a request came from: every pod of the
release may talk to `web:8000` directly, and some of them fetch pages on behalf of users. Three
things together make the headers trustworthy. The router and NetworkPolicy parts are in the chart;
what `web` does with the headers comes with the overlay image.

1. **Only the proxy reaches the `app` listener.** With `networkPolicy.enabled` the router is not
   part of the policy that opens the release's pods to each other; its `app` port accepts
   `networkPolicy.appIngressFrom` and nobody else, the release's own pods included. The chart
   refuses to render `authProxy.enabled: true` without `networkPolicy.enabled: true` and a
   non-empty `appIngressFrom`.
2. **Only the `app` listener knows the shared secret.** The router adds
   `X-Posthog-K8s-Auth-Proxy: <AUTH_PROXY_SECRET>` to what the `app` listener forwards to `web`,
   replacing any copy the client sent. The login backend honours the identity headers only on
   requests that carry it; a pod that goes around the router does not have the secret.
3. **The `ingest` listener strips** `authProxy.userHeader`, `emailHeader`, `groupsHeader`, the
   secret header and `router.stripHeaders`, whether `authProxy` is enabled or not.

The login backend also rejects a request in which an identity header occurs more than once. Django
joins repeated headers with a comma, so a proxy that appends its header to one the client sent,
instead of replacing it, would let the client add groups.

This mirrors how Argo CD (Dex authproxy connector) and Headlamp trust an upstream proxy. Your part
is the proxy: it has to set all three headers on every request, overwriting what the client sent.

### The shared secret

`AUTH_PROXY_SECRET` is handled like the chart's other secrets: `authProxy.sharedSecret` inline,
generated and kept across upgrades when empty, read from `posthog.existingSecret` when that is set,
or mapped with `secrets.keys.AUTH_PROXY_SECRET: {secret: <name>, key: <key>}`. Under Argo CD use
one of the last two; a generated value would change on every sync. Caddy gets it through its
environment, not through the ConfigMap, and the router does not start with an empty one. The
pods read it at start: restart the router after rotating it.

## Not in scope

Project-level access control inside an organization is a paid PostHog feature. Organizations are
the isolation boundary on self-hosted instances; the mapping above is deliberately organization
based.
