# Roblox XP Tracker (Discord Bot)

Tracks player XP manually via slash commands, since the bot has no direct
access to the game's data. Users self-report their current XP with
`/checkin`, and the bot calculates gains, pace, and progress toward a
weekly requirement (default: 20,000 XP/week).

## How it works

- Each `/checkin <xp>` records a timestamped XP total for that user.
- The bot compares checkins to work out XP gained, elapsed time, and rate
  (XP/hour and XP/day).
- A rolling 7-day "week" window is tracked per user, starting from their
  first checkin (not necessarily Monday). It automatically rolls over once
  7 days pass — using the most recent known XP as the new week's starting
  point, since exact XP at the boundary moment isn't known.
- All data is stored locally in `xp_data.json`, scoped per Discord server.

## Setup (Windows)

1. **Install Python** (skip if you already have it)
   - Download from https://python.org/downloads
   - Run the installer. On the **first screen**, check the box **"Add python.exe to PATH"** before clicking Install — this is the step people usually miss.
   - After installing, close any open terminal windows and open a new one so the PATH change takes effect.

2. **Create a Discord application & bot**
   - Go to https://discord.com/developers/applications → New Application
   - Go to the "Bot" tab → Add Bot → copy the **Token** (keep this private, don't share it)
   - **Turn on the "Server Members Intent" toggle** on this same "Bot" page, under "Privileged Gateway Intents." This is required for `/importxp` to see everyone in your server — without it, the bot will either fail to start or only see a small cached subset of members instead of your whole server.

3. **Invite the bot to your server**
   - Go to the "OAuth2 → URL Generator" tab
   - Scopes: `bot`, `applications.commands`
   - Bot Permissions: `Send Messages`, `Embed Links`, `Use Slash Commands`, `Manage Roles` (only needed if you plan to use XP milestone roles — see below)
   - Open the generated URL and add the bot to your server

4. **Open a terminal in the bot's folder**
   - Open the folder containing `bot.py` in File Explorer
   - Hold **Shift** and right-click on empty space inside the folder → **"Open PowerShell window here"** (or "Open in Terminal" on Windows 11)

5. **Install dependencies**
   ```
   pip install -r requirements.txt
   ```
   If you get `'pip' is not recognized`, try `python -m pip install -r requirements.txt` instead.

6. **Configure your token**
   - In File Explorer, copy `.env.example` and paste it in the same folder
   - Rename the copy to `.env` (if Windows won't let you start a filename with a dot, rename it to `.env.` with a trailing dot — Windows will drop the trailing dot automatically)
   - Open `.env` in Notepad and replace `your-bot-token-here` with the token you copied in step 2

   While testing, also set `GUILD_ID` in `.env` to your server's ID so slash commands sync instantly (right-click your server icon in Discord → Copy Server ID — you may need to enable Developer Mode first under Discord Settings → Advanced). Leave it blank for production — global sync takes up to an hour but works across every server the bot is in.

7. **Run the bot**
   ```
   python bot.py
   ```
   Leave this terminal window open — closing it stops the bot. If you get `'python' is not recognized`, revisit step 1 and make sure "Add python.exe to PATH" was checked, or try `py bot.py` instead.

## Commands

**"Owner" below means restricted to the bot's Discord application owner specifically** (or its team, if the application is team-owned) — not "Manage Server" in whichever server the command is run in. This covers store/SKU commands since they're tied to *your* actual storefront and Discord Monetization SKUs, not something that makes sense for each server's own admins to reconfigure. Everything else uses normal per-server "Admin" permissions, so if you invite this bot to other servers, their own admins can manage their own tracking/roles/display settings without being able to touch your store setup. See "Packaging the bot for other servers" near the end of this document for more on this.

**⭐ marks a Premium command** — it still uses normal per-server "Admin" permissions, but additionally requires that specific server to have Premium access (a paid guild subscription, or a free exemption you've granted — see "Premium features" below). A non-Premium server's admins see these commands like any other, they just get told the server needs Premium when they try to use one.

| Command | Who | Description |
|---|---|---|
| `/help` | Everyone | Browse every command by category via a dropdown — works in any channel even if you've restricted others to one. |
| `/store` | Everyone | Shows the product menu and a "Visit Store" link button, if one's configured. Also works in any channel. See below. |
| `/setstoreurl url:<link>` | Owner | Sets where the `/store` button goes — your Ko-fi, a website, wherever purchases actually happen. |
| `/clearstoreurl` | Owner | Removes the store link. |
| `/addproduct name: price: [description] [emoji]` | Owner | Adds a product to the `/store` menu (max 25). |
| `/removeproduct name:<exact name>` | Owner | Removes a product by its exact name. |
| `/settings` | Admin (Manage Server) | Opens an interactive panel with buttons and forms for everything below — no need to remember command syntax. See below. |
| `/checkin xp:<number> [user]` | Everyone | Record current total XP. First checkin starts tracking. The `user` option lets an admin check in on someone else's behalf (e.g. if they can't use Discord themselves) — regular members can only check themselves in. |
| `/status [user]` | Everyone | Show weekly progress, rate, and projection for yourself or another user. |
| `/weeklyleaderboard` | Everyone | Ranked list of everyone's XP gained *this week*, with on-track indicators. |
| `/totalleaderboard` | Everyone | Ranked by each person's *actual total XP* (starting point + everything gained since), not just the gain alone. |
| `/crewtotals` | Everyone | The whole crew's combined XP added together — total XP and this week's gain, summed across everyone tracked (including anyone no longer in the server). |
| `/crewlevel` | Everyone | The crew's in-game level (an exponential formula, see below), progress to the next one, and an ETA based on the crew's combined recent pace. |
| `/setcrewlevelformula base_xp: growth_rate:` | Admin | Sets the level-1→2 XP cost and the per-level growth rate `/crewlevel` uses. See "Crew level" below. |
| `/history [user]` | Everyone | Last 10 checkins for a user. |
| `/undo [user]` | Everyone (own checkins); Admin for others | Remove the most recent checkin and revert current XP to the one before it. If it was someone's only checkin, tracking resets entirely. |
| `/setrequirement xp:<number>` | Admin (Manage Server) | Change the weekly XP goal (default 20,000). |
| `/togglerecentrate enabled:<true/false>` | Admin | Show or hide the "Recent Rate" field server-wide. See note below. |
| `/togglecompactleaderboard enabled:<true/false>` | Admin | Switch `/weeklyleaderboard` and `/totalleaderboard` between full-detail and compact one-line-per-person entries. |
| `/setproratethreshold days: hours: minutes:` | Admin | Control how late someone must join to get a prorated requirement (see below). Default: any lateness at all. |
| `/addxprole xp:<number> role:<@role> [exclusive]` | Admin | Auto-assign a role once someone's total XP reaches this amount. Retroactively grants it to anyone who already qualifies. `exclusive: true` (default) makes it part of a tier ladder — reaching a higher tier removes it. `exclusive: false` makes it permanent, kept regardless of what else is earned later (e.g. a standing crew role). |
| `/removexprole role:<@role>` | Admin | Stop auto-assigning a role at its XP milestone. Doesn't strip the role from people who already have it. |
| `/listxproles` | Everyone | Show all configured XP milestone roles and their thresholds. |
| `/syncxproles` | Admin | Re-checks every tracked member's roles right now — fixes leftover role clutter without waiting for everyone's next checkin. |
| `/progresschart [user]` | Everyone | Shows a line chart of XP over time, built from that person's checkin history. Needs at least 2 checkins. |
| `/profile [user]` | Everyone | Shows a rendered profile card: avatar, colored border, Crew XP, rank, this week's gain, and a progress bar. See below. |
| `/previewtier tier:` | Everyone ⭐ | See what a configured banner tier would look like on your own card, without needing the role. See "Banner tiers" below. |
| `/setbordercolor user: color: [custom_hex]` | Admin ⭐ | Assign a profile border color — a preset from the palette, or an exact hex code. This is the *unpaid/default* look; see "Banner tiers" below for the store-linked version. |
| `/removebordercolor user:<@user>` | Admin ⭐ | Reset a user's border color to the default. |
| `/listbordercolors` | Everyone ⭐ | Show all available preset border colors. |
| `/toggleanimatedprofile enabled:<true/false>` | Admin | Turn the animated rainbow-tier `/profile` card on or off server-wide (static image instead of a GIF). See below. |
| `/addbannertier role: name: mode: color1: [color2] priority:` | Admin ⭐ | Ties a profile banner look to a role — e.g. a role your store/Ko-fi integration grants on purchase. See "Banner tiers" below. |
| `/removebannertier role:<@role>` | Admin ⭐ | Removes a role's banner tier. |
| `/listbannertiers` | Everyone ⭐ | Shows all configured banner tiers, their roles, and their colors. |
| `/addskurole sku_id: role: name:` | Owner | Auto-grants a role while a member has an active Discord purchase/subscription for a SKU, and removes it when that ends. See "SKU role automation" below. |
| `/removeskurole sku_id:<id>` | Owner | Stops auto-granting a role for a SKU. |
| `/listskuroles` | Admin | Shows all configured SKU → role mappings. |
| `/setpremiumsku sku_id:` | Owner | Sets the guild-subscription SKU that grants Premium. See "Premium features" below. |
| `/addpremiumguild [guild_id]` | Owner | Grants a server free Premium — no subscription needed. Defaults to the current server if left blank. |
| `/removepremiumguild [guild_id]` | Owner | Removes a server's free Premium exemption. Defaults to the current server if left blank. |
| `/listpremiumguilds` | Owner | Shows every server with Premium, split into free (exempt) and paid (subscribed). |
| `/exportdata [file_format]` | Admin ⭐ | Downloads everyone's current stats (XP, baseline, gains, last checkin) as a CSV or Excel file. |
| `/backup` | Admin ⭐ | Downloads a **complete** snapshot — every setting plus every tracked user, not just stats. See below. |
| `/restore file:<.json>` | Admin | Restores everything from a `/backup` file. Replaces the entire server's data — confirmation required. Not Premium-gated — see "Backup & restore" below for why. |
| `/analytics` | Admin ⭐ | Server-wide activity: retention week-over-week, most active day, top gainer, average gain. See below. |
| `/setadminlogchannel channel:<#channel>` | Admin | Log destructive/data-altering admin actions to a channel. See below. |
| `/clearadminlogchannel` | Admin | Turn off admin action logging. |
| `/setchannel channel:<#channel>` | Admin | Restrict `/checkin`, `/status`, `/weeklyleaderboard`, `/totalleaderboard`, `/history`, `/undo`, and `/progresschart` to one channel. Admin/config commands still work anywhere. |
| `/clearchannel` | Admin | Remove the channel restriction. |
| `/setannouncechannel channel:<#channel>` | Admin ⭐ | Post a public congratulations message whenever someone earns an XP milestone role. |
| `/clearannouncechannel` | Admin ⭐ | Turn off milestone announcements. |
| `/setweeklypost channel:<#channel>` | Admin | Auto-post the weekly leaderboard shortly before each week ends. Requires a shared server week. |
| `/clearweeklypost` | Admin | Turn off the auto-posted weekly leaderboard. |
| `/setinactivitychannel channel:<#channel>` | Admin ⭐ | Ping anyone with zero checkins as the week nears its end. Requires a shared server week. |
| `/clearinactivitychannel` | Admin ⭐ | Turn off inactivity reminder pings. |
| `/setinactivitythreshold days: hours: minutes:` | Admin ⭐ | How long before week-end the inactivity ping fires (default: 24 hours). |
| `/toggleinactivitybehindpace enabled:<true/false>` | Admin ⭐ | Whether inactivity pings also include people who checked in but are behind pace, not just people with zero checkins (default: off — zero checkins only). |
| `/toggleleaderboardpagination enabled:<true/false>` | Admin | Switch leaderboards between Previous/Next paging and the default single truncated embed. |
| `/setbaseline user:<@user> starting_xp:<number>` | Admin | Set or correct someone's all-time starting XP — this is what `/totalleaderboard` calculates total gain from. |
| `/importxp file:<.csv or .xlsx> [existing_users] [announce_milestones]` | Admin ⭐ | Bulk-set XP for many users at once from a spreadsheet. `existing_users` controls what happens to people already tracked: skip (default), check in (mass check-in, keeps history), or reset. `announce_milestones` (default: no) controls whether XP role milestones hit during this import get posted publicly. See below. |
| `/undoimport` | Admin ⭐ | Reverts **everyone's** data back to exactly how it looked right before the last `/importxp` ran — the only way to undo a whole batch at once, since `/undo` only handles one person's most recent checkin at a time. Only reverts the single most recent import. |
| `/fullreset user:<@user> [clear_history]` | Admin | Reset **both** weekly and all-time totals at once, starting fresh from their current XP. `clear_history: true` (default) also wipes their stored checkin history. Unlike `/removeuser`, they stay registered — no need to re-checkin to restart tracking. |
| `/resetweek user:<@user>` | Admin | Manually restart someone's weekly window right now (0 days elapsed). |
| `/setweekprogress user:<@user> [days] [hours] [minutes] [week_start_xp]` | Admin | Manually set how far into the week someone is, in plain days/hours/minutes (e.g. `days: 3, hours: 12` treats them as if their week started 3.5 days ago). Optionally also override their starting XP for the week. |
| `/setweekprogressall [days] [hours] [minutes] [reset_gains]` | Admin | Same as above, but applies to **every** tracked user at once — no need to update people one at a time. `reset_gains: true` also zeroes out everyone's gained-so-far. |
| `/removeuser user:<@user>` | Admin | Wipe a user's tracking data entirely. Only works if they're still in the server — Discord's picker can't select someone who's left. |
| `/removeuserbyid discord_id:<id>` | Admin | Same as above, but by raw Discord ID — works even if the person has left or been kicked. |
| `/removestaleusers` | Admin | Bulk-remove tracking data for everyone tracked who's no longer in the server. Asks for confirmation first. |
| `/removeallusers` | Admin | Wipe **everyone's** tracking data at once. Asks for confirmation with Confirm/Cancel buttons before doing anything — never fires on a single click. |

## Settings panel

Running `/settings` opens an interactive panel (admins only) instead of typing out command parameters. As of this version, it's organized into categories via a dropdown rather than one long row of buttons — pick a category, see just the controls relevant to it, and use **⬅ Back** to return to the category list.

**Home screen:**
- **Category dropdown** — jump into any of the categories below
- **Manage a User** — shows a dropdown to pick a member, then buttons for: Set Week Progress, Set Starting XP, Set Border Color, Reset Week, Undo Last Checkin, Full Reset, Remove User
- **Refresh** — updates the panel's numbers without re-running the command

**⏱️ Requirement & Week**
- **Set Requirement** — pops up a form to type in a new weekly XP goal
- **Sync Week (Everyone)** — Days/Hours/Minutes form plus whether to reset everyone's gained-so-far, applied server-wide (same as `/setweekprogressall`)
- **Prorate Threshold** — Days/Hours/Minutes form controlling how late someone must join to get a prorated requirement (same as `/setproratethreshold`)

**🎨 Display**
- **Recent Rate: ON/OFF** — toggles the "Recent Rate" field server-wide (see note below)
- **Compact Leaderboard: ON/OFF** — toggles shorter leaderboard entries
- **Pagination: ON/OFF** — toggles Previous/Next leaderboard paging (same as `/toggleleaderboardpagination`)
- **Animated Profiles: ON/OFF** — toggles the rainbow-tier `/profile` GIF server-wide (same as `/toggleanimatedprofile`)

**🏅 XP Roles**
- **Add XP Role** — pick a role from a dropdown, then a form for the XP amount and whether it's a ladder tier or permanent (same as `/addxprole`)
- **Remove XP Role** — pick a role from a dropdown to stop auto-assigning it (same as `/removexprole`)
- **Sync XP Roles Now** — re-checks everyone's roles immediately (same as `/syncxproles`)

**🎗️ Banner Tiers**
- **Add / Edit Tier** — pick a role from a dropdown, then a form for the tier's name (the badge text shown on `/profile`), mode (`solid`/`gradient`/`rainbow`), color(s), and priority. Picking a role that already has a tier opens the form pre-filled with its current values — this is also how you rename a tier's badge or change its colors later, not just how you create a new one (same underlying data as `/addbannertier`).
- **Remove Tier** — pick a role from a dropdown to remove its tier (same as `/removebannertier`)
- **List Tiers** — shows every configured tier, its role, mode, and colors (same as `/listbannertiers`)

**📢 Channels & Automation**
- **Tracking Channel** — restrict tracking commands to one channel (same as `/setchannel`)
- **Announce Channel** — where milestone role announcements post (same as `/setannouncechannel`)
- **Weekly Post Channel** — where the auto-posted weekly leaderboard goes (same as `/setweeklypost`)
- **Inactivity Channel** — where inactivity reminder pings post (same as `/setinactivitychannel`)
- **Admin Log Channel** — where destructive/data-altering admin actions get logged (same as `/setadminlogchannel`)
- **Inactivity: Zero Checkins Only / +Behind Pace** — toggles whether inactivity pings also flag behind-pace members (same as `/toggleinactivitybehindpace`)

**🛒 Store**
- **Set Store URL** — a form for the link `/store`'s button opens (same as `/setstoreurl`)
- **Add Product** — a form for name, price, description, and emoji (same as `/addproduct`)
- **Remove Product** — pick a product from a dropdown to remove it (same as `/removeproduct`)

**⚠️ Danger Zone**
- **Remove All Users** — wipes every tracked user in the server; asks for confirmation first, same as `/removeallusers`
- **Remove Departed Members** — wipes tracking data only for people no longer in the server; asks for confirmation first, same as `/removestaleusers`

Every button/dropdown expires after 5 minutes of inactivity — just run `/settings` again if that happens. It works alongside all the individual slash commands (`/setrequirement`, `/resetweek`, etc.), which still work exactly as before if you prefer typing them directly.

## Bulk importing starting XP from a spreadsheet

If you already have a list of members and their starting XP in Excel or Google Sheets, `/importxp` can set everyone up in one go instead of them each running `/checkin` individually.

**1. Prepare your file** with a header row and these columns (an example `import_template.xlsx` is included alongside this README if you'd rather start from a working file than build one from scratch):

| Column | Required? | Notes |
|---|---|---|
| `discord_id` | Recommended | Their numeric Discord user ID — the most reliable way to match. Right-click a user → Copy User ID (needs Developer Mode on). |
| `username` | Use if no `discord_id` | Their Discord username, current server nickname, or a Roblox username that appears *inside* their nickname. A leading `@` is fine and gets stripped automatically. See matching rules below. |
| `starting_xp` (or `xp`) | Required | Their starting XP as a whole number. |

Example CSV:
```
discord_id,username,starting_xp
123456789012345678,CoolPlayer123,45000
,another_user,12500
```
You only need `discord_id` **or** `username` per row, not both — `discord_id` is checked first if present.

**Matching rules** (checked in this order):
1. `discord_id` — exact, always reliable.
2. Exact match against their Discord username, server nickname, or global display name.
3. **Substring match** — if `username` appears *anywhere inside* their nickname, it counts as a match. This covers servers using a bot (like RoWifi or Bloxlink) that formats nicknames as something like `DiscordName | RobloxUsername` — a plain list of Roblox usernames will match those nicknames without any extra setup.
4. If a `username` value matches more than one member (ambiguous), that row is flagged rather than guessed — you'll need to resolve it manually with a `discord_id`.

**2. Export as `.csv` or `.xlsx`** — both work directly; `.xlsx` needs the `openpyxl` package, which is already in `requirements.txt`.

> ⚠️ **If you use `discord_id`**: format that column as **Text**, not a number, before typing IDs into it (in Google Sheets/Excel: select the column → Format → Number → Plain text). Discord IDs are 17-19 digits long, which exceeds what a spreadsheet numeric cell can store exactly — as a real number, the last few digits can get silently rounded off, corrupting the ID. This doesn't affect the `username` column, which is plain text either way — if you're not sure, skip `discord_id` entirely and just use `username`.

**3. Run `/importxp`**, attach the file, and choose what to do with anyone already tracked via `existing_users`:
- **Skip** (default) — only registers brand-new people; leaves everyone already tracked untouched
- **Check in** — treats the imported XP as a fresh checkin for existing users, exactly like they ran `/checkin` themselves. Their baseline, weekly progress, and checkin history all stay intact — this is the one to use for a routine bulk update (e.g. "here's everyone's current XP from this week's screenshots"), acting as a mass check-in instead of a reset.
- **Reset** — wipes and restarts existing users' tracking, same as `/fullreset`. Use this for a genuine do-over, not routine updates — it discards their history.

New users are always registered regardless of which option you pick for existing ones. If you have XP milestone roles configured (see "XP milestone roles" below) and want a public announcement for anyone who hits one during this import, set `announce_milestones: true` — it defaults to `false` so a routine bulk update doesn't spam the announcement channel.

**4. Check the results** — the bot replies with a summary: how many were newly tracked, skipped, **ambiguous** (matched more than one member), or **unmatched**. For unmatched rows, double-check the spelling matches what's actually in their nickname; for ambiguous ones, add a `discord_id` for those specific rows. Either way, you can always patch up the leftovers manually with `/setbaseline`.

**Undoing an import**: `/undo` (the regular per-person command) is *not* the right tool for reverting a bulk import — it only touches one person's single most recent checkin, and after `existing_users: Reset` there's no prior checkin left to revert to; `/undo` would just delete that person's tracking entirely rather than restore their pre-import numbers. For reverting the whole batch correctly — including recovering history that a "Reset" import wiped out — use `/undoimport` instead, which restores everyone's data to exactly how it looked right before that import ran. It can only undo the single most recent import (not a chain of several), and running a second import overwrites the ability to undo the first.

## XP milestone roles

The bot can automatically assign a Discord role once someone's total XP crosses a threshold you set — e.g. reach 75,000 XP, get the "Veteran" role.

**Setup requirements** — two things need to be true or role assignment silently fails:
1. The bot needs the **Manage Roles** permission. If you invited it before this feature existed, re-invite using an updated OAuth2 URL with that permission checked (see Setup step 3), or just enable it directly for the bot's role in Server Settings → Roles.
2. The bot's own role must be positioned **above** any role it's assigning, in Server Settings → Roles (drag it up the list). Discord doesn't let any bot assign a role ranked higher than its own — this trips people up constantly, so `/addxprole` checks for it up front and tells you clearly if it's a problem, rather than failing silently later.

**How it behaves:**
- Configure thresholds with `/addxprole xp:75000 role:@Veteran` (or the panel's "Add XP Role" button, which walks you through picking the role first, then a form for the XP amount and whether it's permanent).
- Each role you configure is either **ladder** or **sticky**:
  - **Ladder** (`exclusive: true`, the default) — part of a tier progression. Only the single highest ladder tier someone currently qualifies for is kept; reaching a new one removes the old one. Good for rank-style roles like Bronze/Silver/Gold.
  - **Sticky** (`exclusive: false`) — granted once earned and never removed by this system, no matter what else someone reaches later. Good for a standing role that shouldn't disappear just because someone leveled up — e.g. keep a "Crew" ping role forever, while a separate "0 XP" starter role still gets cleared out once they pass 15,000 XP.
- Both types re-evaluate on every checkin: ladder tiers can go up *or down* if a correction changes someone's XP; sticky roles are only ever added, never stripped by this system.
- New thresholds are applied **retroactively** — anyone who already qualifies gets the role immediately, not just on their next checkin.
- Removing a threshold with `/removexprole` only stops *future* assignments; it doesn't strip the role from anyone who already has it.
- Roles are re-evaluated **on that person's next checkin** — not instantly for everyone the moment you change a setting. If you update the bot, add a bunch of thresholds, or import a big batch of members, some people might carry outdated role combinations until they check in again. Run `/syncxproles` (or the panel's "Sync XP Roles Now") to force a full re-check of everyone immediately instead of waiting.
- Roles are checked on `/checkin` and `/importxp` — the two places XP actually gets recorded.
- `/listxproles` marks each configured role 🪜 ladder or 🔒 sticky so you can see the setup at a glance.

## Weekly date range

Both `/status` and `/weeklyleaderboard` show a "📅 Week Runs" field with the actual calendar dates the current week spans (e.g. "July 27 → August 3"), rendered via Discord's timestamp tags so it automatically displays in each viewer's own local timezone rather than a fixed UTC string.

On `/weeklyleaderboard`, this only shows a single shared range if your server uses `/setweekprogressall` to sync everyone's week; otherwise it notes that ranges vary per person (check `/status` for an individual's own window).

`/status` also shows a text progress bar (e.g. `████████░░░░░░ 57%`) tracking gained-this-week against the requirement — the prorated amount, for anyone in a prorated first week, rather than the flat full number.

## Restricting commands to one channel

By default every command works in any channel. If you'd rather keep XP tracking confined to a dedicated channel (e.g. `#xp-tracking`) instead of cluttering general chat:

```
/setchannel channel:#xp-tracking
```

This restricts `/checkin`, `/status`, `/weeklyleaderboard`, `/totalleaderboard`, `/history`, `/undo`, and `/progresschart` — the commands people use often — to that one channel. Trying to run them elsewhere gets a clear message pointing to the right channel instead of silently failing.

**Admin/config commands are deliberately exempt** — `/settings`, `/addxprole`, `/setrequirement`, and so on still work in any channel, since admins often manage settings from a separate admin-only channel. Run `/clearchannel` to remove the restriction entirely.

## Exporting data

```
/exportdata file_format:CSV
```
or
```
/exportdata file_format:Excel (.xlsx)
```

Downloads a file with everyone's current stats: Discord ID, username, current XP (under a `starting_xp` header), baseline XP, total gained, this week's starting XP, this week's gain, and the timestamp of their last checkin. Useful for backups, spreadsheet analysis outside Discord, or handing records to someone who doesn't have bot access.

**The `starting_xp` column name matches what `/importxp` expects**, so an exported file can be re-imported directly with no reformatting — export → (optionally edit) → `/importxp` that same file. What re-importing does depends on which `existing_users` mode you pick: **check in** treats each person's exported `current_xp` as a fresh checkin (history preserved) — the right choice for most re-imports; **reset** treats it as a full restart (history wiped), better suited to disaster recovery than routine use.

## Backup & restore

`/exportdata` only exports current stats — it doesn't capture your settings (requirement, week sync, XP roles, channels, toggles). For a true full snapshot:

```
/backup
```

Downloads a JSON file containing **everything**: every setting in `/settings`, every XP milestone role, and every tracked user's full history — not just their current numbers. Keep this file somewhere safe (not just in Discord, where messages can be deleted).

```
/restore file:<the .json from /backup>
```

Uploads that file back and **completely replaces** the server's current settings and tracked data with it — shows you how many users are in the backup and when it was taken before asking for confirmation, since this discards everything that's happened since that backup, not just undoes one action. This is real disaster-recovery, not the scoped undo that `/undoimport` provides for a single import.

A backup taken with an older version of the bot (missing settings that didn't exist yet) restores fine — any keys not present in the file get filled in with current defaults rather than causing an error.

**`/restore` is deliberately not Premium-gated, even though `/backup` is.** Premium status lives in the same shared data file as everything else — if that file is ever lost (a botched redeploy, a host wiping storage, etc.), the server's premium status is lost right along with it, which would make `/restore` itself unreachable at exactly the moment it's needed most: fixing a lost data file requires premium, but premium was part of what got lost. Since actually running `/restore` requires already possessing a legitimate backup file for that specific server — which only exists if the server was using the bot in the first place — there's no real exploit here, just closing off a lockout trap. The bot owner also always passes any Premium check regardless (see `require_premium()` in the code), as a second layer of the same protection.

## Crew level

```
/crewlevel
/setcrewlevelformula base_xp:1000 growth_rate:1.15
```

Roblox gym crews often have an in-game "crew level" that levels up as the crew's combined XP grows — this recreates that as a bot command, since the game itself doesn't expose it anywhere convenient to check. It's the same exponential math as the standalone **Crew Level Tracker** spreadsheet (a calculator built earlier in this project, kept as a separate tool since not everyone wants this wired into the bot) — set the same `base_xp`/`growth_rate` in both places and they'll always agree.

**How the curve works:** `base_xp` is how much combined XP the crew needs to go from level 1 to level 2. `growth_rate` is how much *more* every level after that costs than the one before it, as a multiplier — `1.15` means each level needs 15% more than the last. This compounds fast: with those defaults, level 16 alone requires roughly 47,500 combined XP, not a simple 16× multiple of the base.

**Calibrating it to match the real in-game numbers:** the defaults (1,000 / 1.15) are a starting guess, not a known-correct value — Roblox doesn't publish the actual formula. Run `/crewlevel`, compare the level it reports against your crew's real level in-game, and adjust `base_xp`/`growth_rate` with `/setcrewlevelformula` until they match. Two known real data points (e.g. "we were level 8 at X combined XP, hit level 9 at Y") pin down both numbers precisely — the same calibration approach the spreadsheet's "Calibration Check" cell is built around.

**What "combined XP" means here:** the same definition as `/crewtotals`' "Total XP (crew)" — every tracked member's current XP added together, including members no longer in the server. Not `/totalxpgained`-style "since they started tracking" — this is the crew's actual current total, since that's what the real in-game level is presumably based on.

**The ETA is a real projection, not a guess:** it sums each member's own current-week `rate_per_day` (the same rate `/status` shows individually) into one combined crew-wide daily rate, then divides the remaining XP to the next level by that rate. A member with a currently negative rate (e.g. right after an admin correction) is floored to 0 for this calculation rather than dragging the whole crew's estimate down — one person's data cleanup shouldn't make the ETA worse. If the crew's combined rate is 0 (no one's gained anything recently), it says so plainly instead of showing an infinite or nonsensical ETA.

## Analytics

```
/analytics
```

A Premium-gated snapshot of how the server's actually doing, not just individual stats:

- **Tracked Members** and **Active This Week** — how many people have checked in at all vs. in the last 7 days.
- **Retention vs Last Week** — what fraction of last week's active members are still checking in this week. This is the number worth watching over time; a server can have a healthy member count while quietly losing engagement, and total-member-count alone won't show that.
- **Avg Gain (Active Members)** — the average of `gained_this_week` across only people who've actually checked in, so a pile of inactive members doesn't drag the number down and hide what's really happening with the people still showing up.
- **Most Active Day** — the day of the week with the most checkins across everyone's history, useful for timing announcements or scheduled posts around when people are actually active.
- **Top Gainer This Week** — same leader `/weeklyleaderboard` would show, surfaced here too since it's relevant context alongside the rest.

Retention and most-active-day are both derived from stored checkin history, which is capped per user (`MAX_CHECKINS_STORED`, 50 by default) to keep the data file from growing forever — for a very long-tracked, very active member, their oldest checkins may have aged out, which can slightly skew those two numbers for that person specifically. Doesn't affect Tracked Members, Active This Week, or Avg Gain, which are based on current totals rather than history.

## Admin action log

With this many destructive commands now available (`/fullreset`, `/removeallusers`, `/undoimport`, `/restore`, and more), it's worth having a record of who did what, especially once more than one person has admin access.

```
/setadminlogchannel channel:#admin-log
```

Posts a log entry to that channel whenever an admin runs a data-affecting command — who did it, what it was, and relevant details (e.g. old value → new value, or how many users were affected). Covers: `/fullreset`, `/removeuser`, `/removeuserbyid`, `/removeallusers`, `/removestaleusers`, `/resetweek`, `/setweekprogress`, `/setweekprogressall`, `/setbaseline`, `/setrequirement`, `/addxprole`, `/removexprole`, `/importxp`, `/undoimport`, `/restore`, and admin-performed `/undo`s (not self-undos — those aren't really "admin actions" on someone else).

Routine display toggles (`/togglerecentrate`, `/togglecompactleaderboard`, etc.) are deliberately **not** logged, to keep the log meaningful rather than noisy — it's focused on things that change tracked data or affect the whole server, not cosmetic preferences.

Turn it off with `/clearadminlogchannel`.

## Progress charts

```
/progresschart user:@SomeMember
```

Renders a line chart of that person's XP over time, built from their stored checkin history, and posts it as an image. Needs at least 2 checkins to plot anything — with only 1, there's no trend to show yet, so the bot says so instead of generating an empty chart. Like everything else, this respects the channel restriction if one is set.

Chart generation depends on the `matplotlib` package (already in `requirements.txt`). If it's somehow missing, the command explains what to install rather than crashing.

## Profile cards, animation & banner tiers

```
/profile user:@SomeMember
```

Renders an image card — avatar with a colored ring, name, rank, Crew XP, this week's gain, and a progress bar toward the weekly requirement — instead of a plain text embed. Rank is calculated the same way as `/totalleaderboard`: by actual total XP, so it includes each person's starting point, not just XP gained while tracked.

Rank #1 gets a gold star and gold-accented rank text regardless of banner tier, so the top spot always stands out. Rank text otherwise uses a brightened, boosted-saturation version of the tier's primary color — the raw color looks great as a thick border/ring, but some colors (dark, muted ones especially) are too hard to read comfortably as plain text against the card's dark background, so text specifically gets a readability boost while the border, ring, and progress bar keep the true color. The card is drawn at high internal resolution and downscaled for smooth, anti-aliased edges rather than the jagged look raw drawing would otherwise produce, then shipped at 1560×630 — a higher native resolution than the card's actual design size, so it still looks sharp in Discord's small inline chat preview rather than blurry. Long display names are truncated based on actual rendered pixel width, not a fixed character count, so wide and narrow character names both fit correctly instead of some overflowing while others get cut short unnecessarily, and truncation leaves extra room automatically when a tier badge (below) is also on the card, so the two never overlap.

**If a member holds a banner tier, its name appears as a small colored badge in the top-right corner** (e.g. "SUPPORTER", "ELITE") — pulled straight from the `name` you gave it in `/addbannertier`, colored with the tier's own primary color, with the badge's text automatically flipped between near-black and near-white depending on which reads better against that particular color.

The rank-1 star and the rank-change up/down indicators are drawn as actual shapes, not Unicode symbol characters (★/▲/▼) — Windows' Arial, a common fallback font, doesn't reliably include those glyph blocks, which could make them silently fail to render (or show as a blank box) depending on what font the host has available. Drawing them as vector shapes instead means they always render correctly regardless of platform or installed fonts.

**Personal bests** — if a member has ever set a new single-day or single-week XP record, a small "Best Day X · Best Week Y" line shows up on their card between their rank and their stats. Both are tracked automatically, not something anyone has to set: best-day is the most XP gained between one checkin and the next on a given calendar day (UTC), best-week is the highest weekly total a completed week has ever closed with. A brand-new member with no records yet simply doesn't get the line at all, rather than showing "Best Day 0."

**Daily rank change** — next to the rank, the card shows how much someone's rank has moved since the last daily snapshot: ▲2 in green for climbing, ▼1 in red for dropping, or — for no change. A background task takes one snapshot per UTC day for every server with tracked users (no shared week required for this specific feature, unlike the scheduled post and inactivity pings). The indicator only appears once at least one snapshot has happened — a server's very first day using the bot won't show a rank-change badge yet, since there's nothing to compare against.

### Banner tiers (store-linked)

```
/addbannertier role:@Supporter name:"Supporter" mode:gradient color1:#FF3EA5 color2:#3498DB priority:1
/addbannertier role:@Elite name:"Elite" mode:rainbow color1:#FFD700 priority:2
```

Each tier ties a **profile banner look** to a Discord role — the intended flow is: someone buys a tier from your store (`/store`, below), your store's own integration (e.g. Ko-fi's Discord role sync) or you manually grants them the matching role, and the bot picks up on that role the next time they run `/profile`. There's no payment processing inside the bot itself; it only reacts to the role.

Three modes:
- **`solid`** — one flat color (`color1`), same as a classic border color.
- **`gradient`** — a two-color sweep across the background, border, avatar ring, and progress bar fill, from `color1` (left) to `color2` (right, required for this mode). All four elements pull from the exact same gradient image, so they always stay in sync rather than each being tinted separately.
- **`rainbow`** — the border outline, avatar ring, and progress-bar fill slowly cycle through the full hue wheel, starting from `color1`, as an animated GIF; the background stays a fixed dark gray rather than tinting along with it. This is intentionally the most premium-looking option — reserve it for your top tier. A full color cycle takes a little over 4 seconds; it's deliberately slow and smooth rather than fast, since rapid color cycling reads as flashing and can be uncomfortable for motion/light-sensitive members. Keeping the background and text fixed and only animating the border/ring/bar isn't just a style choice — a GIF shares one 256-color palette across every frame, so the less of the card that's actually changing color, the more of that budget goes into a smooth border sweep instead of being spread thin across a full-card gradient and showing up as visible banding or noise.

If a member holds roles for more than one tier (e.g. they kept an old role after upgrading), `priority` decides which one wins — higher number = more premium, and only the single highest one is ever shown, never a blend. Set it however you like; it just needs to be higher for the tiers you consider more valuable. `/listbannertiers` shows everything currently configured, highest priority first. `/removebannertier` takes a tier away from a role.

**`/previewtier tier:Elite`** lets anyone see what a tier would look like on their *own* card — real avatar, name, rank, and stats, with that tier's colors/animation applied — without needing the role first. The `tier` field autocompletes from whatever's configured in that server. This is meant as a pre-purchase preview (marketing, basically: "here's what buying Elite actually gets you"), so the response is ephemeral — only the person who ran it can see it, both to keep channels clean and so a preview never gets mistaken for someone's actual current profile. It's still gated by `/setpremiumsku`/`/addpremiumguild` like the rest of the banner tier commands, since previewing a look the server can't currently render anyway wouldn't be very useful.

**Members with no tier role** fall back to the older, individually-assigned system: whatever `/setbordercolor` has set for them (a preset or custom hex, always `solid`), or the Slate default if nothing's been set. That command still works exactly as before — it's a good fit for a free/no-purchase-required look, or for one-off manual color assignments outside the tier system.

`/toggleanimatedprofile enabled:false` is a server-wide override for the `rainbow` mode specifically — solid and gradient tiers are always static regardless of this setting. Turn it off if you'd rather trade the animation for lower bandwidth/faster `/profile` responses (an animated card takes a few seconds longer to render and is several megabytes, versus well under a second and under 200KB for a static one), or for accessibility if anyone in your server is sensitive to on-screen motion. With it off, `rainbow`-tier members still get their tier's card, just as a static frame at `color1` instead of animating.

Card rendering depends on the `Pillow` package (already in `requirements.txt`). If it's somehow missing, `/profile` explains what to install rather than crashing. Font rendering automatically adapts to whatever's available on the host — it checks common Linux, Windows, and macOS font locations and falls back to a built-in font if none are found, so the card looks right on your Windows PC during testing and on whatever you eventually host it on.

## SKU role automation (Discord purchases & subscriptions)

```
/addskurole sku_id:1234567890123456789 role:@Supporter name:"Supporter"
/addskurole sku_id:9876543210987654321 role:@Elite name:"Elite Monthly"
```

If you're selling through **Discord's own Monetization system** (SKUs configured in the Developer Portal — one-time purchases or subscriptions bought right inside Discord, not through an external site), the bot can grant and remove roles automatically as those purchases happen, with no manual step:

- **On purchase** (`on_entitlement_create`) — the mapped role is granted within seconds.
- **On refund or removal** (`on_entitlement_delete`) — the role is removed within seconds.
- **On subscription cancellation** (`on_entitlement_update`) — Discord doesn't cut access off immediately; the member keeps it until their current billing period ends, so the role stays too until that happens.
- **Every hour**, a reconciliation pass (`reconcile_sku_roles`) re-checks every mapped role against Discord's actual current entitlement list and corrects any drift — this is what guarantees a role gets removed once a subscription actually lapses, and it also catches purchases/cancellations that happened while the bot was offline (Discord doesn't replay missed events from downtime).

