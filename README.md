# Wake Claude Up

Small systemd services and timers that send a minimal prompt to a coding agent at four scheduled times each day, so its five-hour usage windows start when you want them to rather than whenever you happen to type the first request. Units ship for Claude Code (`wake-claude-up`) and for Codex (`wake-codex-up`); the two are independent, so install either one or both. The example schedule uses `Asia/Tokyo`; change it in the timer files if needed.

The shipped schedule starts five-hour usage windows at 07:30, 12:31, 17:32, and 22:33. The one-minute offsets leave room for the previous window to close before the next request reaches the agent; the remaining quiet period is 03:33–07:30. Both agents use the same times, and since the two services share nothing, running them together is fine.

The agent must be authenticated for the account the timer runs as. Set that up before installing — see [Authentication](#authentication).

## Authentication

### Claude Code

A timer needs credentials that survive unattended, which an interactive login does not provide (see [Notes](#notes)). Generate a long-lived token instead:

```sh
claude setup-token
```

Hand it to the unit through an environment file. The `read` lines below are bash syntax; run them under bash if your shell is something else.

System service — systemd reads the file as root before dropping to the instance user, so it stays out of the user's home:

```sh
sudo mkdir -p /etc/wake-claude-up
umask 077; read -rsp 'Token: ' T; echo
printf 'CLAUDE_CODE_OAUTH_TOKEN=%s\n' "$T" | sudo tee "/etc/wake-claude-up/$USER.env" >/dev/null
unset T
sudo chmod 600 "/etc/wake-claude-up/$USER.env"
sudo chown root:root "/etc/wake-claude-up/$USER.env"
```

User service:

```sh
mkdir -p ~/.config/wake-claude-up
umask 077; read -rsp 'Token: ' T; echo
printf 'CLAUDE_CODE_OAUTH_TOKEN=%s\n' "$T" > ~/.config/wake-claude-up/env
unset T
```

Both units reference their environment file with a leading `-`, so they still start when it is absent and fall back to whatever login Claude Code has stored. To add or replace the token for an already-installed unit, just write the file; systemd reads it again each time the service starts, so no unit edit or `daemon-reload` is needed.

### Codex

Codex has no equivalent of `claude setup-token`. Log in once as the account the timer runs as:

```sh
codex login
```

The credentials land in `$CODEX_HOME/auth.json` (`~/.codex` by default), and nothing else needs installing beside the unit. The access token there is long-lived — ten days on the login this was tested against — and the file carries a refresh token beside it, so the session is not expected to die the way an interactive Claude Code login does. That renewal has not been observed happening under a timer, though, so treat a run that starts failing as a credentials problem first and re-run `codex login`.

The Codex units keep all wake settings in `ExecStart`; no profile, configuration file, or custom instruction file is needed. `--ignore-user-config` skips the user's Codex configuration while authentication still comes from `CODEX_HOME`. If credentials live elsewhere, set `Environment=CODEX_HOME=/path/to/codex-home` in the service. Use the existing ChatGPT login for a plan-window wake.

The rest of the command line is the minimum a wake needs. `--skip-git-repo-check` lets the run start from a home directory that is not a repository, and `--ephemeral` leaves no session file behind. `--sandbox read-only` and `-c approval_policy=never` are already the defaults with user configuration ignored; they are written out anyway, because a default that widens in some later release would quietly give an unattended root-installed unit more than it needs, and pinning them costs nothing. Nothing else is disabled: every switch that trims context is tied to a capability name that can be renamed or removed between versions, and `-c` overrides are not validated, so a stale key fails silently. No model is pinned either, unlike the Claude unit, because the account-default model avoids selecting one the ChatGPT account does not support. Add `--model` to pin a supported model if needed.

## User service

Use this option when the agent is installed for the current user. Copy the units for whichever agents you want, and leave out the lines for the other:

```sh
mkdir -p ~/.config/systemd/user
cp systemd/user/wake-claude-up.* ~/.config/systemd/user/
cp systemd/user/wake-codex-up.* ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now wake-claude-up.timer wake-codex-up.timer
```

Enable lingering as well, so the user manager keeps running after logout and starts at boot without a login. Without it the timer only fires while a session is open:

```sh
loginctl enable-linger "$USER"
```

Verify each installed service before relying on the schedule:

```sh
systemctl --user start wake-claude-up.service
journalctl --user -u wake-claude-up.service -n 20
systemctl --user start wake-codex-up.service
journalctl --user -u wake-codex-up.service -n 20
```

## System service

The system templates run as the user named in the unit instance. They assume the agent is installed in `~/.local/bin` and that the user's home is `/home/<user>`, matching the tested configuration; adjust `Environment=PATH=` in the unit if that is not the case.

```sh
sudo cp systemd/system/wake-claude-up@.* systemd/system/wake-codex-up@.* /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now "wake-claude-up@$USER.timer" "wake-codex-up@$USER.timer"
```

Verify each installed service before relying on the schedule:

```sh
sudo systemctl start "wake-claude-up@$USER.service"
journalctl -u "wake-claude-up@$USER.service" -n 20
sudo systemctl start "wake-codex-up@$USER.service"
journalctl -u "wake-codex-up@$USER.service" -n 20
```

## Proxy

If the agent requires a proxy, create a service override with `systemctl edit` or `systemctl --user edit` — on the service, not the timer — and add:

```ini
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:8080"
Environment="HTTPS_PROXY=http://127.0.0.1:8080"
```

## Notes

An interactive login is not enough for a Claude Code timer. Its refresh token carries a fixed expiry that does not slide with use, so the session eventually dies while the timer keeps firing, and every run from then on fails with `Failed to authenticate: OAuth session expired and could not be refreshed`. Claude Code also clears the stored credentials on that first failure, so there is nothing left to recover. A token from `claude setup-token` is not subject to this.

Nothing surfaces a failure on its own — the timer stays active and the service just exits non-zero, so a broken schedule can go unnoticed for days. Check with `systemctl is-failed "wake-claude-up@$USER.service"` (or `systemctl --user is-failed wake-codex-up.service`), or attach an `OnFailure=` unit that notifies you.

Missed runs are not caught up (`Persistent=false`): if the machine is off or asleep at a scheduled time, that run is skipped rather than fired late at the wrong point in the day.

Each scheduled run sends a real request and may count toward usage limits or incur charges. The visible prompt and reply are short, but the agent's startup context can make the actual input much larger, so do not treat their length as a measure of usage or cost.

With user configuration ignored, `codex exec` currently defaults to approval policy `never`, a read-only sandbox, and reasoning effort `none`. Recheck them after a CLI upgrade. A Codex run's journal entries open with `Reading additional input from stdin...`, which is `codex exec` finding the empty stdin systemd hands it, not a problem.
