---
title: HPACK
slug: Web/HTTP/HTTP2/HPACK
tags:
  - Guide
  - HTTP
  - WebMechanics
---
{{HTTPSidebar}}

If the payload of HTTP messages could be compressed using gzip or brotli, HTTP headers weren't. They can be larger than thought (Think about the Cookie: header) and some are often repeated identically on all requests on a given page.

HTTP/2 solves this problems by using the **TODO: actual name** (HPACK) algorithm to compress the headers. This algorithm has been especially designed for it, to solve security risks like those from the CRIME attack, and is not designed to evolve in the future.

## How headers are coded in an HTTP Message

## Three compression methods

### Static dictionary

### Hufmann dictionary

### ...

## Linking to previous messages

## Evolution

## See also
