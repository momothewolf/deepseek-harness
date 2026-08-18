# Agent Note: Heartbeat reaps zombie browser downlinks so pending questions recover

Status: implemented

English | [中文](2026-08-18-mux-heartbeat.zh.md)

## Problem

A pending `ask_user_question` hangs the session forever when no browser UI
is actually receiving the frame. The host pushes `question/requested` to
every live mux queue at ask time and replays still-pending questions on
each mux-open, but both paths require a connected client. A silently dead
browser WebSocket — machine sleep, a network switch, or a background-tab
throttle — is not detected by either side: the server has no heartbeat,
and the browser only notices a dead socket on the OS TCP timeout, which
can take tens of minutes. During that window the UI still reports
connected, no new mux generation opens, and the replay never runs, so
the question card never renders and the turn stalls indefinitely.

## Decision

`dsh-client-connection` runs a server-side heartbeat on its two
downlink WebSockets (`/api/events.mux`, `/api/events.host`): every
`heartbeatIntervalMs` (new `ConnectionConfig` key, default 30 s) the
server pings each accepted socket and terminates any socket that fails
one pong on the next heartbeat tick (a fresh socket gets two ticks of
grace from connect). The browser auto-pongs at the protocol level (RFC
6455), so the client code is unchanged. `terminate()` fires `close` in
the browser, the existing `ConnectionController` reconnect loop re-opens
both streams under its backoff policy, and the existing mux-open replay
re-delivers still-pending questions and approvals — the question card
appears and the session continues. A fresh socket gets two ticks of
grace from connect before any termination can occur; the grace is an
algorithm constant, while the cadence is deployment-configurable.

## Alternatives considered

**Client-side liveness watchdog.** The browser cannot observe protocol
pongs, so the client would need an application-level ping frame (a new
MuxFrame variant) plus a silence timer. That fixes the same dead-socket
class but adds wire surface the server-side heartbeat avoids; the
existing reconnect loop already turns the server's terminate into a
reconnect with replay.

**Fail `ask_user_question` when no mux connection is open.** This covers
only cleanly closed sockets; a zombie socket still occupies its queue, so
the ask would hang regardless. It could complement a timeout later, but
does not restore the question to a present user.
