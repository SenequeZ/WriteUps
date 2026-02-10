# glotq - LACTF 2026

> jq / yq / xq as a service!

## tl;dr

Go's `encoding/json` matches struct fields case-insensitively. Go's `yaml.v3` does not. The middleware validates your request with one parser, the handler executes it with the other. Slip in an uppercase `"Command"` that YAML ignores but JSON happily accepts, and suddenly you're running `man -H/readflag jq` instead of `jq .`.

## The Setup

The app is a Go web server with three endpoints (`/json`, `/yaml`, `/xml`) that let you run `jq`, `yq`, or `xq` on your input. There's also `man` for reading the docs, because the author is a gentleman.

A `SecurityMiddleware` sits in front of every handler. It reads your request body, parses it based on your `Content-Type` header, checks that the command is allowed for that endpoint, and if you're running `man`, makes sure you're only looking up `jq`/`yq`/`xq`. Sensible stuff.

The flag lives at `/flag.txt` (mode 400, owned by root). A SUID binary `/readflag` is the only way to read it.

## The Bug

Here's the thing nobody thinks about until it bites them: **the middleware and the handler parse the body independently, and they don't necessarily use the same parser.**

The middleware picks a parser based on `Content-Type`. The handler picks a parser based on which endpoint you hit. So if you POST to `/json` with `Content-Type: application/yaml`...

-**Middleware** → YAML parser

-**Handler** (JSONHandler) → JSON parser

Now here's the spicy part. These two parsers disagree on something fundamental:

```go

typeJSONPayloadstruct {

Commandstring`json:"command"`

Args    []string`json:"args"`

}

```

Go's `json.Unmarshal` does **case-insensitive** matching — `"Command"`, `"COMMAND"`, `"cOmMaNd"` all match the `command` tag. It's documented behavior, look it up. Meanwhile `yaml.v3` is strict: `"Command"` ≠ `"command"`, no match, see ya later.

So I send this perfectly valid JSON (which is also valid YAML, since JSON is a subset of YAML):

```json

{

"command": "jq",

"args": ["."],

"Command": "man",

"Args": ["-H/readflag", "jq"]

}

```

The YAML parser in the middleware sees `command: "jq"` and `args: ["."]`. The uppercase keys? Doesn't know 'em. Not my problem. `jq` is allowed on `/json`. We pass the security check.

The JSON parser in the handler sees *both* — and since `"Command"` case-insensitively matches `command`, it **overwrites** the lowercase value. Last key wins. Now we're executing `man` with args `["-H/readflag", "jq"]`.

## Getting the Flag

The `executeCommand` function only checks a global allowlist (`jq`, `yq`, `xq`, `man`) — not the per-endpoint restrictions. So `man` sails right through.

`man -H<browser> <topic>` formats a man page as HTML and opens it with the specified "browser" program. Unlike the pager (`-P`), this works regardless of whether stdout is a TTY because it writes to a temp file and `exec`s the browser.

So `man -H/readflag jq` formats the jq man page, then runs `/readflag` as the "browser". `/readflag` is SUID root, doesn't care about its arguments, just reads `/flag.txt` and prints it to stdout. Flag captured by `CombinedOutput()`. GG.

## Solve Script

```python

import requests


url = "https://glotq-wafyy.instancer.lac.tf"

resp = requests.post(f"{url}/json",

headers={"Content-Type": "application/yaml"},

json={

"command": "jq", "args": ["."],

"Command": "man", "Args": ["-H/readflag", "jq"]

    })

print(resp.json()["output"])

```
