# Homport — public notes

This repository holds the parts of the Homport project that stand on their own and are worth sharing. The product source lives in a private repository.

## Contents

- [`docs/oauth-cloudflare-desktop.md`](docs/oauth-cloudflare-desktop.md) — Cloudflare's OAuth flow for a **desktop** application: public client, PKCE, no client secret, no refresh token, catching the callback on loopback, the six scopes you need and what each one actually grants, the traps and how to spot them. Everything in it was run against a real Cloudflare account; anything not verified in practice is marked as such.

## About Homport

Homport is a Windows application that turns a web app running on your PC into a private web address, using Cloudflare Tunnel and Cloudflare Access. It runs on your own Cloudflare account. There is no developer backend and no telemetry.

Independent third-party app, not affiliated with Cloudflare, Inc.

## License

See [LICENSE](LICENSE).
