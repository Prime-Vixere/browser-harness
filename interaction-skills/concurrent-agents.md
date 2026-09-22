# Concurrent agents

Two agents on the same Chrome with the default daemon (no `BU_NAME`) share **one daemon and one current session**. A `switch_tab()` or `new_tab()` in one silently retargets the other's `js()`, `goto()`, `screenshot()` and `page_info()`.

Symptoms:
- `js()` starts returning another site's content mid-loop
- a tab you opened gets closed
- `RuntimeError: WebSocket connection closed`
- `restart_daemon()` to recover also breaks the other agent

## 1. One daemon per agent

Prefix every call with a distinct name. Socket, pid and log are namespaced: `/tmp/bu-<name>.sock`, `.pid`, `.log`.

```bash
BU_NAME=agent-a browser-harness <<'PY'
print(page_info())
PY
```

When done, stop only your own daemon:

```bash
kill "$(cat /tmp/bu-agent-a.pid)"; rm -f /tmp/bu-agent-a.sock /tmp/bu-agent-a.pid
```

## 2. Own your target; pin every CDP call to it

Do not rely on the daemon's current session. Create a target, attach, and pass `session_id` explicitly:

```python
tid = cdp("Target.createTarget", url="https://example.com", newWindow=True, width=1440, height=900)["targetId"]
sid = cdp("Target.attachToTarget", targetId=tid, flatten=True)["sessionId"]
open("/tmp/agent-a.tid", "w").write(tid)          # later heredocs re-attach from this

ev = lambda expr: cdp("Runtime.evaluate", session_id=sid, expression=expr,
                      returnByValue=True, awaitPromise=True).get("result", {}).get("value")
shot = cdp("Page.captureScreenshot", session_id=sid, format="jpeg", quality=80,
           captureBeyondViewport=True, clip={"x": 0, "y": 0, "width": 1440, "height": 900, "scale": 1})
```

In a later invocation:

```python
tid = open("/tmp/agent-a.tid").read().strip()
sid = cdp("Target.attachToTarget", targetId=tid, flatten=True)["sessionId"]
```

## 3. A new window, not a focused tab

Use `newWindow=True` instead of `switch_tab()` / `Target.activateTarget`. You never steal focus from the user's or another agent's tab, and screenshots still work because the window is visible.

## 4. Assert the host before every step

A hijacked session should fail loudly, not act on someone else's page:

```python
href = ev("location.href")
assert href.startswith("https://example.com/"), f"wrong page: {href}"
```

## 5. Gotcha: loading a helper file

Inside the harness heredoc, `exec(open("lib.py").read())` defines functions that cannot see each other or top-level names. Pass the globals:

```python
exec(open("lib.py").read(), globals())
```

## 6. Clean up

Close only the targets you created:

```python
cdp("Target.closeTarget", targetId=tid)
```
