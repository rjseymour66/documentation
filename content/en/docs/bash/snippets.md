---
title: "Snippets"
# weight: 10
description: >
  Expressions or code blocks that complete specific tasks.
---

## Text

### Uppercase or lowercase

Use the `tr` (translate) command to change the case of a character set:

```bash
uppercase=$(echo $var | tr '[a-z]' '[A-Z]')

lowercase=$(echo $var | tr '[A-Z]' '[a-z]')
```