Combine this with a banner tier on the same role (see above) and the whole chain becomes automatic: someone buys a SKU in Discord → they get the role within seconds → their next `/profile` shows the tier's banner look. No manual `/setbordercolor` or role-clicking needed anywhere in that chain.

This is separate from the `/store`/Ko-fi setup below — that's for external storefronts where you (or a third-party integration) still have to grant the role yourself. If you're selling through Discord's native purchase flow instead, this is the fully-automated version of the same idea. A server can use either, both, or neither; they don't conflict since they both just end up granting a role.

Requires `discord.py>=2.4.0` (already the pinned minimum in `requirements.txt`) and a bot application with Monetization enabled and at least one SKU created in the Developer Portal. `sku_id` is the numeric SKU ID from that page, not a product name.

## Premium features

```
/setpremiumsku sku_id:1122334455667788990
/addpremiumguild
```

A separate tier from the per-user SKU roles above: this one is a **guild subscription** — a whole server gets unlocked, not one member. `/setpremiumsku` tells the bot which SKU ID (from the Developer Portal) represents that subscription. Once set, a server counts as Premium if it either has an active subscription for that SKU, or is on the free-exemption list.

**The ⭐-marked commands in the table above are Premium-gated:** mass data management (`/importxp`, `/exportdata`, `/undoimport`), inactivity pings (`/setinactivitychannel` and friends), data safety (`/backup`, `/restore`), milestone announcements (`/setannouncechannel`, `/clearannouncechannel`), and customizable profiles (`/addbannertier`, `/removebannertier`, `/listbannertiers`, `/setbordercolor`, `/removebordercolor`, `/listbordercolors`). A non-Premium server's admins can still see and try these commands — they're told the server needs Premium, same tone as a normal permission error, not hidden entirely.

