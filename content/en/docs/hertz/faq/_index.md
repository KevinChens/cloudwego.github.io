---
title: "FAQ"
linkTitle: "FAQ"
weight: 4
description: >
---

## High Memory Usage

### Connections not Closing due to Client Non-standard Usage

If the client initiates a large number of connections without closing them, there will be a large waste of resources in extreme cases, which can cause high memory usage problems over time.

**Solution**

Configure `idleTimeout` reasonably. Hertz Server will close the connection to ensure the stability of the server after the timeout. The default configuration is 3 minutes.

### Vast Request/Response

1. If the request and response are very large, the data will all enter the memory, causing great pressure on it, when stream and chunk are not used.
2. The streaming under the netpoll network library is a fake streaming. Since netpoll uses the LT trigger mode, it will trigger netpoll to read data when data arrives. The `Reader` interface is not implemented in terms of interface design. To enable streaming, Hertz encapsulates netpoll as a Reader, but the data still enters memory uncontrollably, so in a case of very vast streaming requests, memory pressure can result.

**Solution**

For very vast requests cases, use a combination of streaming and go net.

## Common Error Code Checking

If the framework reports the following error codes, you can check it for possible causes. If there is an error code other than the following, the error code is not caused by the framework and needs to be located by the user to see whether it is set by itself or by some middleware.

### 404

1. Access to the wrong port, commonly access to the debug port.
   1. Solution: Distinguish between listening port for framework service and listening port for debug server, the default is 8888.
2. No routes matched
   1. Check whether all expected routes are registered correctly based on the startup log.
   2. Check that the access method is correct.

### 417

The server returns `false` after executing the custom `ContinueHandler` (the server actively rejects the subsequent body of the 100 Continue).

### 500

1. Throwing the panic in middleware or in `handlerFunc`.
   1. Solution: Locate specific problems with panic stack information.
2. In the fs case, the path carries `/../`, and unexpected files may be accessed. The error log in server app log: `cannot serve path with '/../' at position %d due to security reasons: %q`.
   1. Solution: Check for illegal requests.

## Context Guide

### Description

Hertz also provides a standard `context.Content` and a request context as input arguments to the function in the `HandleFunc` Design. The `handler/middleware` function signature is:

```go
type HandlerFunc func(c context.Context, ctx *RequestContext)
```

### Metadata Storage

Both contexts have the ability to store values, which is a simple principle for choosing which one to use: the life cycle of the stored value and the selected context to match.

**Specifics**

The `ctx` is primarily used to store request-level variables, which are recycled after the request ends. It is characterized by high query efficiency (the bottom is `map`), unsafe coroutines and the `context.Context` Interface is not implemented.
`c` is passed as the context between middleware `/handler`. With all the semantics of `context.Content` and safe coroutine. All that requires the `context.Content` interface as input arguments, just pass `c` directly.

In addition, Hertz also provides the `ctx.Copy()` interface to make it easier for businesses to obtain a copy of the safe coroutine if they are faced with cases where they must pass `ctx` asynchronously.
