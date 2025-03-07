# Windows Defender Application Control (WDAC)

## Introduction
Windows Defender Application Control (WDAC) is a powerful security feature in Windows that helps organizations control which applications and scripts can run on their endpoints. Deploying WDAC via Microsoft Intune ensures policy enforcement across managed devices, providing a robust defense against... Anyone doing litteraly anything outside of what you allow them to do.

## Understanding WDAC
WDAC allows organizations to define rules that permit only approved applications and scripts to run. Unlike traditional application control methods, WDAC leverages code integrity policies and can operate in audit or enforced mode. 

Obviously Audit mode is just that, it audits. It doesn't block anything, but reports on anything that may have been blocked. Enforced mode is the opposite, it blocks anything that hasn't been approved.

!!! danger "DANGER"
    WDAC is a powerful security feature that can have a significant impact on your environment. Before deploying WDAC, it is important to understand how it works and how it will affect your organization. ALWAYS run it on a small subset of devices first to ensure it doesn't break anything, even after you run in audit mode and think you have it all under control. Ask me how I know...

## Getting Started
OK so to get started we should firt go over everyrhing you are going to need to get started.
- Defender for Endpoint P2 license
- Windows 10 Enterprise or Education
- Intune enrolled devices
- WDAC Wizard, or even better, App Control Manager. (Please go and check out SpyNetGirl's [App Control Manager](https://github.com/HotCakeX/Harden-Windows-Security))
- Device Groups based on your requirements

So basically what you will be doing is creating an audit mode policy to block everything bar Micorsoft signed. You will then monitor this in the Security centre for a few weeks to see what is getting blocked. Cross check this with your list of approved apps. Then build a supplemental policy to allow these apps to run.
So in a list:
- Create a list of apporved apps
- Create a base policy to block all bar MS signed
- Monitor for a few weeks
- Create a supplemental policy to allow approved apps that are blocked by the base policy
- Monitor for a few weeks
- Deploy the policy to a small subset of devices in enforced
- update as required

!!! note "NOTE"
    Lucky for you Intune adds a few benifits that mean you dont have to keep updating the supplemental policy. We will get to all that soon though.

## Next time
Check out my next article on configuration of WDAC. We will go over the steps to configure a base policy and supplemental policy using [SpyNetGirl's](https://github.com/HotCakeX/Harden-Windows-Security) App Control Manager.
