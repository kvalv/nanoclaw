## Setting up from backup

Prerequisites: **Node.js 20+**, **Docker**, **Claude Code CLI** (`npm i -g @anthropic-ai/claude-code`).

```bash
# 1. Extract (from the synced tarball)
cd ~/src
tar xzf ~/scripts/nanoclaw.tar.gz

# 2. Install dependencies and build
cd nanoclaw
npm install
npm run build

# 2.5 -- you probably want to ensure that the $USER is updated if needed. This is a tarball from another laptop.

# 3. Build the agent container image
./container/build.sh

# 4. Install and start the systemd service
mkdir -p ~/.config/systemd/user
cp nanoclaw.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now nanoclaw

# 5. Verify
systemctl --user status nanoclaw
```

The `.env` file with all API tokens (Slack, Telegram, Linear, Google) is included in the tarball. The SQLite database (`store/messages.db`), group configs, and agent session data are also included. No re-authentication should be needed for Slack or Telegram.

---

