---
layout: post
title: Making Global Administrators safer
categories:
- Entra ID
- Security
tags:
- entra id
- security
pin: true
date: 2023-12-20 11:42 +0100
---
# Making Global Administrators safer 
In the current landscape with threats lurking all over the place I decided to write about how we can make the use of Global Administrators and other high privileged roles safer, what I am about to share is going to make your high privileged roles require to use a Phishing-resistant MFA method if you are licensed for it, so either Windows Hello for Business, FIDO2 security keys or Certificate-based authentication.  

In this post Im going to cover what you can do if you have Entra ID Free, Entra ID P1 or Entra ID P2 licensing.  

## First things to take into consideration
If you have a hybrid identity environment and use either Microsoft Entra Connect or Microsoft Entra Cloud Sync. If you currently have Active Directory and Microsoft Entra ID and don't synchronize your identities you should probably start doing that.  

You should follow the best practice advice and don't synchronize accounts to Entra ID that have high privileges in Active Directory this also goes the other way around, your high privileged accounts in Entra ID should NOT be synchronized from Active Directory.  

There is a whole article dedicated to identity management and access control security best practices over [here](https://learn.microsoft.com/en-us/azure/security/fundamentals/identity-management-best-practices)

The article explains the following:
- What the best practice is
- Why you want to enable that best practice
- What might be the result if you fail to enable the best practice
- Possible alternatives to the best practice
- How you can learn to enable the best practice


## Entra ID Free licensing
If you are stuck with Entra ID Free you only have one option, that is to use Security Defaults. So now you question might be "What is Security Defaults?"  

To explain security defaults in a simple way is that Microsoft has made preconfigured security settings that help organizations that are not licensed for Entra ID P1 or Entra ID P2 license. 
Security defaults will require all users in the organization to register for multifactor authentication.

IF your tenant is created before October 2019 you probably don't have security defaults enabled if you have not enabled it. There is a simple guide on how to enable it over [here](https://learn.microsoft.com/en-us/entra/fundamentals/security-defaults#enabling-security-defaults)  

Even if you tenant is created after October 2019 I would suggest to have a look if its enabled or not.


## Entra ID P1 licensing
> Before configuring any of this, please make sure your administrators have registered a Phishing-Resistant MFA method.
> Otherwise they will be locked out and have no administrative access to Entra ID.
{: .prompt-warning }
If you have Entra ID P1 and don't use Conditional Access yet, now it's a great time to start.
Before we begin this journey we need to see that our tenant had the capabilities we need to go over our Authentication methods. 

![Authentication Methods](/assets/img/2023-12/2023-12-20_AuthMethods.png)

This image only shows the FIDO2 configuration but there is also options for Certificate-based authentication and that I will not go over in this post for now. 
The second step is to check our Authentication strengths and configure what kinds of Phishing-resistant MFA we would like our high privileged roles to be able to use, in my case I have added one that is only for FIDO2

![Authentication Strengths](/assets/img/2023-12/2023-12-20_AuthStrengths.png)

Now armed with this we can now go a head and setup our Conditional Access policy for the high privileged roles. 
Our new Conditional Access policy should look something like this if you only are going to protect the Global Administrator role. 

![Conditional Access Policy Part 1](/assets/img/2023-12/2023-12-20_EIDP1_CA_P1.png) 

And the grant control for the policy should look like this.

![Conditional Access Policy Part 2](/assets/img/2023-12/2023-12-20_EIDP1_CA_P2.png)

This if only if you want to create your own policy and define what roles are being protected and what authentication methods that are allowed yourself. If you want to go the route that Microsoft suggest you should probably create a new policy from template, the correct template to use against this is under "Emerging threats" and is called "Require phishing-resistant multifactor authentication for admins".  

That template will allow for any Phishing-resistant MFA and will secure the following roles, Global Administrator, Security Administrator, SharePoint Administrator, Exchange Administrator, Conditional Access Administrator, Helpdesk Administrator, Billing Administrator, User Administrator, Authentication Administrator, Application Administrator, Cloud Application Administrator, Password Administrator, Privileged Authentication Administrator and Privileged Role Administrator

> Remember to remove yourself from the exclusion in the policy if you decide to use the template. But only after you have registered a FIDO2 security key.
{: .prompt-warning }

When the user with Global Administrator role or some other role sign in they will be prompted to sign in with a security key. 

## Entra ID P2 licensing
> Before configuring any of this, please make sure your administrators have registered a Phishing-Resistant MFA method.
> Otherwise they will be locked out and have no administrative access to Entra ID.
{: .prompt-warning }
With Entra ID P2 licensing you can use the same solution as the one for Entra ID P1. But to be realistic the most probable cause of you to have Entra ID P2 is to use the security features and then you are probably using some parts of the Identity Governance features. 
One feature thats hopefully is being used is the Privileged Identity Management part where you can assign both users to be eligible to activate high privileged roles instead of having them active constantly. 

If you have Entra ID P2 and have not looked at Privileged Identity Management you should consider it and read more about it [here](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)

The next steps require you to have enabled Entra ID PIM. 
First we have to create whats called an Authentication Context, that we do under the Conditional Access pane. We name it and we give it a description and make sure to publish it to apps, we also select a free ID for the context. In my case it looks like this.

![Conditional Access Authentication contexts](/assets/img/2023-12/2023-12-20_AuthContext.png)

With the authentication context created we can head over to Privileged Identity Management under the Identity governance selection. Under manage we click on Microsoft Entra roles and under manage again we select Roles. Now we can search up the role we want to make changes to here I will showcase the Global Administrator role so we select that and head over to Role settings and select edit.

![Privileged Identity Management role settings Global Administrator](/assets/img/2023-12/2023-12-20_PIM_Role_Settings.png)

In the edit menu we change the "On activation, require" and we select Microsoft Entra Conditional Access authentication context and we select the authentication context we created in the previous step. 
> Note that this is not a production environment and I would suggest that you change the duration and that you require a justification. Some might even require ticket information and approvers.
{: .prompt-info }

When this is configured we head over to Conditional Access and prepare a new policy.
The big difference between the new policy and the one for the Entra ID P1 is that this will target all users and just not the roles that we specify and of course make use of the previously created authentication context. The new policy will look like this. 

![Conditional Access Policy for Authentication context](/assets/img/2023-12/2023-12-20_EIDP2_CA.png)

It will have the same grant as the policy for Entra ID P1 making use of the setting "Require authentication strength" and our own created one in my case FIDO2 or the Microsoft managed one called Phishing-resistant MFA.  

The admin can sign in with any kind of multifactor authentication but when they go to active the role that requires a FIDO2 security key or any of the other Phishing-resistant MFAs and they used a lower kind of authentication they will be prompted with this message: 

![Privileged Identity Management activation warning](/assets/img/2023-12/2023-12-20_PIM_Activation.png)

When they click the warning they will be asked to sign in with the authentication required to activate the role.

## Conclusions 
This is a great way to secure your high privileged roles depending on your current licensing setup. 
There are more ways to make it even more secure but that will make it into another post in the future, stay safe out there!