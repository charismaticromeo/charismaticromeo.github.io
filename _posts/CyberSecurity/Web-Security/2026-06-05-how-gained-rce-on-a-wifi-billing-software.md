---
layout: post
title: "How I Hacked a WiFi Billing System"
description: "A technical write-up documenting the discovery, validation, and disclosure of a command injection
vulnerability in a WiFi billing system."
image: /assets/images/90af0d69-989b-42da-9323-02650e153afd.jpeg
date: 2026-06-05
categories:
  - Write-Ups
  - CyberSecurity
  - Web-Security
tags:
  - Web Security
  - cybersecurity
---

## Introduction

Injection vulnerabilities - from query injections to command injections - are among the worst nightmare for developers
and maintainers but a sweetheart for hackers.

In this article, I'll walk through how I discovered a command injection vulnerability in a WiFi billing system that
ultimately led to an authenticated root shell. But first before diving into the technical details, let's get some
background story behind the discovery.

If you're only interested in the exploitation and analysis, feel free to skip ahead to part 2.

> **Disclaimer**: 
> 
> This article is intended to document the discovery process, impact assessment, and remediation considerations. It is
> not intended as a guide for abusing vulnerable systems. If you use this knowledge to do harm I can't be held to 
> account for your actions.
{: .prompt-warning}

## Background

Full disclosure: I wasn't out hunting bow and arrow when I found this bug. 

My uncle had reached out after watching a reel about the internet reselling business and wanted to know what it would
take to get setup. One of the requirements mentioned a billing system, and he asked whether I could build one. My short
answer was Yes.

Before writing anything from scratch, I started exploring some of the open-source solutions available locally. One
project in particular caught my attention; **The Reduzer WiFi Billing System**. I had seen it's author talk about it on
**X**, I guess this is one of the importances of talking about your work.

My goal was to see what the existing solutions offered, evaluate whether they met the requirements, and identify any
gaps that would need custom development. I wasn't looking to reinvent the wheel.

![IQ distribution meme with cartoon faces](/assets/images/IQ-distribution-meme-with-cartoon-faces.png)

## Discovery

