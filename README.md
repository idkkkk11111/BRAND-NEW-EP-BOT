# GAR EP / OP Bot

A Discord bot for tracking **Event Points (EP)** and **Officer Points (OP)** with
quota enforcement and an automatic strike ladder.

- **Officers** give members **EP**.
- **HICOM** give Officers **OP**.
- **Upper HICOM / Officer Overseer** run quota checks that apply strikes (and, for EP, kicks).

Storage is a plain JSON file — no database to install and nothing to compile, so it
deploys cleanly on Railway, Render, a VPS, or your own machine.

---

## 1. Create the bot application

1. Go to https://discord.com/developers/applications → **New Application**.
2. Open the **Bot** tab → **Add Bot**. Copy the **token** (you'll need it).
3. Still on the **Bot** tab, scroll to **Privileged Gateway Intents** and enable:
   - **Server Members Intent** (needed for member fetch / quota checks)
   - **Message Content Intent** (needed for the `-logep` prefix command)
4. Open **General Information** and copy the **Application ID** (only needed if you
   later use the optional global deploy script).

## 2. Invite the bot

Use the **OAuth2 → URL Generator**:
- Scopes: `bot` **and** `applications.commands`
- Bot permissions: **Manage Roles**, **Kick Members**, **Send Messages**,
  **Embed Links**, **Read Message History**, **View Channels**.

> Drag the bot's role **above** the Officer / strike roles it needs to add/remove,
> and above anyone it might kick. Discord won't let it manage roles or members
> ranked above its own highest role.

## 3. Configure & run

```bash
npm install
cp .env.example .env       # then paste your token into .env
npm start
```

On startup the bot registers its slash commands in every server it's in (instant).

## 4. Run `/setup`

`/setup` opens an interactive panel (needs **Manage Server**). Set:

- **Rank roles** — Officer, HICOM, Upper HICOM, Officer Overseer
- **Strike roles** — Strike 1–4
- **Channels & Member** — EP log channel, OP log channel, and an optional Member role
- **Quotas** — the minimum EP and OP everyone is expected to have

> The optional **Member role** limits who the **EP quota check** targets. If you
> leave it unset, the EP quota check runs on every non-bot member — which could
> strike/kick officers too, so setting it is recommended.

---

## Commands

### Slash commands
| Command | Who can use it | What it does |
|---|---|---|
| `/setup` | Manage Server | Open the config panel |
| `/ep view [user]` | anyone | Show your (or someone's) EP |
| `/ep leaderboard` | anyone | Top 4 members by EP |
| `/ep quotacheck` | Upper HICOM | EP strike ladder (see below) — asks for confirmation first |
| `/op view [user]` | anyone | Show your (or someone's) OP |
| `/op leaderboard` | anyone | Every Officer and their OP |
| `/op quotacheck` | Upper HICOM **or** Officer Overseer | OP strike check |
| `/assign ep <users> <amount>` | Officer | Give EP to one or more members |
| `/assign op <users> <amount>` | HICOM | Give OP to one or more officers |
| `/reset` | Manage Server | Reset EP, OP, or both (logs a snapshot) |

For `/assign`, the `users` field accepts any mix of `@mentions` and raw IDs, so you
can give the same amount to several people at once. Use a **negative** amount to
remove points.

### Prefix command (prefix `-`)
- **`-logep`** — reply to a message that mentions some members, then send `-logep`.
  The bot asks how much EP to give and applies it to everyone mentioned in the
  replied-to message. (Officer role required, same as `/assign ep`.)

---

## How the quota checks work

**`/op quotacheck`** (Officers only):
- OP **below** quota → 1st strike. Already has it → **2nd strike** and the command
  runner is told who got escalated.
- OP **≥ quota + 4** → their OP strikes are cleared.
- Everything is logged to the **OP log channel**.

**`/ep quotacheck`** (members, or just the Member role if set):
- EP **below** quota → next strike up the ladder: none → 1 → 2 → 3 → 4.
- Already on the **4th** strike and still under quota → **kicked**.
- EP **above** quota → highest strike removed (one per run).
- Everything — strikes, removals, and kicks — is logged to the **EP log channel**.
- Because it can kick, it asks you to confirm before running.

(Exactly *at* quota = no change, in both checks.)

---

## Deploying with GitHub + Railway

### A. Push to GitHub

> **Folder structure matters.** `package.json` must sit at the **root of the repo**,
> not inside a subfolder. If you unzip and your repo looks like
> `my-repo/gar-bot/package.json`, Railway won't find it. It should be
> `my-repo/package.json`, with `src/` next to it.

```bash
# from inside the unzipped folder (the one containing package.json)
git init
git add .
git commit -m "EP/OP bot"
git branch -M main
git remote add origin https://github.com/YOUR_NAME/YOUR_REPO.git
git push -u origin main
```

`.gitignore` already keeps `node_modules/`, `.env`, and your local `data/*.json`
out of the repo, so your token never gets committed.

### B. Deploy on Railway

1. **New Project → Deploy from GitHub repo**, pick your repo. Railway reads
   `railway.json` and `package.json` and runs `npm install` then `npm start`.
2. **Variables** (Variables tab):
   - `DISCORD_TOKEN` = your bot token
   - `DATABASE_PATH` = `/data/db.json`
3. **Add a Volume** (right-click the service → Add Volume, or the Volumes tab) and
   set its **mount path to `/data`**. This is what makes EP/OP survive restarts and
   redeploys. The path must match the folder in `DATABASE_PATH` (`/data`).
4. Railway redeploys automatically. Watch the **Deploy Logs** — you should see
   `Health server listening on :…` then `Logged in as <bot>#1234` and
   `Registered 5 commands in "<your server>"`.

That's it. Every `git push` to `main` triggers a fresh deploy.

### Troubleshooting

| Symptom | Fix |
|---|---|
| Build can't find `package.json` | It's nested in a subfolder — move the files so `package.json` is at the repo root (see warning above). |
| `Missing DISCORD_TOKEN` in logs | Add the `DISCORD_TOKEN` variable in Railway. |
| Bot connects but `Used disallowed intents` | Enable **Server Members** + **Message Content** intents in the Developer Portal → Bot tab. |
| EP/OP resets to 0 after every deploy | The volume isn't mounted at `/data`, or `DATABASE_PATH` doesn't point into it. Both must be `/data`. |
| Commands don't appear | Make sure the bot was invited with the `applications.commands` scope, then wait a few seconds and refresh Discord. |
| Bot can't add strikes / kick | Move the bot's role **above** the roles it manages, and grant **Manage Roles** + **Kick Members**. |

> Render works the same way: set `DATABASE_PATH` to a path on a mounted disk, add a
> persistent disk there, and set the start command to `npm start`. The included
> health server means it also runs fine as a Render **Web Service**.

---

## Tweaks you might want

- **Let HICOM also give EP:** in `src/handlers.js`, the EP permission check is one
  line — add `|| hasRole(interaction.member, cfg.hicom_role)`.
- **Change EP leaderboard size:** `getTopEp(..., 4)` in `src/handlers.js`.
- **Change the OP "clear" threshold:** the `quota + 4` checks in `src/quota.js`.
