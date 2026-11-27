# Database storage – ideas & plan

> **Status:** proposal / work in progress – nothing of this is implemented yet.
> Discussion: [deso85/Antony#1](https://github.com/deso85/Antony/issues/1)

## Goal

Antony currently stores all data in JSON files. In the future the data shall be stored in a database
(see issue #1). This document collects the current state, the ideas and the open questions so the
migration can be done step by step.

## Current state

All persistent data is written and read through two helpers in `bot.antony.utils.Utils`:

- `Utils.saveJSONData(...)`
- `Utils.loadJSONData(...)`

The files are stored below `flatfile.path` (see `antony.properties`).

| Area | Class | File(s) | Kind of data |
|---|---|---|---|
| Guild settings | `GuildController` | `guilds/<id> - <name>/<id>.json` | channels, roles, command permissions |
| Users | `UserController` | `guilds/<id> - <name>/user/<userId>.json` | user data per guild |
| Voice mutes | `MuteReaction` | `guilds/<id> - <name>/voice.mute.json` | muted user IDs |
| Husbandry reports | `AAMHBController` | `guilds/<id> - <name>/hb.overdues.json`, `hb.lastchecked.json` | overdue reports, last check |
| Reminders | `ReminderController` | `antony.reminder.json` | active reminders |
| Giveaways | `GiveawayController` | `antony.giveaways.json` | active giveaways |
| Notifications | `NotificationController` | `antony.notifications.config.json`, `antony.notifications.pending.json` | notification lists, pending notifications |
| Soft bans | `SoftbanController` | `antony.softbanneduser.json` | soft banned users |
| White/black/watch lists | `WhiteListController`, `BlackListController`, `WatchListController` | `antony.whitelist.json`, `antony.blacklist.json`, `antony.watchlist.json` | word / user lists |
| Deletion pairs | `RedFlagReaction` | `antony.deletionpair.json` | message pairs to delete |
| Shopping lists | `ShoppingController` | `shopping/shoppingLists.json` | categories and items |
| Antcheck cache | `AntcheckController` | `antcheck/currencies.json`, `products.json`, `shops.json`, `shops.blacklisted.json`, `species.json`, `variants.json` | cached data of the antcheck.info API |
| Antcheck notifications | `AntcheckNotificationController` | `antcheck/availabilityNotificationRequests.json` | availability notification requests |
| Emergency texts | `EmergencyCmd` | `antony.emergency.*.json` | static help texts |

Not on `main` yet, but on open feature branches:

- Hibernation reminders (`feature/hibernation-alarm`): `antony.hibernation.json`
- Role reaction channel (`feature/role-reaction-cleanup`): new field in the guild settings

### Problems of the current approach

- **No atomic writes:** files are overwritten directly. If the bot crashes while writing, the file can be corrupted.
- **No locking:** events, timer threads and commands read and write the same files without synchronisation.
- **Many disk reads:** e.g. every `GuildController.get...Channel()` call reads the guild JSON file again.
- **Folder names depend on the guild name** (`<id> - <name>`), so a renamed guild needs its folder to be renamed (`GuildController.changeGuildName()`).
- **No queries:** e.g. "all reminders that are due" means loading and filtering the whole list.

### Removed

An unused SQLite + Hibernate + Weld (CDI) setup (`bot.antony.db`, `bot.antony.service`, `META-INF/beans.xml`)
and its dependencies were removed on this branch. It was never called and mixed `javax.*` and `jakarta.*` APIs.
Hibernate is not planned for now.

## Proposal

### Database: SQLite

- Assuming Antony runs as a single instance (see open questions): SQLite needs no separate server, the database is a single file.
- Backups stay simple: copy one file (ideally via SQLite's online backup / `VACUUM INTO` while the bot is running).
- Driver: [`org.xerial:sqlite-jdbc`](https://github.com/xerial/sqlite-jdbc).
- A server database (PostgreSQL / MariaDB) only makes sense if several instances or external tools (e.g. a web dashboard)
  shall access the data. The repository layer below keeps that option open.

### Access: lightweight instead of an ORM

- Plain JDBC or [JDBI](https://jdbi.org/) (small library for SQL with object mapping).
- Schema changes as versioned SQL scripts with [Flyway](https://github.com/flyway/flyway)
  (`V1__init.sql`, `V2__...`), applied automatically on startup.
- No automatic schema generation (like Hibernate's `hbm2ddl.auto=update`) – every schema change is a reviewed script.

### Repository layer

One interface per area, the controllers only talk to the interface:

```java
public interface ReminderRepository {
    List<Reminder> findAll();
    List<Reminder> findDue(Instant now);
    void save(Reminder reminder);
    void delete(long id);
}
```

Two implementations per area during the migration:

- `JsonReminderRepository` – wraps the current JSON code (no change in behaviour)
- `SqliteReminderRepository` – the new implementation

Which one is used is decided at startup (e.g. a property `storage.type = json | sqlite`).

### Migration of existing data

- On startup: if a JSON file exists and the corresponding table is empty, the data is imported.
- The JSON file is kept (e.g. renamed to `*.json.migrated`) as a backup.
- The import is idempotent and logged.

## Step-by-step plan

Each step is its own small pull request.

1. **Remove unused DB code** – done on this branch.
2. **Infrastructure:** SQLite connection, Flyway, `storage.type` property, empty schema `V1__init.sql`.
3. **First area:** reminders (small, self-contained) – repository interface, JSON + SQLite implementation, import, tests.
4. **Further small areas:** giveaways, soft bans, white/black/watch lists, hibernation reminders.
5. **Notifications** (`NotificationController`, pending notifications).
6. **Guild settings and users** – used almost everywhere, therefore last. Also removes the dependency on the guild name in the storage path.
7. **Clean up:** remove the JSON implementations once all areas run on SQLite (keep the import for a while).

Probably no migration needed:

- **Emergency texts:** static content, can stay as files (or move to resources).
- **Antcheck cache:** data of an external API that can be fetched again – a cache table is optional and has low priority.

## Draft schema (first areas)

Only a starting point for discussion – column names follow the current Java fields.

```sql
-- V1__init.sql
CREATE TABLE reminder (
    id              INTEGER PRIMARY KEY,
    guild_id        TEXT    NOT NULL,
    user_id         TEXT    NOT NULL,
    channel_id      INTEGER NOT NULL,
    text            TEXT    NOT NULL,
    end_epoch_sec   INTEGER NOT NULL
);
CREATE INDEX idx_reminder_end ON reminder(end_epoch_sec);

CREATE TABLE softban (
    user_id         TEXT    PRIMARY KEY,   -- currently a global list (no guild ID)
    user_name       TEXT
);

CREATE TABLE list_entry (
    list_name       TEXT    NOT NULL,   -- whitelist / blacklist / watchlist
    value           TEXT    NOT NULL,
    PRIMARY KEY (list_name, value)
);
```

## Open questions

Asked in [deso85/Antony#1](https://github.com/deso85/Antony/issues/1):

1. Is it okay to go without Hibernate/CDI and use something lightweight (JDBC/JDBI + Flyway)?
2. SQLite – or a server database (PostgreSQL/MariaDB) later on?
3. How is the bot run (single server, Docker, …) and how are backups done?
4. Which area should be migrated first, which one should be left alone for now?

Further points to decide:

- Location of the database file – new property (e.g. `db.path`) or below `flatfile.path`?
- Backup strategy: automatic daily backup by the bot, or external?
- Should the JSON implementation stay as fallback, or be removed completely in the end?
