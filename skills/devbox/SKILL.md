---
name: devbox
description: Use on a shared development machine running CARLOS devbox mode, whenever you'd hand someone a link to work running on this machine. That means a dev server to review, a mockup folder, or a prototype ("give me a link", "share this", "review link", "can I see it on my phone"). Publishes an owned, tailnet-only HTTPS link with `carlos devbox share`, and removes it when the review is done. Not for public sharing or production deploys.
---

# 🤖 Devbox links

A devbox is a shared Linux workstation on a private network (a tailnet) where the platform edge runs as a service. Anyone with an account on it can publish `https://<name>.<devbox domain>` with no port. The link is reachable only from the tailnet, and it points at either:

- **a live link**: your own dev server on `127.0.0.1:<port>`. Edits show up live, because live reload passes through.
- **a folder link**: a copy of a static folder (mockups, decks, prototypes), served with its `_headers` file.

The name belongs to the account that published it. Nobody else can repoint or remove it.

## Is this machine a devbox?

```sh
carlos devbox list
```

- **It prints a table, or "no links yet":** it's a devbox. Carry on.
- **"could not reach the devbox service":** run it again outside the sandbox (next section). If it still fails, this machine isn't a devbox or the service is down: tell the user. Don't fall back to `tailscale serve`, `sudo`, or binding `0.0.0.0`.

## Run outside the sandbox

Agent command sandboxes usually have their own private network and block Unix sockets. Two consequences:

1. A dev server started **inside** the sandbox listens where the devbox can't reach it. The link will exist, but it never answers.
2. `carlos devbox share` can't reach the devbox service from inside the sandbox.

So run everything that touches the dev server or the devbox outside the sandbox: starting and stopping the dev server, `carlos devbox share` / `unshare` / `list`, and the `curl` check. In Claude Code, that means `dangerouslyDisableSandbox: true` on those calls, which goes through the permission gate as usual. Editing files and running tests stay sandboxed.

## Share a dev server

1. Start the dev server on loopback only: `127.0.0.1:<port>`, never `0.0.0.0`. Run it as a background command (your harness's background option, or `&` and `$!`), and note its PID so you can stop it later.
2. Publish it:

   ```sh
   carlos devbox share --port <port>          # name = your git branch, folded: lower case, runs of
                                              # other characters become one '-', trimmed, max 63
   carlos devbox share --port <port> shop     # or name it
   ```

   It prints exactly one line: the URL. Its suffix after the first label is the devbox domain, which the dev server settings below need.
3. Check that it answers before handing it over: `curl -sI <url>` should return your app's status. `carlos devbox list` shows `answering` or `not answering`.
4. Give the user the **full URL** and a short list of what to look at.

## Share a folder

```sh
carlos devbox share --dir ./mockup pricing-mockup
```

Name folder links explicitly: the default is the git branch name, which your live link may already be using.

The folder is uploaded as a snapshot. Sharing it again with the same name replaces the copy, so re-run it after edits. Plain files and folders only: symlinks and special files are refused. The cap is 200 MiB per folder, and the devbox has a shared total.

## When the review is done

```sh
carlos devbox unshare <name>
```

`unshare` only removes the link. Stop the dev server you started yourself, then run `carlos devbox list` to check nothing of yours is left. Links don't expire, so the session that opened one closes it.

## Dev server settings, once per project

The edge passes the link's own hostname to your dev server, and live reload has to connect on port 443:

| Tool | Setting |
|---|---|
| Vite | `server.allowedHosts` includes the devbox domain (e.g. `.devbox.example.com`); `server.hmr.clientPort: 443` |
| Rails | `config.hosts << ".devbox.example.com"` in development |
| Others | allow the devbox domain as a host, and point the live-reload client at port 443 |

Without the host setting, the dev server answers its own "blocked host" page through the link.

## Refusals

| Message starts with | Do |
|---|---|
| `<name> is taken by <owner>` | Pick another name. It belongs to someone else. |
| `<name> is already in use on this devbox` | Pick another name. |
| `not your link` | You can't remove another account's link. Leave it. |
| `the branch ... has no letters or digits` / `no name given` | Pass a name explicitly. |
| `could not reach the devbox service` | Run the command outside the sandbox. If it still fails, tell the user, since the service may be down. |
| `... no room for another folder` | Unshare a folder you no longer need, or tell the user. |

## Don't

- Use `tailscale serve`, `sudo`, or a `0.0.0.0` bind to work around a refusal.
- Hand over a link you haven't checked.
- Use this for public sharing. Devbox links are tailnet-only. Deploy to the hosted platform for anything wider.
- Leave links or dev servers running after the review.
