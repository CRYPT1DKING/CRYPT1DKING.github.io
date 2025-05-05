---
title: Messages
---

## Message Structure

| Bytes | Byte 1 | Byte 2 | Byte 3 (char) | Byte 4 (char) | Byte 5 (int8)| Byte 6-x | Byte x+1 | Byte x+2 |
| ----- | ------ | ------ | ------ | ------ | ------ | -------- | -------- | -------- |
| Contents | 'A' | 'Z' | sender ID | receiver ID | Message ID | Message contents | 'Y' | 'B' |

## Team ID

| Name | Tyler | Alex | Frank | Luis |
| ---- | ----- | ---- | ----- | ---- |
| ID | c | a | d | b |

## Messages Sent

| Bytes | Byte 5 |
| ----- | ------ |
| Message ID | 2 |
| Variable Name | Motor Speed |
| Variable Type | uint8_t |
| Min Value | 0 |
| Max Value| 100 |
| Example | 27 |

## Messages Recieved

| Bytes | Byte 5 |
| ----- | ------ |
| Message ID | 3 |
| Variable Name | Target RPM |
| Variable Type | uint8_t |
| Min Value | 0 |
| Max Value| 100 |
| Example | 50 |