# JIP-1: Debug message host call

A host call for passing a debugging message from the service/authorizer to the hosting environment for logging to the node operator.

## Host-call specification

**Index**: 100

**Name**: `log`

**Gas usage**: 10 (same as host-call with bad index)

**Input registers**: $\varphi_{7\dots+5}$

- `level` = $\varphi_7$
- `target` = $\begin{cases}\varnothing &\text{when } \varphi_8 = 0 \wedge \varphi_9 = 0 \\ \mu_{\varphi_8\dots+\varphi_9} &\text{otherwise}\end{cases}$
- `message` = $\mu_{\varphi_{10}\dots+\varphi_{11}}$

**Output registers**: $\varphi'_7$

- $\varphi'_7$ = `WHAT`. `WHAT` is always returned so that authorizer/service behaviour is the same
  whether or not this JIP is implemented.

### Side-effects

No side-effects if memory access is invalid.

Otherwise, express a message to user according to the user-agent.



## Suggestions & examples

### Levels definition

- 0: User agent displays as fatal error ⛔️
- 1: User agent displays as warning ⚠️
- 2: User agent displays as important information ℹ️
- 3: User agent displays as helpful information 💁
- 4: User agent displays as pedantic information 🪡

### Display format for console logging

Note that `<CORE>` is assumed to be the integer index of the core on which the PVM is executing, which may not exist (e.g. in the Accumulate logic).

Note that `<SERVICE_ID>` is assumed to be the integer index of the service for which the PVM is executing, which may not exist (e.g. in the Is-Authorized logic).


```
<YYYY-MM-DD hh-mm-ss> <LEVEL>[@<CORE>]?[#<SERVICE_ID>]? [<TARGET>]? <MESSAGE>
```

#### Example log item

```
2025/01/01 12:10:42 DEBUG@1#42 bootstrap-refine Hello world!
```

### Format for JSON logging

```
{
    "time": "<YYYY-MM-DD hh-mm-ss>",
    "level": "<LEVEL>",
    "message": "<MESSAGE>",
    "target": "<TARGET>" | null
    "service": "<SERVICE_ID>" | null
    "core": "<CORE>" | null
}
```
