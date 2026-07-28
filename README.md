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

| Command | Who | Description |
|---|---|---|
| `/settings` | Admin (Manage Server) | Opens an interactive panel with buttons and forms for everything below — no need to remember command syntax. See below. |
| `/checkin xp:<number> [user]` | Everyone | Record current total XP. First checkin starts tracking. The `user` option lets an admin check in on someone else's behalf (e.g. if they can't use Discord themselves) — regular members can only check themselves in. |
| `/status [user]` | Everyone | Show weekly progress, rate, and projection for yourself or another user. |
| `/weeklyleaderboard` | Everyone | Ranked list of everyone's XP gained *this week*, with on-track indicators. |
| `/totalleaderboard` | Everyone | Ranked by each person's *actual total XP* (starting point + everything gained since), not just the gain alone. |
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
| `/exportdata [file_format]` | Admin | Downloads everyone's current stats (XP, baseline, gains, last checkin) as a CSV or Excel file. |
| `/setchannel channel:<#channel>` | Admin | Restrict `/checkin`, `/status`, `/weeklyleaderboard`, `/totalleaderboard`, `/history`, `/undo`, and `/progresschart` to one channel. Admin/config commands still work anywhere. |
| `/clearchannel` | Admin | Remove the channel restriction. |
| `/setannouncechannel channel:<#channel>` | Admin | Post a public congratulations message whenever someone earns an XP milestone role. |
| `/clearannouncechannel` | Admin | Turn off milestone announcements. |
| `/setweeklypost channel:<#channel>` | Admin | Auto-post the weekly leaderboard shortly before each week ends. Requires a shared server week. |
| `/clearweeklypost` | Admin | Turn off the auto-posted weekly leaderboard. |
| `/setinactivitychannel channel:<#channel>` | Admin | Ping anyone with zero checkins as the week nears its end. Requires a shared server week. |
| `/clearinactivitychannel` | Admin | Turn off inactivity reminder pings. |
| `/setinactivitythreshold days: hours: minutes:` | Admin | How long before week-end the inactivity ping fires (default: 24 hours). |
| `/toggleleaderboardpagination enabled:<true/false>` | Admin | Switch leaderboards between Previous/Next paging and the default single truncated embed. |
| `/setbaseline user:<@user> starting_xp:<number>` | Admin | Set or correct someone's all-time starting XP — this is what `/totalleaderboard` calculates total gain from. |
| `/importxp file:<.csv or .xlsx> [overwrite]` | Admin | Bulk-set starting XP for many users at once from a spreadsheet. See below. |
| `/fullreset user:<@user> [clear_history]` | Admin | Reset **both** weekly and all-time totals at once, starting fresh from their current XP. `clear_history: true` (default) also wipes their stored checkin history. Unlike `/removeuser`, they stay registered — no need to re-checkin to restart tracking. |
| `/resetweek user:<@user>` | Admin | Manually restart someone's weekly window right now (0 days elapsed). |
| `/setweekprogress user:<@user> [days] [hours] [minutes] [week_start_xp]` | Admin | Manually set how far into the week someone is, in plain days/hours/minutes (e.g. `days: 3, hours: 12` treats them as if their week started 3.5 days ago). Optionally also override their starting XP for the week. |
| `/setweekprogressall [days] [hours] [minutes] [reset_gains]` | Admin | Same as above, but applies to **every** tracked user at once — no need to update people one at a time. `reset_gains: true` also zeroes out everyone's gained-so-far. |
| `/removeuser user:<@user>` | Admin | Wipe a user's tracking data entirely. Only works if they're still in the server — Discord's picker can't select someone who's left. |
| `/removeuserbyid discord_id:<id>` | Admin | Same as above, but by raw Discord ID — works even if the person has left or been kicked. |
| `/removestaleusers` | Admin | Bulk-remove tracking data for everyone tracked who's no longer in the server. Asks for confirmation first. |
| `/removeallusers` | Admin | Wipe **everyone's** tracking data at once. Asks for confirmation with Confirm/Cancel buttons before doing anything — never fires on a single click. |

## Settings panel

Running `/settings` opens an interactive panel (admins only) instead of typing out command parameters:

