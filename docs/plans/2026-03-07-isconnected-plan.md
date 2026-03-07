# IsConnected Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add `IsConnected` to `IRedisClient` so consumers can check connection state, with an optional PING-based liveness verification.

**Architecture:** A boolean field `FIsConnected` tracks state, set in `Connect`/`Disconnect`. Two overloads: parameterless returns the flag; with `aVerifyConnection = True` sends a PING and updates the flag on failure.

**Tech Stack:** Delphi, DUnit (existing test framework in `tests/TestRedisClientU.pas`)

---

### Task 1: Create feature branch

**Step 1: Create and switch to feature branch**

Run: `git checkout -b feature/isconnected`

**Step 2: Verify branch**

Run: `git branch --show-current`
Expected: `feature/isconnected`

---

### Task 2: Add IsConnected to IRedisClient interface

**Files:**
- Modify: `sources/Redis.Commons.pas:338` (after `procedure Disconnect;`)

**Step 1: Add overloads to IRedisClient**

In `sources/Redis.Commons.pas`, after line 338 (`procedure Disconnect;`), add:

```delphi
    function IsConnected: Boolean; overload;
    function IsConnected(const aVerifyConnection: Boolean): Boolean; overload;
```

The result at lines 337-341 should look like:

```delphi
    procedure Connect;
    procedure Disconnect;
    function IsConnected: Boolean; overload;
    function IsConnected(const aVerifyConnection: Boolean): Boolean; overload;
    function InTransaction: boolean;
```

**Step 2: Commit**

```bash
git add sources/Redis.Commons.pas
git commit -m "Add IsConnected overloads to IRedisClient interface"
```

---

### Task 3: Add FIsConnected field and class declarations to TRedisClient

**Files:**
- Modify: `sources/Redis.Client.pas:46` (private section, after `FIsTimeout`)
- Modify: `sources/Redis.Client.pas:278` (public section, after `Disconnect`)

**Step 1: Add private field**

In `sources/Redis.Client.pas`, after line 46 (`FIsTimeout: boolean;`), add:

```delphi
    FIsConnected: boolean;
```

**Step 2: Add public method declarations**

In `sources/Redis.Client.pas`, after line 278 (`procedure Disconnect;`), add:

```delphi
    function IsConnected: Boolean; overload;
    function IsConnected(const aVerifyConnection: Boolean): Boolean; overload;
```

**Step 3: Commit**

```bash
git add sources/Redis.Client.pas
git commit -m "Add FIsConnected field and IsConnected declarations to TRedisClient"
```

---

### Task 4: Implement IsConnected and update Connect/Disconnect

**Files:**
- Modify: `sources/Redis.Client.pas` (Connect, Disconnect, new implementations)

**Step 1: Update Connect to set FIsConnected**

In `sources/Redis.Client.pas`, the `Connect` procedure (line 462-465) should become:

```delphi
procedure TRedisClient.Connect;
begin
  FTCPLibInstance.Connect(FHostName, FPort);
  FIsConnected := True;
end;
```

**Step 2: Update Disconnect to clear FIsConnected**

In `sources/Redis.Client.pas`, the `Disconnect` procedure (line 513-519) should become:

```delphi
procedure TRedisClient.Disconnect;
begin
  try
    FIsConnected := False;
    FTCPLibInstance.Disconnect;
  except
  end;
end;
```

**Step 3: Add IsConnected implementations**

Add the two implementations near the `Connect`/`Disconnect` methods (after `Disconnect`):

```delphi
function TRedisClient.IsConnected: Boolean;
begin
  Result := FIsConnected;
end;

function TRedisClient.IsConnected(const aVerifyConnection: Boolean): Boolean;
begin
  if aVerifyConnection and FIsConnected then
  begin
    try
      PING;
    except
      FIsConnected := False;
    end;
  end;
  Result := FIsConnected;
end;
```

**Step 4: Commit**

```bash
git add sources/Redis.Client.pas
git commit -m "Implement IsConnected with optional PING verification"
```

---

### Task 5: Add tests

**Files:**
- Modify: `tests/TestRedisClientU.pas`

**Step 1: Add test method declarations**

In the `published` section of `TestRedisClient` class, add:

```delphi
    procedure TestIsConnected;
    procedure TestIsConnectedAfterDisconnect;
```

**Step 2: Implement tests**

Add in the implementation section:

```delphi
procedure TestRedisClient.TestIsConnected;
begin
  CheckTrue(FRedis.IsConnected, 'IsConnected should be True after Connect');
  CheckTrue(FRedis.IsConnected(False), 'IsConnected(False) should be True after Connect');
  CheckTrue(FRedis.IsConnected(True), 'IsConnected(True) with PING should be True for active connection');
end;

procedure TestRedisClient.TestIsConnectedAfterDisconnect;
begin
  CheckTrue(FRedis.IsConnected, 'IsConnected should be True before Disconnect');
  FRedis.Disconnect;
  CheckFalse(FRedis.IsConnected, 'IsConnected should be False after Disconnect');
  CheckFalse(FRedis.IsConnected(True), 'IsConnected(True) should be False after Disconnect');
end;
```

**Step 3: Commit**

```bash
git add tests/TestRedisClientU.pas
git commit -m "Add tests for IsConnected"
```

---

### Task 6: Compile and verify

**Step 1: Compile the test project using delphi-build MCP**

Compile `tests/RedisClientTests.dpr` for Win64.

**Step 2: Fix any compilation issues if needed**

**Step 3: Commit any fixes**

```bash
git add -A
git commit -m "Fix compilation issues"
```
