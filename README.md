# trade

Options trading tool built on the IBKR Web API.

- `engined` runs the engine and serves a client API; the web UI, `optctl` and scripts are all clients.
- Every input (adapter traffic, client actions, clock) is written to a session log; replaying it through `engined` reproduces the session.
- The engine and the adapters' `decode`/`encode` modules do no IO.