This isn't just a command-time check. If a server's subscription lapses, three things that were already running stop on their own rather than quietly continuing forever: inactivity pings stop firing, milestone announcements stop posting, and `/profile` cards revert to the default look (banner tiers and custom border colors both fall back) — all re-checked live, not just when they were originally configured. The alternative — checking only at setup time — would let a server subscribe once, configure everything, cancel, and keep every perk permanently.

**Free exceptions, e.g. your own server:**

```
/addpremiumguild
```

Run with no arguments in a server to exempt that server specifically — this is the common case, including your own. Pass a `guild_id` to exempt a different server without being in it. `/removepremiumguild` undoes it, and `/listpremiumguilds` shows everything currently exempt or subscribed, by name where the bot can resolve it.

Like the SKU role automation, this is kept in sync by two layers: `on_entitlement_create`/`on_entitlement_delete`/`on_entitlement_update` react within seconds, and `reconcile_premium_guilds` re-derives the truth from Discord's actual entitlement list once an hour as a safety net (catches anything the gateway events alone can't fully cover — see "SKU role automation" above for why that safety net matters). All four premium commands (`/setpremiumsku`, `/addpremiumguild`, `/removepremiumguild`, `/listpremiumguilds`) are Owner-only, same restriction as the store commands — this is a business decision about who gets access, not a per-server setting.

