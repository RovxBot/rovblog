# WDAC Configuration

## Introduction
Hey all! Welcome back to my series on WDAC. In this article we will go over the steps to configure a base policy and supplemental policy using [SpyNetGirl's](https://github.com/HotCakeX/Harden-Windows-Security) App Control Manager. If you haven't already, go check out my previous article on [Getting Started](/WDAC/GettingStarted.md) with WDAC.

## The Base Policy
The base policy for WDAC is all about blocking anything that is not signed by Microsoft. If its not used to run Windows or 365 Apps, then its blocked. This is not an ideal policy for production, but it is a great starting point for building your own policy.

Each organization will have its own requirements for what is allowed to run on their devices and this is where the supplemental policy comes in.

- The base policy blocks all bar Microsfot signed
- The supplemental policy allows approved apps that are blocked by the base policy

If you have not already, go and install App Controll manager:

```powershell
(irm 'https://raw.githubusercontent.com/HotCakeX/Harden-Windows-Security/main/Harden-Windows-Security.ps1')+'AppControl'|iex
```
!!! UPDATE "UPDATE"
    Since writing this article, SpyNetGirl has now released a new version that is available on the Windows Store!

For further guidance on installing App Control Manager, check out the [GitHub page](https://github.com/HotCakeX/Harden-Windows-Security)
