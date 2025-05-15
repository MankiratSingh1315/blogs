---
title: ABI Compliance Reporting for PostgreSQL - The Plan - Part 1
date: 2025-05-12
author: Mankirat Singh
description: This blog introduces the motivation and initial plan for implementing an automated ABI compliance reporting system for PostgreSQL. I have talked about ABIs, ABI stability, recent incidents highlighting the need for automation, and explores the constraints and requirements for such a tool.
tags: ["PostgreSQL", "abicc"]
---

## Who am I?
I am **Mankirat Singh**, a 3rd Year Computer Engineering undergrad from India, passionate about building any piece of tech. This year, I will work with PostgreSQL to develop an **ABI Compliance reporting system** for the official PostgreSQL Repository under the **Google Summer of Code 2025** Program.

## What is ABI and ABI Compliance Checking?
### ABI?
ABI stands for **Application Binary Interface**. It is a set of rules and conventions that define how compiled code interacts with other compiled code at the binary level; This includes details about data types, function calling conventions, and binary formats.

### ABI Compliance Checking?
ABI compliance checking is the process of verifying that a binary interface remains consistent across different versions of a software library or application.

## Why PostgreSQL needs an Automated ABI Compliance Checker?
PostgreSQL introduced [Server API and ABI Stability Guidance](https://www.postgresql.org/docs/devel/xfunc-c.html#XFUNC-API-ABI-STABILITY-GUIDANCE) to help extension authors understand compatibility expectations. The policy ensures ABI stability between minor releases (e.g., 17.0 to 17.1) but allows changes in major versions. Committers rely on manual reviews to enforce this, occasionally leading to unintended ABI changes in a minor release. A recent example occurred in PostgreSQL 17.1, where an ABI-breaking change was shipped and later reversed in 17.2. This incident highlights the need for an automated process to catch such issues before release.

## Constraints for this tool
ABIs depend on various factors due to which this tool also depends on several constraints like:
 - Cross Compiler Support.
 - Cross Platform Support.
 - Should compare commits to generate report of ABI changes between commits. 
 - Generate detailed ABI compliance reports for easy preview.
 - Inform the contributors if any ABI instability is detected.
 - Should be able to generate reports for various PostgreSQL versions in parallel.

## Implementation

### Git Receive hooks?
The very initial thought I got for implementation was to make something from scratch using git recieve hooks which will fire ABI compliance reporting flow whenever a new commit is made and inform for any breaking changes but as per PostgreSQL core contributors, Anything that slows down the process of pushing to the server is rejected out of hand, and other proposals for git receive hooks have been rejected in the past.

### PostgreSQL Build Farm?
Written in Perl, the [build farm](https://buildfarm.postgresql.org/) is a very beautiful piece of software which is being used in the PostgreSQL community for a very long time. It is a distributed system for automatically testing changes in the source code for PostgreSQL as they occur, on a wide variety of platforms. 
#### Build Farm Client
The individual clients that run the tests are called "members" or "animals". Each member is very very customisable which depends on a single config file.The config file has a large variety of options like `branches_to_build`, parallel running, `mail_events`, C and C++ flags to run build with.<br>
Volunteers Join the Build Farm members list to have the build farm client run on their machines on cron jobs and perform regular testing on newer commits(if there are some) and send the results to the build farm central server.<br>
The client machines could be using any compiler, operating system and any specific version of these of the volunteers' choice to perform testing on PostgreSQL which makes build farm a real distributed system.
#### Build Farm Server
The build farm server is responsible for receiving the testing results from authorised clients, processing and viewing them on the web and sending email notifications to the people on the mailing list.
#### Why to use the build farm for ABI compliance reporting?
As all the above mentioned features are already present in the build farm which resonate the requirements of the tool, and it also have the possibility of creating extensions for build farm client which made build farm a perfect choice to implement this tool on.

# How build farm will be used for this tool?
Dropping soon in part 2 of this blog :)<br>
Thanks for reading!