# Wake Claude Up

A small systemd service and timer that send a minimal prompt to Claude Code at four scheduled times each day. The example schedule uses `Asia/Tokyo`; change it in the timer files if needed.

Claude Code must be authenticated for the account the timer runs as. Set that up before installing — see [Authentication](#authentication).

## Authentication

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

Both units reference their environment file with a leading `-`, so they still start when it is absent and fall back to whatever login Claude Code has stored. To add the token to an already-installed unit, write the file and run `daemon-reload` — no unit edit is needed.

## User service

Use this option when Claude Code is installed for the current user:

```sh
mkdir -p ~/.config/systemd/user
cp systemd/user/* ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now wake-claude-up.timer
```

Enable lingering as well, so the user manager keeps running after logout and starts at boot without a login. Without it the timer only fires while a session is open:

```sh
loginctl enable-linger "$USER"
```

Verify the service works before relying on the schedule:

```sh
systemctl --user start wake-claude-up.service
journalctl --user -u wake-claude-up.service -n 20
```

## System service

The system template runs as the user named in the unit instance. It assumes Claude Code is installed in `~/.local/bin` and that the user's home is `/home/<user>`, matching the tested configuration; adjust `Environment=PATH=` in the unit if that is not the case.

```sh
sudo cp systemd/system/* /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now "wake-claude-up@$USER.timer"
```

Verify the service works before relying on the schedule:

```sh
sudo systemctl start "wake-claude-up@$USER.service"
journalctl -u "wake-claude-up@$USER.service" -n 20
```

## Proxy

If Claude Code requires a proxy, create a service override with `systemctl edit` or `systemctl --user edit` and add:

```ini
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:8080"
Environment="HTTPS_PROXY=http://127.0.0.1:8080"
```

## Notes

An interactive login is not enough for a timer. Its refresh token carries a fixed expiry that does not slide with use, so the session eventually dies while the timer keeps firing, and every run from then on fails with `Failed to authenticate: OAuth session expired and could not be refreshed`. Claude Code also clears the stored credentials on that first failure, so there is nothing left to recover. A token from `claude setup-token` is not subject to this.

Nothing surfaces that failure on its own — the timer stays active and the service just exits non-zero, so a broken schedule can go unnoticed for days. Check with `systemctl is-failed "wake-claude-up@$USER.service"` (or `systemctl --user is-failed wake-claude-up.service`), or attach an `OnFailure=` unit that notifies you.

Missed runs are not caught up (`Persistent=false`): if the machine is off or asleep at a scheduled time, that run is skipped rather than fired late at the wrong point in the day.

Each scheduled run sends a real Claude request and may count toward usage limits or incur charges.
