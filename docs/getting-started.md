# Getting started with Homport

Homport's Microsoft Store release is being prepared. This guide describes the intended first-release workflow. Visit [homport.dev](https://homport.dev/) for current availability and an interactive demonstration.

## Before you start

Use a Windows 10 (build 19041 or later) or Windows 11 x64 PC. You need a Cloudflare account, a domain using Cloudflare nameservers, and Zero Trust configured. Cloudflare may require billing details during Zero Trust setup; review its plan terms before confirming.

Start the web app you want to share and confirm that it works in a browser on the same PC using an HTTP localhost address, such as `http://localhost:3000`. Homport does not install or configure the source app for you.

## Connect and create a share

1. Open Homport and sign in to Cloudflare in your browser. Review the requested permissions before approving.
2. Follow the setup checklist to choose your account and domain and complete the Zero Trust checks.
3. Choose **New share**, select the local web app and choose an available subdomain.
4. For a private share, enter the email addresses allowed to visit. For a public share, read and confirm that anyone with the address can reach the app.
5. Wait for the connection checks. A timed-out check is unverified, not proof that the link is ready.
6. Copy the ready link and send it yourself. For private sharing, ask an allowed recipient to open the link and complete the email-code check.

## What the recipient sees

A public link opens the source web app in the recipient's browser. A private link first presents Cloudflare Access sign-in. After the allowed recipient verifies their email, they can reach the source app, which may still require its own login.

The recipient does not need Homport or a VPN. Your PC must stay on and connected, your Windows session must remain signed in, and the source app must keep running. Homport is not cloud hosting.

## Stop or troubleshoot access

- Use **Turn off** to disable a particular share.
- Use **Lock now** to stop this PC's connector while retaining its configuration. This is separate from locking Windows.
- Closing the main window leaves Homport running in the tray. Use **Quit** when you want to exit.
- If a link needs attention, use diagnostics and follow the displayed guidance. Avoid editing resources managed by Homport directly in the Cloudflare dashboard; unexpected changes can pause further updates until reviewed.

See the [FAQ](faq.md) or [support guide](../SUPPORT.md) for help. Never include credentials or email verification codes in a public report.
