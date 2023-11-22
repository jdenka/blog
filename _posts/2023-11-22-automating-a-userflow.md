---
layout: post
title: Automating a userflow
categories:
- Azure
- Automation
- PowerShell
- Identity
tags:
- azure
- automation
- powershell
- identity
pin: true
date: 2023-11-22 16:00 +0100
---
Quite recently I was tasked to code a solution due to an acquisition. This post will walk you through on the process of creating an automated flow that relies on data collected from a webhook.  

## Discussion phase
There were several tasks that needed to be completed before day 1 where everything was supposed to be in place and there were some questions that needed answers, but the two biggest questions that came to my table was the following. 

1. How do we give the users their new account information?
2. External communication needs to be blocked on the old account how do we do that? 

The discussion went in all places during the discussion phase, should the service desk provide the new user information to each user or should all 4000+ users have to visit the office to get hold of the new user account in the new company? 

The answer to both the questions where PowerShell.

## Identifying the users
To identify the users that needed everything for Day 1 we were lucky enough that we have a HR system that is considered the master for Active Directory and since this was not the first acquisition there were already a custom attribute in place for this. 

Luckily, my awesome friend [Emil](https://www.ehmiiz.se) had a previous automation that took a look at the custom attribute and put the users inside of a group in Active Directory so that was easy enough just to update with another group and a new rule for the new company. It's quite an awesome automation since it makes the groups dynamic depending on the custom attribute set on the user, and since those groups are synced to Entra ID it had its use case defined. 

Armed with everything needed I went to work. 

## The scripting process
Early on, I started scripting to solve everything that was needed for day 1 of the acquisition. 
It came to be three different kinds of scripts. 

Started with the script to send out the users new identities in the new company. I got use of an old function that I had since before that sends mail with [Send-MgUserMail](https://github.com/jdenka/Powershell-script-collection/blob/main/Send-AutomatedEmail.ps1) that was modified to solve the issue. During this time we used an App registration that had the required Graph permissions (Mail.Send) to impersonate anyone that had a mailbox in EXO. 
Had access to a CSV file with made up information, but that had the required information about users current mail, the new mail and the password so that we could have a version of the script done before day 1.

The second script where for the external communications part. There was a legal requirement that no external communication could be done with the old company email after the announced acquisition. With the groups synced we could map those to disable external sending of mails in Exchange Online.
Here, I stared simple, imported the CSV, a foreach loop ran and set a forwarding rule in Exchange Online on the old mailbox to the new mailbox, when that was done it went over to Teams and set a new external access policy, a new meeting policy and a new channel policy.

The third script was all about the guest accounts that needed to be created. This was to solve a issue with OneDrive migration since it was decided it was going to be done by the users themselves. The first thing here was to create a group, that group then was excluded from a Conditional Access policy due to limitations on OneDrives ability to prompt a guest user for MFA. The script used the same CSV file as before and looked on the new company email field and just invited them to the tenant with the command New-MgInvitation, during the testing phase I discovered that inviting a user to Entra ID takes a while since I ran into issues adding the invited users to the group that excluded them from MFA, my discovery is that it takes about 45 seconds after the invite until the user is visible via Graph that they could be added to the group.

## The automation part
Before the automation of everything above it was done manually each day from day 1 to day 7 I would say. 
It all started with an automation to deliver a CSV file so that we could run the scripts above, I did not have the intention to do any kind of manual work regarding this, so I started to develop an automation within the Azure Automation account that I had access to. 

I started to combine all of the scripts that were done with the CSV file and added a webhook. The webhook was quite a nice touch since we then can define a header when in the code and if the header is missing or is wrong the automation will come to a halt.

During the time I was coding, we discovered that the solution to deliver out of office messages was not working properly and they wanted me to set an automatic reply on every mailbox that was in the acquisition, so I did a simple fix for the day 1 users and took that and put it in with the rest of the code. 

After about 8-10 hours of coding and coffee, I had a solution to the manual process. It was not perfect by any means but It was working. During the testing phase I used the CSV files that we received and sent them to the webhook manually before the whole process was automated. 
I would import it into a snippet here but that would take too much space, so a link to the "final" version that is no where near the final version since its always evolving, some information has been redacted from the script due to obvious reasons. If you like to have a look at the script, its here [New-Userflow](https://github.com/jdenka/Powershell-script-collection/blob/main/New-Userflow.ps1)

All the required permissions to run the automation are the following starting with Graph permissions. User.Read.All, Group.ReadWrite.All, Mail.Send, User.Invite.All, TeamSettings.ReadWrite.All. Office 365 Exchange Online permissions are Exchange.ManageAsApp and if that was not enough due to some limitations it requires you to assign some Entra ID roles as well, namely Teams Administrator and Exchange Recipient Administrator. 