The first step was to clone the [Reduzer WiFi billing
repository](https://github.com/reduzersolutions/radius-billing-system.git) and inspect the source code for myself.

A quick glance revealed that the project is largely a wrapper around existing open-source technologies. As its core, it
integrates solutions such as **PHPNuxBill** and **RADIUS** server-client architecture, while adding **M-Pesa Daraja**
integration and orcherstration logic to tie the components together into a deployable system - kudos to the author.

To understand how everything fit together, I opened Neovim and started where most developers do: at the `README` file. Once I
had a rough idea of the installation and deployment process, I began working my way through the codebase one directory
at a time.

There's a rule I try to follow whenever I'm evaluating software from the internet: *never run code you haven't taken the
time to understand*. Even when the source is public, I prefer to verify that the application does what it claims to do
before deploying it anywhere.

It didn't take long before I came across something that immediately caught my attention; user-controlled input appearing
to flow directly into PHP's `exec()` function without any sanitization.
![Coa_handler.php](/assets/images/Screenshot From 2026-04-30 09-56-51.png) 

As shown in the screenshot above, the `coa_handler.php` endpoint accepts several parameters through a POST request,
including **username**, **type**, **nasIP**, **secret**, and **attributes**. Tracing the execution flow revealed that
these values were used to construct a shell command that was later passed to `exec()`.

At this point, I had not yet confirmed exploitability, but the pattern was concerning. Whenever user-controlled input
reaches a shell execution function without proper validation or escaping, it raises the possibility of command injection
\- a vulnerability class capable of escalating into full remote code execution under the right conditions.

## Validation

![Live Exploit Screen](/assets/images/Screenshot From 2026-04-30 14-52-33.png)

To validate my findings, I set up a local test environment and reproduced the application's behavior under controlled
conditions. The installation process is outside the scope of this write-up, but readers interested in reproducing the
findings can refer to the project's repository for setup instructions.

Based on my analysis of the `coa_handler.php` endpoint, a legitimate POST request followed the structure shown below:

```bash
$ curl -X POST http://172.21.16.1:7080 -H "Content-Type: application/json" -d '{"username": "joe", "type": "disconnect",
"nas_ip": "127.0.0.1", "secret": "mysupersecret"}' 
```
{: file="A Valid POST Request" }

The supplied values were used to construct a radclient command responsible for sending CoA (Change of Authorization) and
disconnect requests to the RADIUS server.

Depending on the request type and the validity of the supplied secret, the server would process the operation and
respond accordingly.

While tracing the request flow, I identified a separate issue: the coa_handler.php endpoint lacked access-control
checks. As a result, unauthenticated users could interact with the endpoint directly. This meant that an attacker could
reach the vulnerable code path without first having to authenticate to the application.

With a potentially injectable command sink identified and a reachable attack surface confirmed, the next step was to
determine whether arbitrary command execution was possible.

My initial proof-of-concept focused on executing a simple system command to verify exploitability.

```bash
$ curl -X POST http://172.21.16.1:7080 -H "Content-Type: application/json" -d '{"username": "joe'\''; id; #", "type":
disconnect", "nas_ip": "127.0.0.1", "secret": "mysupersecret"}'
```
{: file="POC payload executing id command"}

> **Note:**
> Multiple parameters appeared to be affected by the underling vulnerability. For demonstration purposes, I focused on a
> single injection point during testing.
{: .prompt-warning}

To streamline testing and validation, I developed a small Python proof-of-concept that automated the requests and
allowed me to observe the application's behavior more efficiently.
![Python POC](assets/images/Screenshot From 2026-04-30 13-31-39.png) 

## Impact Assessment

> **Disclaimer:**
> 
> My assessment was limited to a laboratory environment running a single workstation. I did not have access to
> production deployments or additional networking equipment, such as routers and access points, that would have allowed
> me to simulate a more realistic deployment architecture. As a result, the impact described below reflects what I was
> able to verify directly.
{: .prompt-warning}

The vulnerability allows an unauthenticated attacker to execute arbitrary system commands within the `coa-service`
container.
During testing, I confirmed that command execution was possible without authentication and that the affected container
had connectivity to the internal Docker network. Under these conditions, an attacker may be able to:

  - Interact with internal services exposed on the backend network, such as databases, RADIUS components, or other
    application services.
  - Perform network reconnaissance from within the container to identify reachable hosts and services.
  - Access sensitive configuration files, environment variables, and application secrets available to the container.
  - Modify application files within the container, potentially allowing the insertion of malicious code or
    credential-harvesting functionality.
  - Establish persistence within the affected container.


## Advisory

To remediate this vulnerability and reduce the likelihood of similar issues in the future, the following measures are
recommended:

  1. **Validate and sanitize all user-supplied input.** User-controlled data should never be passed directly to shell
     execution function. Where command execution is unavoidable, use parameterized APIs or proper escaping mechanisms
     and enforce strict input validation.
  1. **Require authentication and authorization.** Access to the `coa_handler.php` endpoint should be restricted to
     authenticated users and trusted services. Appropriate authorization checks should be implemented to ensure that
     only permitted users can perform CoA operations.
  1. **Restrict network exposure.** The service should not be publicly accessible unless absolutely necessary. Where
     possible, limit access to trusted hosts, internal networks, or specific application components.
  1. **Apply the principle of least privilege.** Containers and services should run with only the permissions required
     for their intended function. Reducing privileges limits the potential impact of a successful compromise.
  1. **Avoid constructing shell commands from user input.** Consider replacing shell-based interactions with native
     libraries or APIs where possible. Eliminating shell invocation entirely removes a significant attack surface and
     prevents command injection vulnerabilities by design.

## Conclusion

I reported this vulnerability to the project maintainers on 30 April 2026. On 4 May 2026, I received a response
acknowledging receipt of the report. During the discussion, the maintainers explained that the project had already been
abandoned in favor of a newer solution. They also indicated that they would take the necessary action, and shortly
thereafter the repository was archived.

From a technical perspective, this vulnerability serves as a reminder of how seemingly small implementation decisions
can introduce significant security risks. In this case, user-controlled input was allowed to reach a shell execution
function without adequate validation or safeguards, ultimately resulting in arbitrary command execution. What began as a
routine review of an open-source project quickly escalated into a vulnerability capable of compromising the entire
service. More broadly, this experience reinforced two lessons for me.

The first is the value of sharing your work publicly. Had the project not been discussed openly and made accessible to
the community, I likely would never have encountered it in the first place. You never really know who is watching,
evaluating, or building upon what you create.

The second is the importance of understanding the software you depend on. Whether the code is open source or
proprietary, taking the time to review how a system works before deploying it can uncover limitations, design flaws, and
sometimes critical security issues that would otherwise go unnoticed.
For me, what started as research into an existing WiFi billing solution turned into a vulnerability discovery, a
responsible disclosure, and ultimately a valuable reminder that trust in software should always be accompanied by
verification.
