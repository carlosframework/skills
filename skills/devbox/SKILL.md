---
name: devbox
description: Use on a shared development machine running CARLOS devbox mode, whenever you'd hand someone a link to work running on this machine. That means a dev server to review, a mockup folder, or a prototype ("give me a link", "share this", "review link", "can I see it on my phone"). Publishes an owned, tailnet-only HTTPS link with `carlos devbox share`, and removes it when the review is done. Not for public sharing or production deploys.
---

# 🤖 Devbox links

A devbox is a shared Linux workstation on a private network (a tailnet) where the platform edge runs as a service. Anyone with an account on it can publish `https://<name>.<devbox domain>` with no port. The link is reachable only from the tailnet, and it points at either:

- **a live link**: your own dev server on `127.0.0.1:<port>`. Edits show up live, because live reload passes through.
- **a folder link**: a copy of a static folder (mockups, decks, prototypes), served with its `_headers` file.
- **a socket link** (Linux devboxes only): a tool of yours listening on a Unix socket instead of a port, so no other account on the machine can reach it except through the link.

On a Linux devbox, any link can be limited to named tailnet logins with `--allow`, and requests through a link carry the visitor's identity in headers your tool can trust (absent when the visitor is anonymous).

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

## Share a socket (Linux only)

Use this for a tool that should only be reachable through the link, such as a review daemon that needs to know who is looking.

1. Have the tool create its socket directly inside `/run/carlos-devbox/sockets` (any account can create one there; nobody can list the folder, and only the owner can remove a socket). Run the tool outside the sandbox, like a dev server.
2. Publish it:

   ```sh
   carlos devbox share --socket /run/carlos-devbox/sockets/review.sock review
   ```

   `share` makes the socket private to you and the devbox service, and checks that something answers on it.
3. After any restart of the tool, run `share` again: a restarted tool almost always removes and recreates its socket file, and the new file is not private until `share` runs.

On a Mac devbox `--socket` is refused: use `--port`.

## Limit who can open a link: `--allow`

```sh
carlos devbox share --port 5173 --allow alice@github,bob@github shop
```

- Logins are tailnet logins, separated by commas, no spaces. Works for every kind of link.
- Anyone else sees a page saying the link is private. Without `--allow`, everyone on the tailnet can open it.
- Sharing again replaces the list; sharing without `--allow` opens the link to the tailnet again.
- A browser on the devbox itself (or on another shared machine the operator listed) is anonymous, so an allow-listed link shows it the private page. Test from your own device.
- `carlos devbox list` shows an access column: `private` or `tailnet`. `carlos devbox list --allowed` also shows who can open your own private links (other people's lists stay hidden).
- On a Mac devbox `--allow` is refused: every link there is open to the tailnet.

## Who is visiting: identity headers

On a Linux devbox the edge adds these headers to every request through any link:

| Header | Value |
|---|---|
| `Carlos-Visitor-Login` | the visitor's tailnet login, e.g. `alice@github` |
| `Carlos-Visitor-Name` | their display name |
| `Carlos-Visitor-Node` | the device they are on |

The edge removes any of these a visitor sends (in any spelling, including underscores), so a tool behind a link can trust them. They are absent when the visitor is anonymous: a request from the devbox itself, from a shared machine, or when the devbox can't ask Tailscale. Treat a missing login as anonymous, never as the owner.

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
| `... is not directly inside <sockets dir>` | Have the tool create its socket in that folder, then share again. |
| `... is not a socket` / `... belongs to another account` | Give the path of a socket your own tool created. |
| `nothing is answering on ...` | Start the tool, then share again. |
| `could not let the devbox service open ...` / `this machine has no devbox service user ...` | Tell the user; the devbox setup needs attention. |
| `--socket works on Linux devboxes only` / `--allow works on Linux devboxes only` | On a Mac devbox, use `--port` and an open link. |
| `... is not a tailnet login` | Give logins such as `alice@github`, separated by commas, no spaces. |
| `flag provided but not defined: -socket` (or `-allow`) | The installed carlos CLI predates these flags. Tell the user. |

## Don't

- Use `tailscale serve`, `sudo`, or a `0.0.0.0` bind to work around a refusal.
- Hand over a link you haven't checked.
- Use this for public sharing. Devbox links are tailnet-only. Deploy to the hosted platform for anything wider.
- Leave links or dev servers running after the review.
