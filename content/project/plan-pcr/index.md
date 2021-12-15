---
author: Kai
categories:
- shiny
date: "2021-12-14"
draft: false
featured: true
excerpt: A Shiny app for planning qPCR experiments reproducibly
layout: single-sidebar
tags:
- shiny
links:
- icon: door-open
  icon_pack: fas
  name: App
  url: https://kai-a.shinyapps.io/plan-pcr/
- icon: github
  icon_pack: fab
  name: source
  url: https://github.com/KaiAragaki/plan-pcr
title: plan-pcr
---

# Overview

plan-pcr is a shiny app meant to help those performing comparative (ddCt) qPCR with [this protocol](https://bookdown.org/adamaragaki/public_knowledge/deltadelta-qrt-pcr.html) (or similar).

Below is a screenshot:

![A screenshot of the plan-pcr Shiny app](./plate-overview-screenshot.png)

It has features that can help you:

- Calculate the amount of mastermix needed
- Dilute your samples
- Lay out your samples/primers
- Create a report for your records