<p align="center">
  <img src="assets/starbeam-logo-v4.png" alt="Starbeam symbol" width="240">
</p>

<p align="center">
  Datastar SDK for Clojerl on the BEAM VM.
</p>

<p align="center">
  <a href="https://github.com/antlobach/starbeam/actions/workflows/ci.yml"><img src="https://github.com/antlobach/starbeam/actions/workflows/ci.yml/badge.svg" alt="CI"></a>
  <a href="src/starbeam.app.src"><img src="https://img.shields.io/badge/version-0.1.0-blue" alt="Version 0.1.0"></a>
  <img src="https://img.shields.io/badge/Erlang%2FOTP-27%20%7C%2028-purple" alt="Erlang/OTP 27 and 28">
</p>

## Requirements

| Component | Version |
| --- | --- |
| Clojerl | 0.9.1 |
| Erlang/OTP | 27 or 28 |
| Cowboy | 2.13 |
| Datastar | 1.0.2 |
| rebar3 | newer than 3.14 |
| rebar3_clojerl | 0.8.8 |

Starbeam pins Clojerl 0.9.1 and supports Erlang/OTP 27 and 28. OTP's built-in `json` module supplies JSON encoding and decoding.

## Get Clojerl

### Build from source

Replace `0.9.1` with the tag shown on the [releases page](https://github.com/antlobach/clojerl/releases/latest) when a newer release exists.

```shell
git clone --depth 1 --branch 0.9.1 https://github.com/antlobach/clojerl.git
cd clojerl
make
make repl
```

See the [Clojerl README](https://github.com/clojerl/clojerl) for platform-specific setup.

## Add Starbeam to a Clojerl project

Add Clojerl, Cowboy, and Starbeam to `rebar.config`:

```erlang
{deps,
 [{clojerl,
   {git, "https://github.com/antlobach/clojerl.git", {tag, "0.9.1"}}},
  {cowboy, "2.13.0"},
  {starbeam,
   {git, "https://github.com/antlobach/starbeam.git", {branch, "main"}}}]}.

{plugins,
 [{rebar3_clojerl, "0.8.8"}]}.
```

Compile the project:

```shell
rebar3 clojerl compile
```

For a new application, install `rebar3_clojerl` globally in `~/.config/rebar3/rebar.config`:

```erlang
{plugins, [{rebar3_clojerl, "0.8.8"}]}.
```

Then create and compile an OTP application:

```shell
rebar3 new clojerl_app demo
cd demo
# Replace the generated rebar.config with the dependencies above.
rebar3 clojerl compile
```

Clojerl application names cannot contain dashes.

## Start the project REPL

Run the REPL from the project root:

```shell
rebar3 clojerl repl
```

Load Starbeam:

```clojure
(require '[starbeam.core :as starbeam])
```

Cowboy calls handler functions directly from Clojerl namespaces. Re-evaluate a `defn` in the running REPL to replace it for subsequent requests.

Project commands:

```shell
rebar3 clojerl compile
rebar3 clojerl test
rebar3 clojerl repl
```

## First stream

Create `src/demo/events.clje`:

```clojure
(ns demo.events
  (:require [starbeam.core :as starbeam]))

(defn init [request state]
  (let [stream (starbeam/open! request)]
    (starbeam/patch-elements!
      stream
      "<main id=\"app\">Hello from Starbeam</main>")
    (starbeam/close! stream)
    #erl[:ok (starbeam/request stream) state]))
```

Compile and start the REPL:

```shell
rebar3 clojerl compile
rebar3 clojerl repl
```

Start Cowboy and route `/events` to the handler:

```clojure
(#erl application/ensure_all_started :cowboy)

(let [handlers (#erl erlang/tuple_to_list
                     #erl[#erl["/events" :demo.events nil]])
      routes (#erl erlang/tuple_to_list
                   #erl[#erl[:_ handlers]])
      dispatch (#erl cowboy_router/compile routes)
      socket-options (#erl erlang/tuple_to_list
                           #erl[#erl[:port 8080]])]
  (#erl cowboy/start_clear
    :starbeam-demo
    #erl{:socket_opts socket-options}
    #erl{:env #erl{:dispatch dispatch}}))
```

Inspect the stream:

```shell
curl -N http://127.0.0.1:8080/events
```

Expected event:

```text
event: datastar-patch-elements
data: elements <main id="app">Hello from Starbeam</main>

```

Stop the listener from the REPL:

```clojure
(#erl cowboy/stop_listener :starbeam-demo)
```

Open the endpoint from Datastar:

```html
<body data-init="@get('/events')">
  <main id="app"></main>
</body>
```

## Stream lifecycle

Handler sequence:

```clojure
(let [stream (starbeam/open! request)]
  (starbeam/patch-elements! stream html)
  (starbeam/close! stream)
  #erl[:ok (starbeam/request stream) state])
```

`open!` sends status `200` and starts the response with these headers:

```text
Cache-Control: no-cache
Content-Type: text/event-stream
```

HTTP/1.1 responses also receive `Connection: keep-alive`.

`request` returns the updated Cowboy request; return it from the handler. `close!` sends the final body marker. Leave long-lived subscriptions open until their owner process exits.

Only the process that called `open!` may write to the stream. Other processes notify the owner with BEAM messages. Process ownership preserves event order without locks.

### Observe client disconnects

Cowboy calls `terminate/3` when each `/time` loop handler stops. Log the handler PID and direct TCP peer to identify the request:

```clojure
(defn terminate [reason request _state]
  (let [handler-pid (#erl erlang/self)
        peer (#erl cowboy_req/peer request)]
    (#erl io/format
          "Clock client pid=~p peer=~p terminated reason=~p~n"
          (clj_rt/to_list.1 [handler-pid peer reason])))
  nil)
```

Run two clients:

```shell
curl --no-buffer http://127.0.0.1:8080/time
```

Stop one client with `Ctrl-C`. Its PID and peer appear in the REPL; the other stream remains open. Reconnection creates a new handler PID and usually a new source port. Behind a reverse proxy, `cowboy_req/peer` identifies the proxy.

Re-evaluate `terminate` directly. Route dispatch remains unchanged, so `apply-routes!` is unnecessary.

## Patch elements

Send complete HTML elements:

```clojure
(starbeam/patch-elements!
  stream
  "<main id=\"app\"><h1>Current state</h1></main>")
```

Pass patch options as the third argument:

```clojure
(starbeam/patch-elements!
  stream
  "<main id=\"app\">Updated</main>"
  {:selector "#app"
   :mode :inner
   :use-view-transition? true
   :view-transition-selector "#app"
   :namespace :html
   :event-id "view-42"
   :retry-duration 2000})
```

Element options:

| Option | Default | Values |
| --- | --- | --- |
| `:selector` | omitted | CSS selector string |
| `:mode` | `:outer` | `:outer`, `:inner`, `:replace`, `:prepend`, `:append`, `:before`, `:after`, `:remove` |
| `:use-view-transition?` | `false` | boolean |
| `:view-transition-selector` | omitted | CSS selector string |
| `:namespace` | `:html` | `:html`, `:svg`, `:mathml` |
| `:event-id` | omitted | string |
| `:retry-duration` | `1000` | non-negative integer in milliseconds |

Remove an element without sending HTML:

```clojure
(starbeam/patch-elements!
  stream
  nil
  {:selector "#flash" :mode :remove})
```

Starbeam accepts Clojerl strings, Erlang binaries, and valid iodata. It does not parse HTML. Each supplied top-level item must be a complete element.

## Patch signals

Send an RFC 7386 JSON Merge Patch using pre-encoded JSON:

```clojure
(starbeam/patch-signals!
  stream
  "{\"ready\":true,\"count\":4}")
```

OTP's `json` module encodes Erlang JSON terms:

```clojure
(starbeam/patch-signals!
  stream
  #erl{"ready" true
       "count" 4})
```

Set missing signals without replacing existing values:

```clojure
(starbeam/patch-signals!
  stream
  #erl{"theme" "dark"}
  {:only-if-missing? true
   :event-id "signals-9"
   :retry-duration 2000})
```

Pre-encoded JSON passes through unchanged. Decoded and encoded values remain Erlang terms, avoiding another copy of the object graph.

## Execute a script

`execute-script!` appends a `<script>` element to `body`:

```clojure
(starbeam/execute-script!
  stream
  "document.querySelector('#search').focus()")
```

Add attributes or keep the script element after execution:

```clojure
(starbeam/execute-script!
  stream
  "console.log('connected')"
  {:auto-remove? false
   :attributes {"type" "application/javascript"
                "data-source" "starbeam"}
   :event-id "script-3"
   :retry-duration 2000})
```

`:auto-remove?` defaults to `true` and adds `data-effect="el.remove()"`. Starbeam treats script content as trusted developer-authored JavaScript. Do not interpolate untrusted input into it.

## Read Datastar signals

`read-signals` uses the Datastar SDK request contract:

| Method | Signal location |
| --- | --- |
| `GET` | URL-decoded `datastar` query parameter |
| `DELETE` | URL-decoded `datastar` query parameter |
| `POST` | JSON request body |
| `PUT` | JSON request body |
| `PATCH` | JSON request body |

Decode signals and return `204`:

```clojure
(ns demo.command
  (:require [starbeam.core :as starbeam]))

(defn init [request state]
  (let [[signals request] (starbeam/read-signals request)
        _ (when-not (#erl erlang/is_map signals)
            (throw (ex-info "Expected a signal object."
                            {:signals signals})))
        ;; Validate and commit `signals` before replying.
        request (#erl cowboy_req/reply 204 request)]
    #erl[:ok request state]))
```

`read-signals` returns `[signals updated-request]`. Use the returned request after reading a body because Cowboy request values are immutable.

Decoded JSON uses Erlang maps with binary keys, lists, binaries, numbers, booleans, and the JSON null atom. Missing signal data, malformed JSON, unsupported methods, and body-read failures raise exceptions.

## Long-lived CQRS stream

Keep commands and long-lived queries separate at the application layer:

```clojure
(ns demo.subscription
  (:require [starbeam.core :as starbeam]))

(defn- render-main []
  "<main id=\"app\"><h1>Authoritative state</h1></main>")

(defn- stream-loop [stream]
  (starbeam/patch-elements! stream (render-main))
  (receive*
    :refresh (stream-loop stream)
    :stop (starbeam/close! stream)))

(defn init [request state]
  (let [stream (starbeam/open! request)]
    (stream-loop stream)
    #erl[:ok (starbeam/request stream) state]))
```

A command path should:

1. Validate the request.
2. Commit the state change.
3. Notify subscription processes with `:refresh`.
4. Return `204` without an HTML patch.

Each subscription process should:

1. Open its SSE response.
2. Read authoritative state.
3. Render and send a complete element.
4. Wait for a refresh notification.
5. Repeat from authoritative state.

A notification means state may have changed; it carries no HTML delta. Reconnection renders current state and needs no event replay.

## API

```clojure
(starbeam/open! request)
(starbeam/request stream)

(starbeam/patch-elements! stream elements)
(starbeam/patch-elements! stream elements options)

(starbeam/patch-signals! stream signals)
(starbeam/patch-signals! stream signals options)

(starbeam/execute-script! stream script)
(starbeam/execute-script! stream script options)

(starbeam/read-signals request)
(starbeam/close! stream)
```

Event operations return `:ok` or propagate the underlying error. Each operation builds one iodata frame and sends one Cowboy `stream_body` message.

## Scope

Starbeam provides:

- Datastar SDK event framing
- Cowboy SSE response setup
- Element patches
- Signal merge patches
- Script execution events
- Datastar signal decoding

Applications own routing, rendering, persistence, authentication, supervision, pub/sub, compression, and browser assets.

## Development

```shell
rebar3 clojerl compile
rebar3 clojerl test
```

Build the optional Cowboy HTTP/3 profile:

```shell
rebar3 as http3 compile
```

The HTTP/3 profile adds Quicer and Cowboy's QUIC adapter. Starbeam uses the same handler API across HTTP/1.1, HTTP/2, and HTTP/3.

Integration tests start Cowboy and cover exact SSE frames, response headers, signal decoding for every supported method, stream ownership and closure, payload limits, concurrent subscribers, and middleware cookies.

## References

- [Datastar SDK ADR](https://github.com/starfederation/datastar/blob/develop/sdk/ADR.md)
- [Clojerl releases](https://github.com/antlobach/clojerl/releases)
- [rebar3_clojerl](https://github.com/clojerl/rebar3_clojerl)
- [Cowboy 2.13 request API](https://github.com/ninenines/cowboy/blob/2.13.0/src/cowboy_req.erl)
- [Erlang/OTP JSON](https://www.erlang.org/doc/apps/stdlib/json.html)
- [Design notes](DESIGN.md)
