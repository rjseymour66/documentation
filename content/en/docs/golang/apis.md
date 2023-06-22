---
title: "APIs"
weight: 210
description: >
  Implementation details for application programming interfaces (APIs).
---

## Versioning

You can version your API one of two ways:
- Prefix all URLs with the version:
  ```
  /v1/healthcheck
  ```
- Use custom `Accept` and `Content-Type` headers.