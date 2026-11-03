# 🤖 Private workstation links

Read only when sharing a local server or static folder from a configured
CARLOS devbox. This is a workstation feature, not a Carloku deployment or
public hosting recipe. If the installed devbox skill is available, use it
for the complete workstation workflow.

```sh
carlos devbox list
carlos devbox share --port 5173 preview
carlos devbox share --dir ./public mockup
carlos devbox share --port 5173 --allow alice@github preview
carlos devbox share --socket /run/carlos-devbox/sockets/review.sock review
carlos devbox unshare preview
```

On a Linux devbox, `--allow` limits a link to named tailnet logins, and
`--socket` serves a tool's own Unix socket from `/run/carlos-devbox/sockets`.
Every request carries `Carlos-Visitor-Login`, `Carlos-Visitor-Name` and
`Carlos-Visitor-Node`, which the edge strips from visitors and sets itself;
a missing login means anonymous. Neither flag works on a Mac devbox.

Run the development server on `127.0.0.1:5173`, never a public or LAN bind.
The returned HTTPS link is tailnet-only and owned by the account that created
it. Share that exact returned link. Do not guess the devbox domain, use a
public tunnel, or change production DNS. Folder links are copied snapshots;
re-share after edits. Remove links and stop task-owned servers when done,
unless the user explicitly asked to keep them running.

The dev server and share command need the host network namespace, outside
an isolated agent sandbox, using the environment's permitted execution path.
If sandbox policy blocks that path, request access instead of bypassing it.

If the devbox service is unavailable, check the local setup; an installed CLI
alone does not make the machine a devbox. Respect the sandbox and machine
network policy.

Source: platform `devbox_cli.go`, `internal/edge/devboxfront.go`,
`docs/site/cli.md` (#486 A);
CARLOS plugin `skills/devbox/SKILL.md`.