- **Set Requirement** — pops up a form to type in a new weekly XP goal
- **Sync Week (Everyone)** — pops up a form with separate Days/Hours/Minutes fields plus whether to reset everyone's gained-so-far, applied to the whole server at once (same as `/setweekprogressall`)
- **Prorate Threshold** — pops up a Days/Hours/Minutes form controlling how late someone must join to get a prorated requirement (same as `/setproratethreshold`)
- **Add XP Role** — pick a role from a dropdown, then a form for the XP amount required (same as `/addxprole`)
- **Remove XP Role** — pick a role from a dropdown to stop auto-assigning it (same as `/removexprole`)
- **Sync XP Roles Now** — re-checks everyone's roles immediately (same as `/syncxproles`)
- **Set Tracking Channel** — shows a channel dropdown to restrict tracking commands to (same as `/setchannel`)
- **Set Announce Channel** — shows a channel dropdown for milestone announcements (same as `/setannouncechannel`)
- **Set Weekly Post Channel** — shows a channel dropdown for the auto-posted weekly leaderboard (same as `/setweeklypost`)
- **Set Inactivity Channel** — shows a channel dropdown for inactivity reminder pings (same as `/setinactivitychannel`)
- **Pagination: ON/OFF** — toggles leaderboard paging (same as `/toggleleaderboardpagination`)
- **Manage a User** — shows a dropdown to pick a member, then buttons for: Set Week Progress, Set Starting XP, Reset Week, Undo Last Checkin, Full Reset, Remove User
- **Remove All Users** — wipes every tracked user in the server; asks for confirmation first, same as `/removeallusers`
- **Remove Departed Members** — wipes tracking data only for people no longer in the server; asks for confirmation first, same as `/removestaleusers`
- **Recent Rate: ON/OFF** — toggles the "Recent Rate" field server-wide (see note below); the button's label and color reflect the current state
- **Compact Leaderboard: ON/OFF** — toggles shorter leaderboard entries; also reflects current state
- **Refresh** — updates the panel's numbers without re-running the command

The panel expires after 5 minutes of inactivity — just run `/settings` again if that happens. It works alongside all the individual slash commands (`/setrequirement`, `/resetweek`, etc.), which still work exactly as before if you prefer typing them directly.

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

**3. Run `/importxp`** and attach the file. By default, anyone already being tracked is **skipped** (not overwritten) — pass `overwrite: true` if you want the spreadsheet to replace their existing data instead (this acts like `/fullreset` combined with `/setbaseline` for that person).

**4. Check the results** — the bot replies with a summary: how many were newly tracked, skipped, **ambiguous** (matched more than one member), or **unmatched**. For unmatched rows, double-check the spelling matches what's actually in their nickname; for ambiguous ones, add a `discord_id` for those specific rows. Either way, you can always patch up the leftovers manually with `/setbaseline`.

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

Downloads a file with everyone's current stats: Discord ID, username, current XP, baseline XP, total gained, this week's starting XP, this week's gain, and the timestamp of their last checkin. Useful for backups, spreadsheet analysis outside Discord, or handing records to someone who doesn't have bot access. The CSV format uses the same column names `/importxp` expects, so round-tripping (export → edit → re-import) works without reformatting.

## Progress charts

```
/progresschart user:@SomeMember
```

Renders a line chart of that person's XP over time, built from their stored checkin history, and posts it as an image. Needs at least 2 checkins to plot anything — with only 1, there's no trend to show yet, so the bot says so instead of generating an empty chart. Like everything else, this respects the channel restriction if one is set.

Chart generation depends on the `matplotlib` package (already in `requirements.txt`). If it's somehow missing, the command explains what to install rather than crashing.

## Public milestone announcements

```
/setannouncechannel channel:#achievements
```

Once set, the bot posts a congratulations message to that channel whenever someone **earns a new XP milestone role** (see "XP milestone roles" above) — e.g. "@Someone just earned @Veteran!" Only role *additions* trigger an announcement, never removals (so someone getting demoted between tiers doesn't generate an awkward public message).

**Bulk operations stay quiet on purpose.** If you import 50 people at once via `/importxp`, retroactively grant a newly-added role with `/addxprole`, or run `/syncxproles`, none of those post individual announcements — only an organic `/checkin` does. Otherwise a big import would flood the announcement channel with dozens of messages at once. Those bulk commands still report their own summary (e.g. "X roles granted") in their normal response, just not as separate public posts per person.

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

As the week nears its end, the bot @mentions anyone tracked who has **zero checkins so far that week** in the channel you set, once — not repeatedly. `/setinactivitythreshold` controls how early this fires (default: 24 hours before week-end); set it earlier for a bigger heads-up, or closer to the deadline for a final-hours nudge. Like the scheduled post, this also **requires a shared server week**.

Run `/clearinactivitychannel` to turn pings off.

## Leaderboard pagination

By default, a leaderboard that's too long to fit in one Discord embed gets truncated with a note about how many entries were cut. As an alternative:

```
/toggleleaderboardpagination enabled:true
```

Switches `/weeklyleaderboard` and `/totalleaderboard` (and the scheduled auto-post) to Previous/Next buttons instead, paging through 10 entries at a time so nobody gets cut off the list. Turn it back off with `enabled:false` to return to the single-embed view.

## Notes & limitations

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

