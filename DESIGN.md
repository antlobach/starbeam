# Starbeam Design

![Starbeam](assets/starbeam-logo-v4.png)

Starbeam reads Datastar signals from Cowboy requests and writes Datastar events to Cowboy streaming responses.

## Goals

- Implement the Datastar SDK event contract.
- Use Cowboy without another HTTP abstraction.
- Preserve BEAM process ownership and failure behavior.
- Build streamed responses as iodata.
- Keep routing, rendering, persistence, and application lifecycle outside the SDK.
- Support long-lived query streams and command/query separation.

## Non-goals

Starbeam excludes routing, middleware, persistence, pub/sub, HTML templating, browser attributes, application lifecycle, authentication, and CQRS orchestration.

## Platform

Supported versions:

- Clojerl 0.9.1
- Erlang/OTP 27 or 28
- Cowboy 2.13
- Datastar 1.0.2

OTP handles JSON. Cowboy handles requests and streamed responses. Applications choose their HTML renderer.

## Public API

The public API lives in `starbeam.core`.

```clojure
(open! request)
(request stream)
(patch-elements! stream elements)
(patch-elements! stream elements options)
(patch-signals! stream signals)
(patch-signals! stream signals options)
(execute-script! stream script)
(execute-script! stream script options)
(read-signals request)
(close! stream)
```

`open!` starts a Cowboy streaming response and returns an opaque handle. `request` returns the updated Cowboy request for the handler response. Event operations return `:ok` or propagate the underlying error. `close!` finishes the body.

`read-signals` returns `[signals updated-request]`. Use the returned request after every body read.

## Stream handle

A stream handle contains:

- Cowboy request after `stream_reply`
- PID that opened the stream
- Open/closed state where required by Cowboy interaction

Only the owner process may write. Other processes send BEAM messages to the owner. This preserves event order without locks.

## Response initialization

`open!` sends status `200` with:

```text
Cache-Control: no-cache
Content-Type: text/event-stream
```

For HTTP/1.1 only:

```text
Connection: keep-alive
```

Cowboy starts the response immediately. Starbeam adds no buffering, compression, proxy, or vendor-specific cache headers.

## Event framing

The event writer builds one iodata frame and sends it with one Cowboy `stream_body` operation.

Frame order is fixed:

1. `event`
2. optional `id`
3. optional non-default `retry`
4. one `data` field per payload line
5. blank line terminating the event

The default retry duration is 1000 milliseconds and stays off the wire. One send per frame prevents byte-level interleaving.

## Element patches

`patch-elements!` emits `datastar-patch-elements`.

Options:

```clojure
{:selector "#app"
 :mode :outer
 :use-view-transition? false
 :view-transition-selector "#app"
 :namespace :html
 :event-id "view-42"
 :retry-duration 2000}
```

Modes:

- `:outer`
- `:inner`
- `:replace`
- `:prepend`
- `:append`
- `:before`
- `:after`
- `:remove`

Namespaces:

- `:html`
- `:svg`
- `:mathml`

Default values stay off the wire. `elements` accepts binaries and iodata. Each top-level item must contain a complete element. Starbeam does not parse HTML.

## Signal patches

`patch-signals!` emits `datastar-patch-signals` with RFC 7386 JSON Merge Patch semantics.

Options:

```clojure
{:only-if-missing? true
 :event-id "signals-9"
 :retry-duration 2000}
```

The function accepts pre-encoded JSON or an Erlang JSON term. Pre-encoded input passes through unchanged.

## Script execution

`execute-script!` sends a script element as an element patch:

```text
mode append
selector body
```

Options:

```clojure
{:auto-remove? true
 :attributes {"type" "application/javascript"}
 :event-id "script-3"
 :retry-duration 2000}
```

Automatic removal adds `data-effect="el.remove()"`. Starbeam sends script content unchanged and never evaluates it.

## Reading signals

Signal location depends on method:

| Method | Location |
| --- | --- |
| GET | URL-decoded `datastar` query parameter |
| DELETE | URL-decoded `datastar` query parameter |
| POST | JSON request body |
| PUT | JSON request body |
| PATCH | JSON request body |

Body reads continue through Cowboy's final chunk. Malformed JSON, missing signals, unsupported methods, and read failures raise exceptions.

Decoded JSON remains an Erlang term. Converting it to another collection type would copy the object graph.

## CQRS integration

One application pattern:

```text
command request
  -> validate
  -> commit state change
  -> notify subscribers
  -> 204 response

subscription request
  -> open SSE stream
  -> read authoritative state
  -> render complete element
  -> patch outer element
  -> wait for notification
  -> repeat
```

Subscriptions send current state on connection. Notifications trigger another read; they carry no UI delta. Reconnection renders current state without event replay.

Applications may coalesce notifications or reuse immutable rendered binaries across equivalent connections. The SDK implements neither policy.

## Performance model

- Build frames as iodata.
- Send each event with one streaming operation.
- Avoid response-wide buffers.
- Avoid recursive conversion of decoded JSON.
- Keep compression outside SDK core.
- Keep each long-lived response owned by one BEAM process.

## Verification

Tests start a real Cowboy listener and cover:

- exact event bytes and response headers
- signal decoding for `GET`, `DELETE`, `POST`, `PUT`, and `PATCH`
- stream ownership, closure, and payload limits
- concurrent CQRS subscribers and reconnect-safe full morphs
- middleware cookie preservation

## References

- [Datastar SDK ADR](https://github.com/starfederation/datastar/blob/develop/sdk/ADR.md)
- [Cowboy request API](https://github.com/ninenines/cowboy/blob/2.13.0/src/cowboy_req.erl)
- [Clojerl](https://github.com/antlobach/clojerl)
- [Erlang JSON](https://www.erlang.org/doc/apps/stdlib/json.html)
