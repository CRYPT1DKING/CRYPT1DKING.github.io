---
title: Messages
---

## Message Structure

| Bytes | Byte 1 | Byte 4 | Byte 3 | Byte 4 | Byte 5-x | Byte x+1 | Byte x+2 |
| ----- | ------ | ------ | ------ | ------ | -------- | -------- | -------- |
| Contents | 'A' | 'Z' | sender ID | receiver ID | Message contents | 'Y' | 'B' |

## Team ID

| Name | Tyler | Alex | Frank | Luis |
| ---- | ----- | ---- | ----- | ---- |
| ID | c | a | d | b |

## Messages Sent

| Bytes | Byte 4 |
| ----- | ------ |
| Variable Name | Motor Speed |
| Variable Type | uint8_t |
| Min Value | 0 |
| Max Value| 127 |
| Example | 27 |

## Only receiving broadcast messages