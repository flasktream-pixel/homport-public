# Frequently asked questions

## Is Homport available now?

Yes. [Get Homport on Microsoft Store](https://apps.microsoft.com/detail/9PGG4B8DNL6B) for Windows 10 / 11 (x64). The [website](https://homport.dev/) has an interactive product walkthrough. Install and update through Microsoft Store.

## Is it free?

The first release is free. Domain registration and any Cloudflare charges are separate. Pricing for later releases has not been announced.

## What does a shared link open?

It opens a compatible web app that is already running on your Windows PC, such as a small project preview. Homport supplies the address and access controls; it does not create the content. Private shares require an allowed email address and email verification before the visitor can reach the app.

## Do visitors need to install anything?

They use a browser. They do not need Homport or a VPN. A source app can still have its own login or browser requirements.

## Will the link work when my PC is off?

No. Your PC, its connection, Homport's connector and the source app must remain available. Homport runs in your signed-in Windows session, not as an always-on Windows service.

## Can I share any app or port?

No. The first release supports compatible HTTP web apps on localhost, with connection checks. Native-app protocols, camera feeds, video streaming, large file storage and uploads are outside the intended scope.

## Where does my data go?

Your web app remains on your PC and uses Cloudflare's network for remote access. Homport uses your own Cloudflare account and has no developer backend or desktop telemetry. The public website uses Cloudflare Web Analytics. See the [privacy policy](https://homport.dev/privacy/) for details.

## Does Homport send the link through a messenger?

No. Homport copies the address; you paste and send it in your preferred messenger or email app. The website's animated conversation is a demonstration of that workflow.

## Is Homport open source?

This repository publishes product information and reusable documentation. The application source code remains private. The [documentation license](../LICENSE) does not license the application.

## Is this an official Cloudflare app?

No. Homport is an independent third-party application, not affiliated with or endorsed by Cloudflare, Inc.

## Where can I get help?

Visit [support](https://homport.dev/support/) or read the [reporting guide](../SUPPORT.md). Security concerns should be reported [privately](../SECURITY.md).
