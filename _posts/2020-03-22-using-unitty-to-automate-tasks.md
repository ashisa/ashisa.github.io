---
title: Using UniTTY to automate tasks
tags: unitty azure docker
---

Thinking inside, outside and around the box is a big part of my job allows me to do exactly that. I have been exploring stuff related to Azure, Kubernetes and automating certain tasks which the current ways of automation do not allow.

Last week, this quest lead me to two awesome projects on GitHub - [ttyd](https://github.com/tsl0922/ttyd) and [GoTTY](https://github.com/yudai/gotty). Both these projects allow you to turn any command line tool to a web app.

This got me started with building something that uses this newfound power to do things in a new way and I came up with [UniTTY](https://github.com/ashisa/unitty).

[UniTTY](https://github.com/ashisa/unitty) is a Linux container image that comes preintalled with _Azure CLI_, _PowerShell Core 7_ and _GoTTY_. You can head over the the project here - https://github.com/ashisa/unitty - to read more about UniTTY and learn how to use it because here we will just see how I am using it for automation.

Let's dive in!

The problem that I want to solve with _UniTTY_ deals with deploying services on Azure. You'd think why is that a big deal? And, you are right! - deploying Azure services and automating them isn't a big deal and there are a lot of ways to do that - ARM templates, Azure CLI, PowerShell, Ansible, Terraform, REST APIs and more...