## Store

```
/setstoreurl url:https://ko-fi.com/yourpage
/addproduct name:"Gold Border" price:$5 description:"Unlocks the Gold profile border" emoji:🏆
```

`/store` shows everyone a menu of whatever products you've configured, plus a **Visit Store** button linking wherever purchases actually happen — a Ko-fi page, a website, a Discord shop channel, wherever you're set up to take payments. The bot doesn't process payments itself; this is a catalog and a signpost, not a checkout.

**This is the other half of the banner tier system** described above — `/store` is where members find out *what's for sale and where to buy it*, and the role they receive after buying is what `/addbannertier` keys off of to show the right banner on their `/profile` card. If you're selling through an external storefront, that role still has to come from somewhere: your store's own Discord role-sync integration if it has one (e.g. Ko-fi's), or granted manually. If you're selling through Discord's own Monetization/SKU system instead, see "SKU role automation" above for the fully automatic version — no manual step at all.

Products are stored per-server (up to 25, a Discord embed limit) with a name, price (any free-text format — `$5`, `500 Robux`, `Free`, whatever fits how you actually sell things), an optional description, and an optional emoji. Remove one with `/removeproduct` using its exact name, or via the settings panel's dropdown if you'd rather not worry about exact spelling.

## Public milestone announcements

```
/setannouncechannel channel:#achievements
```

