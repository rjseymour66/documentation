---
title: "Context"
weight: 200
description: >
  Working with the context pacakge.
---

A function that accepts a context should always check if the context is canceled.

`Background` creates a non-cancelable context. Then, you can derive a cancelable timeout context using the `WithTimeout` function.

The signal package has a NotifyContext function to catch an OS signal.