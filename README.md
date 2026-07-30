# Wake Claude Up

A small systemd service and timer that send a minimal prompt to Claude Code at four scheduled times each day. The example schedule uses `Asia/Tokyo`; change it in the timer files if needed.

Claude Code must already be authenticated for the account the timer runs as. Run `claude -p "Reply with only OK."` once by hand before installing.

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

Missed runs are not caught up (`Persistent=false`): if the machine is off or asleep at a scheduled time, that run is skipped rather than fired late at the wrong point in the day.

Each scheduled run sends a real Claude request and may count toward usage limits or incur charges.
