---
title: "Using Claude Code CLI Over SSH: Fixing the macOS Keychain Problem"
description: "If you're running claude -p on a headless Mac, Keychain ACLs silently block your credentials. Here's the fix."
pubDate: 2026-03-16
tags: ["Claude Code", "macOS", "SSH", "DevOps", "AI"]
---

## If you're running `claude -p` on a headless Mac, Keychain ACLs silently block your credentials. Here's the fix.

If you're running Claude Code (`claude -p`) on a headless Mac — via SSH, pm2, cron, or any non-interactive session — you've probably hit this:

```
Not logged in · Please run /login
```

Even though you just ran `claude login` and it said "Login successful." Here's why, and how to fix it.

---

### The problem

Claude Code stores OAuth tokens in the macOS [Keychain](https://developer.apple.com/documentation/security/keychain-services). Keychain entries have [access control lists](https://developer.apple.com/documentation/security/access-control-lists) (ACLs) that, by default, require user interaction — a GUI prompt — to read the password field. In a terminal session (with a desktop), this works transparently. Over SSH, there's no GUI context, so Keychain returns [`errSecInteractionNotAllowed`](https://developer.apple.com/documentation/security/errsecinteractionnotallowed) (exit code 36), and Claude Code sees no credentials.

You can verify this yourself:

```bash
# From SSH — fails with exit code 36
ssh myserver "security find-generic-password -a \$USER -s 'Claude Code-credentials' -w"

# From a local terminal on the same machine — works fine
security find-generic-password -a $USER -s "Claude Code-credentials" -w
```

`security unlock-keychain` doesn't help because the restriction is per-item ACL, not the keychain lock state.

---

### The fix

[Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) has a built-in file-based credential fallback. If `~/.claude/.credentials.json` exists, it reads credentials from there instead of Keychain. The file uses the exact same JSON format.

After logging in from a GUI terminal, sync the token to the file:

```bash
security find-generic-password -a "$USER" -s "Claude Code-credentials" -w > ~/.claude/.credentials.json
chmod 600 ~/.claude/.credentials.json
```

That's it. `claude -p` now works over SSH.

---

### Automating it

Add these to your `~/.zshrc` (or `~/.bashrc`):

```bash
# Sync Claude Code credentials from Keychain to file (for SSH/pm2 access)
claude-sync-creds() {
  security find-generic-password -a "$USER" -s "Claude Code-credentials" -w > ~/.claude/.credentials.json 2>/dev/null
  if [ -s ~/.claude/.credentials.json ]; then
    chmod 600 ~/.claude/.credentials.json
    echo "Credentials synced to ~/.claude/.credentials.json"
  else
    rm -f ~/.claude/.credentials.json
    echo "ERROR: Keychain entry is empty. Try logging in again."
  fi
}

# Login + auto-sync in one step
claude-login() {
  claude login && claude-sync-creds
}
```

Now `claude-login` handles everything — OAuth flow plus file sync. After a token refresh or re-login, just run it again.

---

### Bonus: Node.js vs native binary

If you have two Claude Code installs (npm/homebrew and the native standalone), be aware they behave differently with Keychain. In our case, the Node.js version (`/opt/homebrew/bin/claude`) wrote empty Keychain entries — the entry existed but the data blob was NULL. The native binary (`~/.local/bin/claude`) wrote correctly. If `claude-sync-creds` reports the Keychain entry is empty, check which binary you're logging in with.

---

### Why this matters

If you're using Claude Code for automation — CI/CD pipelines, background agents, cron jobs, or process managers like pm2 on macOS — this is a prerequisite. The Keychain-only approach simply doesn't work outside a GUI session.

The `.credentials.json` fallback is a first-class feature of [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview), not a hack. It's the same mechanism used on Linux, where there's no Keychain equivalent. On macOS, it's just not the default path.

---

*Read more on [oldeucryptoboi.com](https://oldeucryptoboi.com/blog/claude-code-ssh-keychain-fix/)*