Once set, the bot posts a congratulations message to that channel whenever someone **earns a new XP milestone role** (see "XP milestone roles" above) — e.g. "@Someone just earned @Veteran!" Only role *additions* trigger an announcement, never removals (so someone getting demoted between tiers doesn't generate an awkward public message).

**Bulk operations stay quiet by default, on purpose.** Retroactively granting a newly-added role with `/addxprole`, running `/syncxproles`, and importing via `/importxp` all skip individual announcements by default — otherwise a big batch would flood the channel with dozens of messages at once. `/addxprole`'s retroactive grants and `/syncxproles` always stay quiet (they report their own summary instead, e.g. "X roles granted"). `/importxp` is the one exception: it has an `announce_milestones` toggle if you *do* want a public post for every milestone hit during that specific import — useful for a big "everyone's XP is now official" batch update where you want the celebration, not just for routine data corrections.

Run `/clearannouncechannel` to turn announcements off again.

## Scheduled weekly leaderboard post

```
/setweeklypost channel:#xp-results
```

Auto-posts the weekly leaderboard to that channel once, shortly before each week ends — a background check runs every 15 minutes looking for that moment.

**Requires a shared server week** (set up via `/setweekprogressall`) — without one, there's no single "week end" moment for every tracked person to schedule around, so the command explains this and won't let you set it up until you have one.

**Timing detail that matters:** the post captures the week's *final* results, not the just-reset numbers of the new week. It fires in roughly the last 20 minutes before the actual rollover — deliberately *before* the boundary, since checking after it would mean the data has already reset to the new week's near-zero starting point. Run `/clearweeklypost` to turn this off.

## Inactivity pings

```
/setinactivitychannel channel:#xp-reminders
/setinactivitythreshold days:0 hours:24 minutes:0
```

As the week nears its end, the bot @mentions relevant members in the channel you set, once — not repeatedly. `/setinactivitythreshold` controls how early this fires (default: 24 hours before week-end); set it earlier for a bigger heads-up, or closer to the deadline for a final-hours nudge. Like the scheduled post, this also **requires a shared server week**.

**By default, this only catches people with zero checkins that week** — it doesn't look at pace at all. Someone who checked in once but is way behind on XP gets no ping by default, same as someone who's comfortably on track. To also flag people who *have* checked in but are behind pace to hit the requirement:

```
/toggleinactivitybehindpace enabled:true
```

With this on, the ping message splits into two clearly labeled groups — "haven't checked in at all" and "checked in, but behind pace" — so it's obvious which situation each person is in rather than lumping everyone into one undifferentiated list.

Run `/clearinactivitychannel` to turn pings off entirely.

## Leaderboard pagination

By default, a leaderboard that's too long to fit in one Discord embed gets truncated with a note about how many entries were cut. As an alternative:

```
/toggleleaderboardpagination enabled:true
```

Switches `/weeklyleaderboard` and `/totalleaderboard` (and the scheduled auto-post) to Previous/Next buttons instead, paging through 10 entries at a time so nobody gets cut off the list. Turn it back off with `enabled:false` to return to the single-embed view.

## Bot status (presence)

The bot's Discord status automatically rotates every 45 seconds between a few useful lines shown under its name — "Watching 12 XP trackers," "Listening to /checkin," "Watching 3 servers," and a pointer to `/settings`. This needs no setup and isn't configurable per-server since a bot only has one global status across every server it's in.

Worth knowing: this is the extent of what a Discord *bot* can do here. True Rich Presence — the image, buttons, and party info you see on some Discord user profiles for games — is a feature of Discord's RPC SDK for user game clients, not something available to bots through the bot API. A bot's status is limited to one activity verb (Playing/Watching/Listening/Competing) plus a short text line, which is what's implemented here.

To customize the rotation text, edit the list inside `build_presence_statuses()` near the top of `bot.py`.

## Notes & limitations

- **`/restore` has no undo of its own**: unlike `/undoimport`, restoring a
  backup doesn't snapshot the state it's about to overwrite — it's meant
  for disaster recovery (a corrupted data file, a bad migration), not
  routine experimentation. If you want to try `/restore` safely, run
  `/backup` immediately beforehand to create a fallback point you can
  restore back to if needed. This action is logged to the admin log
  channel if one is configured, same as every other data-affecting
  command.
- **Scheduled posts and pings need the bot online at the right moment**:
  both the weekly auto-post and inactivity pings only fire during a
  narrow window before the week ends (about 20 minutes for the post,
  whatever you set for pings). If the bot happens to be offline or
  restarting during that exact window, that week's scheduled message is
  silently skipped — there's no catch-up mechanism, since by the time the
  bot comes back the week's data may have already rolled over and the
  "final results" would no longer be available. This matters most if
  you're running the bot on a machine that isn't reliably always-on (see
  "Keeping it running" below) — for anything mission-critical, a proper
  always-on host removes this risk entirely.
- **Milestone announcements need Send Messages in that channel**: if you
  point `/setannouncechannel` at a channel the bot can't post in (private
  channel it lacks access to, permission overrides blocking it, etc.),
  announcements silently fail rather than erroring loudly at setup time —
  Discord's own send failures are swallowed so a permissions hiccup in the
  announcement channel never breaks the actual role assignment. If
  announcements aren't showing up, double-check the bot can see and post
  in that channel.
