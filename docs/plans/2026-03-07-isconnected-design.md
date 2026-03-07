# Design: IsConnected property for TRedisClient

## Problem

`NewRedisClient()` calls `.Connect` automatically, but there is no way to check
whether the connection is still active. Consumers holding an `IRedisClient`
reference cannot determine if the client has been connected or if the connection
is still alive after some time has passed.

## Solution

Add two `IsConnected` overloads to `IRedisClient` and implement them in
`TRedisClient`:

```delphi
function IsConnected: Boolean; overload;
function IsConnected(const aVerifyConnection: Boolean): Boolean; overload;
```

- `IsConnected` returns a simple boolean flag (no network call).
- `IsConnected(True)` additionally sends a `PING` to verify the connection is
  alive. If the PING fails, the flag is set to `False`.

## Changes

### Redis.Commons.pas - IRedisClient interface

Add after `procedure Disconnect;`:

```delphi
function IsConnected: Boolean; overload;
function IsConnected(const aVerifyConnection: Boolean): Boolean; overload;
```

### Redis.Client.pas - TRedisClient class

1. Add private field `FIsConnected: Boolean`.
2. Set `FIsConnected := True` at the end of `Connect`.
3. Set `FIsConnected := False` at the start of `Disconnect`.
4. Implement both `IsConnected` overloads.

The verified variant tries `PING` inside a try/except. On any exception it sets
`FIsConnected := False` and returns `False`.

### Tests

Add a test that creates a client, checks `IsConnected` returns `True`, then
disconnects and checks it returns `False`.

## Rejected alternatives

- **Ping in the constructor**: `NewRedisClient` already raises
  `ERedisConnectionException` on failure, so pinging at construction adds no
  value. The real use case is checking liveness later.
- **Always ping**: Too expensive for the common case where you just need to know
  if `Connect` was called.
