# Starbeam Design

![Starbeam](assets/starbeam-logo.svg)

Starbeam is a lean Datastar SDK for Clojerl applications served by Cowboy. Its core responsibility is narrow: read Datastar signals from Cowboy requests and write ADR-compliant Datastar events to Cowboy streaming responses.

## Goals

- Fully implement the Datastar SDK ADR.
- Use Cowboy directly, without a generic HTTP abstraction.
- Preserve BEAM-native process ownership and failure behavior.
- Stream iodata without flattening complete responses into copied binaries.
- Remain small enough to understand as a single subsystem.
- Support long-lived query streams, full-view morphs, and command/query separation without embedding an application framework.

## Non-goals

Starbeam does not provide routing, middleware, persistence, pub/sub, HTML templating, browser attributes, application lifecycle, authentication, or a CQRS framework. Applications compose those concerns around the SDK.

## Platform

Initial target:

- Clojerl 0.9.1
- Erlang/OTP 27 or newer
- Cowboy 2.13

OTP provides JSON encoding and decoding. Cowboy provides request parsing, response headers, and streamed response bodies. HTML renderers remain optional application dependencies.

## Public API

The initial namespace is `starbeam.core`.

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

`open!` starts the Cowboy streaming response and returns an opaque stream handle. `request` exposes the updated Cowboy request that a handler must return to Cowboy. Event operations return `:ok` or propagate an Erlang/Clojerl error. `close!` finishes the response body.

`read-signals` returns `[signals updated-request]`. Cowboy request values are immutable, and body reads return a new request value that subsequent response operations must use.

## Stream handle

A stream handle contains:

- Cowboy request after `stream_reply`
- PID that opened the stream
- Open/closed state where required by Cowboy interaction

Only the owner process may write. This process-confinement rule provides deterministic event ordering without locks. Other processes notify the owner through normal BEAM messages.

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

Cowboy's streamed reply starts the response immediately. Starbeam does not add buffering, compression, proxy, or cache-vendor headers.

## Event framing

A private event writer builds one iodata frame and sends it with one Cowboy `stream_body` operation.

Frame order is fixed:

1. `event`
2. optional `id`
3. optional non-default `retry`
4. one `data` field per payload line
5. blank line terminating the event

The default retry duration is 1000 milliseconds and is omitted from the wire. Building and sending a complete frame in one operation prevents byte-level interleaving.

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

Default-valued options are omitted. `elements` may be a binary or iodata. Every supplied top-level item must be a complete element. Starbeam does not parse or validate HTML.

## Signal patches

`patch-signals!` emits `datastar-patch-signals` using RFC 7386 JSON Merge Patch semantics.

Options:

```clojure
{:only-if-missing? true
 :event-id "signals-9"
 :retry-duration 2000}
```

The function accepts pre-encoded JSON or an Erlang JSON term supported by OTP's JSON encoder. Pre-encoded input is sent without key conversion or structural rewriting.

## Script execution

`execute-script!` creates a script element and sends it as an element patch with:

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

Automatic removal adds `data-effect="el.remove()"`. Script content is developer-authored code; Starbeam does not evaluate or interpolate it.

## Reading signals

Signal location depends on method:

| Method | Location |
| --- | --- |
| GET | URL-decoded `datastar` query parameter |
| DELETE | URL-decoded `datastar` query parameter |
| POST | JSON request body |
| PUT | JSON request body |
| PATCH | JSON request body |

Body reads continue until Cowboy returns the final chunk. Malformed JSON, missing signal data, unsupported methods, and body read failures produce descriptive exceptions.

Decoded JSON remains in Erlang-native form: maps with binary keys, lists, binaries, numbers, booleans, and the JSON null atom. Automatic conversion to a second collection representation would duplicate the decoded object graph.

## CQRS and fat morph support

Starbeam stays transport-focused but permits this application flow:

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

A subscription sends current state immediately after connecting. Notifications indicate that state may have changed; they are not treated as UI deltas. Reconnection therefore needs only a fresh render, not event replay.

Applications may coalesce repeated notifications before re-reading state. Shared immutable rendered binaries may be reused across connections when users share the same view. Neither behavior belongs in SDK core.

## Performance model

- Build frames as iodata.
- Send each event with one streaming operation.
- Avoid response-wide buffers.
- Avoid recursive conversion of decoded JSON.
- Keep compression outside SDK core.
- Keep each long-lived response owned by one BEAM process.

## Verification

Verification has three layers:

1. Pure event tests compare exact bytes for defaults, all options, multiline payloads, scripts, and invalid values.
2. Cowboy integration tests verify headers, body streaming, all signal-bearing methods, chunked bodies, closure, and disconnect behavior.
3. Datastar's external SDK conformance runner exercises the public `/test` endpoint over real HTTP.

A CQRS example provides an end-to-end smoke scenario with two subscribers, a command returning 204, a full-view update, and reconnect recovery.

## References

- [Datastar SDK ADR](https://github.com/starfederation/datastar/blob/develop/sdk/ADR.md)
- [Cowboy request API](https://github.com/ninenines/cowboy/blob/2.13.0/src/cowboy_req.erl)
- [Clojerl](https://github.com/antlobach/clojerl)
- [Erlang JSON](https://www.erlang.org/doc/apps/stdlib/json.html)