- **Removing kicked/departed members**: `/removeuser`'s member picker is
  provided by Discord itself, and it can only show people currently in
  the server — once someone's kicked or leaves, they simply don't appear
  as an option anymore, with no error explaining why. This isn't
  something the bot can work around directly, so two alternatives exist:
  `/removeuserbyid` (paste their raw Discord ID instead of picking from a
  list) for one person, or `/removestaleusers` to bulk-clean everyone
  tracked who's no longer in the server in one confirmation-gated action.
- **Departed members still count toward crew-wide totals**: `/crewtotals`
  and per-person leaderboard entries include everyone in the tracking
  data, whether or not they're still in the server — someone who was
  kicked mid-week still contributed real XP, so it stays counted. If you
  want a departed member's numbers fully out of the totals (not just
  hidden from the member list), remove their data with `/removeuserbyid`
  or `/removestaleusers` first.


- **First-week requirement is automatically prorated for late joiners**:
  if your server uses a shared week (via `/setweekprogressall`) and someone
  checks in for the first time partway through it — say, 5 hours before
  the week ends — holding them to the full flat requirement would be
  unwinnable by design. The bot scales their requirement down to the fair
  share of the time they actually had (5 hours out of 7 days ≈ 3% of the
  full requirement), shown as "prorated" on `/status` and marked with 🆕 on
  `/weeklyleaderboard`. This only affects someone's first partial week —
  once they roll into a full 7-day week, the full requirement applies
  normally. It has no effect for servers that don't use a shared week
  anchor, since each person's own first checkin already starts their own
  full week.
  - **How late is "late enough" to prorate is configurable** with
    `/setproratethreshold` (or the panel's "Prorate Threshold" button). By
    default, *any* lateness at all triggers proration — joining with 6
    days left still prorates slightly. Set a threshold (e.g. `hours: 24`)
    to only prorate genuinely late joins — someone with more than a day
    left gets the full requirement, and only those joining with a day or
    less remaining get scaled down. Set it to `0` to disable proration
    entirely.


- **Leaderboards auto-truncate for very large rosters**: Discord embeds
  have a hard 4096-character limit. If your tracked-user list is big enough
  that the full-detail format would exceed it, the bot automatically cuts
  the list short and tells you how many more there are, rather than the
  command failing outright. Turning on `/togglecompactleaderboard` fits
  significantly more people before hitting that limit.
- **Admin check-ins are logged, not silent**: when an admin uses `/checkin`
  with the `user` option, the resulting embed's footer says who checked
  them in, so it's clear the entry wasn't self-reported by that member.


- **"Recent Rate" can look wrong after `/setweekprogress` / `/setweekprogressall`**:
  Recent Rate is calculated from real elapsed time between someone's last
  two actual checkins — it has no idea the week's start time was manually
  moved. Avg Rate, on the other hand, *does* use the (possibly backdated)
  week start, so after a manual week-progress edit the two numbers can
  genuinely disagree and look inconsistent, even though both are "correct"
  by their own definition. If that's confusing for your group, use
  `/togglerecentrate enabled:false` (or the panel's toggle button) to hide
  Recent Rate server-wide until you want it back.

- **Already had the bot running before this update?** After pulling this
  version of `bot.py`, you must also go to the Discord Developer Portal →
  your application → Bot tab → enable **"Server Members Intent"** under
  Privileged Gateway Intents, then save. If you skip this, the bot will
  crash on startup with a `PrivilegedIntentsRequired` error (or, on older
  bot.py versions without the intent set at all, `/importxp` would silently
  see only a few cached members instead of your whole server, causing
  everyone to show up as "unmatched" with no obvious error).

- **Starting XP matters — two different baselines are tracked**:
  - **Weekly baseline** (`week_start_xp`) resets automatically every 7 days
    (or whenever `/resetweek` / `/setweekprogress` runs). It's what
    `/status` and `/weeklyleaderboard` use.
  - **All-time baseline** (`baseline_xp`) is set once, on someone's very
    first `/checkin`, and never changes on its own. It's what
    `/totalleaderboard` uses to show total XP gained since tracking began.
  - Because the very first checkin *is* the baseline, gain is correctly 0
    at that moment — you don't need to do anything special to avoid a
    crazy first-time rate. But if someone was already tracked before this
    total-leaderboard feature existed, or if their first checkin wasn't
    actually accurate (e.g. it was mid-session, not their true starting
    point), use `/setbaseline` (or the panel's "Set Starting XP" button)
    to correct it — otherwise `/totalleaderboard` may show a total that's
    too high or too low for that person.

- **All durations display as days/hours/minutes** (e.g. `3d 12h 30m`,
  `2h 5m`, `45m`) instead of decimals — this covers "Since Last Checkin,"
  "Recent Rate," "Days Left in Week," and the total leaderboard's tracked
  time. Setting week progress works the same way: `/setweekprogress` and
  `/setweekprogressall` (and their `/settings` panel equivalents) take
  plain `days`/`hours`/`minutes` fields instead of a single decimal like
  `3.5`.
- **Recent Rate needs at least 10 minutes between checkins** to display.
  Two checkins close together in time (e.g. a few minutes apart) produce a
  tiny, noisy interval that gets wildly extrapolated when converted to an
  hourly rate — a 5-XP gain over 2 minutes would otherwise show as "150
  XP/hr," which is technically the math but not a meaningful pace. Below
  that threshold, the bot shows "too soon for a reliable rate" instead.
- **Self-reported data — a single last-minute checkin can't be verified**:
  the bot only ever knows XP at the moments someone checks in. If someone
  checks in once at the start of the week and once right before it ends,
  the bot reports the difference as "this week's gain" — it has no way to
  confirm that XP was earned steadily rather than being misreported, or to
  catch a typo. To flag this, any user with only 1 checkin in the current
  week gets a "⚠️ Low Data Confidence" note on `/status` and a 🔸 marker on
  `/weeklyleaderboard`. The best mitigation is encouraging frequent checkins
  (daily or every couple of days) so there's a real trend to look at rather
  than one trusted data point.
- **New users automatically align to the group's week**: once an admin
  runs `/setweekprogressall`, that week alignment is remembered for the
  server. Anyone who checks in for the *first time* after that point has
  their week aligned to the same schedule as everyone else, instead of
  starting their own fresh 7-day window from the moment they join. Their
  starting XP for the week is still just "whatever they reported first,"
  though — the bot has no way to know what they had at the *true* start of
  the week if they check in mid-week for the first time.
- **Undo only goes back one step**: `/undo` removes the single most recent
  checkin. Running it repeatedly walks back further, one checkin at a time.
  In the rare case a checkin triggered a weekly rollover, undoing it won't
  automatically un-roll the week — use `/setweekprogress` afterward if that
  matters for your situation.
- **Lower XP than last time**: if someone reports XP lower than their last
  checkin (e.g. an in-game reset), the bot logs it but flags it. Use
  `/resetweek` to give them a clean slate if needed.
- **Week boundaries are approximate**: since XP is only known at checkin
  times, the exact XP at the 7-day mark is estimated using the latest
  known value.
- **Import file size**: Discord attachment limits apply (typically 25MB,
  higher on boosted servers) — a spreadsheet with even thousands of rows is
  nowhere near that, so this shouldn't be a practical concern.
- **Windows hides file extensions by default**, so `.env` might silently
  end up saved as `.env.txt`. In File Explorer, go to the "View" tab and
  check "File name extensions" to confirm the file is really named `.env`
  with nothing after it.
- Data persists in `xp_data.json` next to `bot.py`. Back this file up if
  you care about historical data — deleting it wipes all tracking.

## Packaging the bot for other servers

The bot already supports being added to more than one server with no code changes — every setting (`requirement`, roles, banner tiers, channels, etc.) is stored per-server, so servers never see or affect each other's data or configuration. What needed deliberate handling was the **store and SKU commands** (`/setstoreurl`, `/clearstoreurl`, `/addproduct`, `/removeproduct`, `/addskurole`, `/removeskurole`): those aren't really "per-server settings" the way a weekly XP goal is — they're tied to *your* actual storefront and *your* Discord Monetization SKUs (configured once, application-wide, in the Developer Portal). If any admin in any server could reconfigure them, someone could point `/store`'s button at their own link, list fake products, or map a role to a real SKU ID and have it silently pick up entitlements from your actual paying customers. So those six commands — and their `/settings` panel equivalents — check specifically for **you** (or your team, if the application is team-owned), not "Manage Server" in whichever server the command happens to run in. Everything else stays normal per-server admin control.

Two ways to get the bot into another server:

**If you're adding it yourself** (e.g. your own alt server, or a community's admin invites you to set it up): go to the [Discord Developer Portal](https://discord.com/developers/applications) → your application → **OAuth2 → URL Generator**. Under **Scopes**, check `bot` and `applications.commands`. Under **Bot Permissions**, check at minimum `Send Messages`, `Embed Links`, `Use Slash Commands`, and `Manage Roles` (needed for XP roles, banner tiers, and SKU role automation). Copy the generated URL and open it — it'll prompt you to pick which server to add the bot to, same as any bot invite.

**If you want other people to be able to add it to their own servers**: the same invite URL works for them too, as long as your application's **Bot → Public Bot** toggle (in the Developer Portal) is turned on. Share that URL however makes sense — your own server, a website, wherever. Whoever adds it will need "Manage Server" in their own server to complete the invite, same as any bot, but once it's in, their local admins only get the "Admin" commands from the table above — the store/SKU ones stay locked to you specifically, everywhere, automatically, with no per-server setup needed on your end.

One thing worth knowing: `/settings` is an ephemeral response (only visible to whoever ran it) specifically so its buttons and dropdowns can't be seen or clicked by anyone else in the channel — Discord doesn't restrict component interactions to the original command invoker on its own, so a public settings message would otherwise let anyone who could see it attempt to click through it.

## Keeping it running

The bot only runs while `python bot.py` is active in a terminal window. A few options for keeping it up without babysitting a terminal:

- **Simplest**: just leave the terminal window open on your PC while you want the bot online. Closing the window or shutting down the PC stops it.
- **Minimize the hassle**: create a `start_bot.bat` file in the folder containing:
  ```
  @echo off
  python bot.py
  pause
  ```
  Double-click it to launch the bot without opening a terminal manually.
- **Real 24/7 uptime**: your PC needs to stay on and awake, which most people don't want long-term. The rest of this section covers getting there properly.

## Setting up Git

Doing this now — even before you host anywhere — makes every future update a `git push` instead of manually re-copying files.

**1. Install Git** (skip if `git --version` already works in your terminal)
- Download from https://git-scm.com/download/win, run the installer, defaults are fine throughout
- Close and reopen your terminal afterward (same PATH-refresh requirement as installing Python)

**2. Initialize the repo** — in your project folder:
```
git init
git add .
git commit -m "Initial commit"
```
The `.gitignore` in this project already excludes `.env` (your bot token) and `xp_data.json` (your live tracking data) — those should never end up in version control, since `.env` is a secret and `xp_data.json` is per-deployment data, not source code.

**3. Create a GitHub repo and push**
1. Go to https://github.com/new, create a repo (private is fine, and probably what you want here)
2. GitHub will show you commands like these — run them in your project folder:
   ```
   git remote add origin https://github.com/<your-username>/<repo-name>.git
   git branch -M main
   git push -u origin main
   ```

**From now on, updating the bot looks like:**
```
git add .
git commit -m "describe what changed"
git push
```
No more downloading files and dragging them into a folder — new versions of `bot.py` just replace the old one locally, then this pushes the change up.

## Hosting on Railway (free tier available)

Railway can run the bot 24/7 and auto-redeploy every time you `git push`, once connected.

1. Go to https://railway.app and sign up (GitHub login is easiest)
2. **New Project → Deploy from GitHub repo** → pick your repo
3. Railway auto-detects Python and installs `requirements.txt` automatically
4. **Set your environment variable**: in the project → Variables tab → add `DISCORD_TOKEN` with your bot token as the value (this replaces your local `.env` file — Railway injects it the same way `python-dotenv` does locally, so no code changes needed)
5. **Set the start command** if Railway doesn't infer it: Settings → Deploy → Start Command → `python bot.py`
6. Deploy. Check the Logs tab for `Synced N commands. Logged in as ...` to confirm it's running

**One important gotcha**: `xp_data.json` lives on Railway's filesystem, which is **not persistent by default** — a redeploy can wipe it. Before relying on this for real, add a **Volume** (Railway's persistent storage feature, free tier includes one) mounted at your project folder, so `xp_data.json` survives redeploys. Without this, every `git push` would silently reset everyone's tracked XP.

Fly.io and Render work similarly (GitHub-connected auto-deploy, environment variable for the token, persistent volume for the data file) if you'd rather compare options — the setup shape is nearly identical across all three.

