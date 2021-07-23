---
title: "vArcade – An exclusive preview"
date: 2021-07-23T17:07:15+01:00
draft: false
featured: true
tags: [
    "code",
    "clarity",
    "angular"
]
---


Here is our new project.
*We are still working on it and expect to release the first version early September. __Stay tuned!__*

## Why vArcade

What do people like in arcade gaming? Simplicity. Choose your game, insert a coin, play. There are only 3 simple steps with no complex setup, no specific prerequisites, nothing to install on your computer.

What if you could consume infrastructure automation as simple as that? With vArcade, we’ve got you covered.
> We want to make automation easy. We want to make automation fun.

Traditionally, when you want to start with automation there is a lot to learn and a slow learning curve. Of course, a ton of great VMware automation content is available on the web, from bloggers that are truly inspirational for us, to the VMware Code sample exchange. But they are difficult to consume for newbees, all using different tooling or scripting languages.

vArcade allows you to demonstrate the automation capabilities of VMware products easily and quickly to your team, to your customer, to your boss… You don’t need to learn how the APIs works, a programming/scripting/automation language, nor to install anything on your computer. You either don’t need to download a script somewhere on your computer, look at the instructions file, install the required packages and dependencies, adapt the instructions whether you are running on Windows, Mac or Linux, customize it in a text editor, and s.o.

We’ve tried to make infrastructure automation as easy as playing an arcade game.

## vArcade principles

vArcade provides a simple web user interface to deploy community pre-built automation scripts. It only requires 3 simple steps to run a script:
-	Select a script in the Playstore,
-	customize the variables,
-	run it.

A collection of scripts is included and ready to use in the interface. We provide automation demo content for VMware solutions, including NSX-T, VMware SD-WAN, HCX and NSX Advanced Load Balancer.

vArcade support scripts written in major automation languages (i.e. Terraform, Ansible, Python, Bash, Powershell…).

## Getting started

There is nothing to install on your computer.

Download and install the OVA package. It’s based on Photon OS with virtual hardware version 13, so hopefully, it’s compatible with a lot of ESXi, Workstation and Fusion versions.

Once powered on, you can access the Web UI (HTTPS/443) on the VM IP. You can then login using the credentials you provided at install.

![varcade-login](/varcade-login.png "vArcade Login Page")

## Play

### Step 1 : In the Playstore, find an interesting automation script you’d like to run 
The user is presented a list of scripts that are available to run from the interface.

![varcade-playstore](/varcade-playstore.png "vArcade Playstore Page")

### Step 2 : Fill the required input variables
Then user is prompted to fill the list of necessary variables (example : NSX Manager IP, login and password). The Graphical interface will adapt to the requested script, building the web UI on the fly. 

![varcade-variables](/varcade-variables.png "vArcade variables Page")

### Step 3 : review the generated code and hit the actions buttons to run it !
And that’s it. vArcade will generate the code. Just observe the code running. 

![varcade-playstore](/varcade-run.png "vArcade run Page")

## vArcade needs you

We’d like everyone to join and contribute to the vArcade Playstore. We try to keep things readable so that everyone could contribute. vArcade Playstore is maintained on our github repository. It’s just an easy to read and update JSON file. When someone publish a scripts, they need to fill a datamodel form in order to describe the variables requested by the script. And that's it.

Please note : Currently the project is open only to the main team. If you like to see your automation scripts in vArcade Playstore please drop us a message.

## Wait, there is more

We learn a lot creating this new projet and we can't wait to share more with you. Other posts are coming. 

And yes, vArcade runs in Dark Mode ;-)
