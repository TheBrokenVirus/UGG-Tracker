# Privacy Policy — Ultimate Gym Game Tracker (Discord Bot)

**Effective date:** 7/28/26
**Last updated:** 7/28/26

This Privacy Policy explains what data UGG Tracker ("the Bot," "we," "us") collects when a server adds it, how that data is stored and used, and what rights you have over it. This Bot is operated by Kelen McDonald ("the Bot Operator").

This policy is written to be read alongside [Discord's own Privacy Policy](https://discord.com/privacy), which governs Discord itself. This policy only covers what the Bot specifically does.

---

## 1. What this Bot does

UGG Tracker is an XP-tracking bot for Roblox game crews. Since the Bot cannot read game data directly, members self-report their XP totals using Discord slash commands (e.g. `/checkin`), and the Bot calculates progress, rates, and rankings from those self-reported numbers.

## 2. What data we collect

We collect only what's needed to provide this tracking functionality:

**Per tracked member, within each server:**
- Their **Discord user ID** (a numeric identifier — not their username; see note below)
- **Self-reported XP values** they or a server admin enter via commands, with a timestamp for each entry
- A running history of these entries (used to calculate rate, pace, and progress over time)

**Per server, configured by admins:**
- The weekly XP requirement, and any custom settings (week schedule, XP-based role thresholds, designated channels for announcements/reminders, display preferences)
- Configured Discord role IDs and channel IDs (these identify server structure, not individuals)

**We do NOT persist Discord usernames or nicknames.** They are looked up live from Discord's own systems each time they're needed for display or for matching a spreadsheet import — never written to our stored data.

**We do NOT collect:**
- Message content. The Bot does not request or use Discord's Message Content privileged intent — it only reads structured slash command input.
- Passwords, payment information, or Roblox account credentials.
- IP addresses, device information, or voice data.
- Any data from servers the Bot has not been added to, or from users who never interact with a Bot command.

## 3. Admin actions and logging

If a server admin enables the optional Admin Action Log feature, a record of certain admin-performed actions (e.g. resetting or removing a user's data, changing server settings) is posted as a message in a Discord channel the admin chooses. **This log lives in that Discord channel as ordinary Discord messages** — it is not stored in the Bot's own data files, and is subject to Discord's own message retention and deletion, and to that server's permissions.

## 4. How data is stored

All tracked data is stored in a local data file controlled by the Bot Operator, scoped per Discord server. It is not sold, shared with third parties, or used for advertising. It is used only to power the Bot's tracking features within the server(s) it's added to.

## 5. Data retention

Data for a tracked member is retained for as long as the server continues tracking them, or until:
- A server admin removes it (via commands such as `/removeuser`, `/removeallusers`, or `/removestaleusers`), or
- The server is removed from the Bot's data by the Bot Operator.

**Removing the Bot from a server does not automatically delete that server's stored data.** If you want a server's data fully deleted, an admin should run the appropriate removal command(s) before or after removing the Bot, or contact the Bot Operator directly (see Section 8).

Server admins can also download a full copy of a server's data at any time (`/backup`) or its current statistics (`/exportdata`).

## 6. Your rights

If you are a member of a server using this Bot, you can:
- **Request removal of your data** by asking a server admin to run `/removeuser` (or the equivalent command) for you, or by contacting the Bot Operator directly.
- **See what's tracked about you** via the Bot's own commands (e.g. `/status`, `/history`), which are visible to you and server admins.

If you are a resident of the EU/UK, California, or another jurisdiction with data protection laws, you may have additional rights (such as access, correction, portability, or restriction of processing). Contact the Bot Operator using the details in Section 8 to exercise these rights.

## 7. Children's privacy

This Bot is intended for use in accordance with Discord's own Terms of Service, which require users to meet Discord's minimum age requirements. We do not knowingly target or collect data from children under the age required by Discord's Terms in your region.

## 8. Contact

Questions about this policy or requests regarding your data can be sent to:

**kelentmcdonald@gmail.com**


## 9. Changes to this policy

We may update this policy as the Bot's features change. Material changes will be reflected by updating the "Last updated" date above. Continued use of the Bot after changes take effect constitutes acceptance of the updated policy.

---