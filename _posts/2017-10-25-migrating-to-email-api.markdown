---
layout: post
title: "Migrating to the new Email Search API"
description: "If you use the Search API to map users by an email address, it's time to move to the new Email Search API"
date: 2017-10-24 08:30
category: Technical Guide, Email Search API
author:
  name: "Chris Moyer"
  url: "https://blog.coredumped.org
  mail: "kopertop@gmail.com"
  avatar: "https://en.gravatar.com/avatar/54d15d5d1700546270365e272098ca48?s=200"
tags:
- Email Search
- API
- Authentication Workflow
- New Features
---

One very common use case for the Auth0 Search API is to identify users by email address, merging accounts together, or linking existing accounts based on a shared email address. In the case of my company, Cannabiz Media, we also used this API to identify new users and create accounts when a user signed up, de-duplicating by email address to prevent users from having multiple accounts.

## Introduction to the Search API

* Why using the Search API for authentication workflow is bad
* Use-cases for mapping by email
* How the new Email Search API helps

## Introduction to the Email Search API

* Why you should migrate to the Email Search API
* What code to replace, and how to migrate from the Search API
* Limitations, things that it can not be used for

## How Cannabiz Media uses the new Email Search API

* How we made the switch to the Email Search API
* Performance/Reliability improvements
* How we connect with WooCommerce Webhooks


## Conclusion

* Search was a big pitfall, most issues were related to it
* Email Search is more limitted, but fits the general use case
* Doesn't need to be used just for the auth-workflow, can also be used for generalized lookups

