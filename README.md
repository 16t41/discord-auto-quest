# *Hopeless V2*

## Important educational disclaimer

**This project is provided strictly for education, research, and experimentation. We do not endorse, encourage, or condone using it to manipulate Discord badges, activity history, account metrics, or any platform feature.**

**This is not an official Discord feature or API. It modifies undocumented in-memory Discord client internals and may violate Discord's Terms of Service. Running it can cause activity removal, badge loss, client instability, warnings, suspension, or permanent account termination. It is not guaranteed to be safe, undetectable, or compatible with any account.**

**Use it only where you accept those risks. The authors and contributors are not responsible for bans, suspensions, lost data, purchases, progress, or any other consequence. By using, modifying, or distributing this project, you accept full responsibility for your actions.**

---

## Auto Quests

Hopeless V2 ships with an **Auto Quest** engine that scans your Discord quest store, runs every accepted quest it can complete, and tracks progress live in an in-client panel. It does **not** use OAuth, does **not** authorize third-party apps on your account, and does **not** attempt any external bypass. Everything runs through the same endpoints the Discord client already uses.

If a quest type can't be completed without manual input, the UI tells you and hands you a **GO TO QUESTS** button so you can finish it yourself.

---

## Supported quest types

| Quest type | Method | Status |
|---|---|---|
| **WATCH_VIDEO** | Posts advancing `timestamp` values to `/quests/{id}/video-progress` | Fully automatic |
| **PLAY_ON_DESKTOP** | Patches `RunningGameStore` with a fake process; Discord's own quest engine detects it | Fully automatic |
| **STREAM_ON_DESKTOP** | Patches `ApplicationStreamingStore.getStreamerActiveStreamMetadata` | Fully automatic |
| **PLAY_ACTIVITY** | Sends heartbeat pings to `/quests/{id}/heartbeat` with a voice channel stream key | Fully automatic |
| **ACHIEVEMENT_IN_ACTIVITY** | Tries the heartbeat path first; falls back to manual completion | **Manual fallback** — you get a **GO TO QUESTS** button |

Games, streams, and activities report their progress through Discord's own heartbeat dispatcher (`QUESTS_SEND_HEARTBEAT_SUCCESS`), so the client sees them the same way it would see a real running game. Video progress is the only path that hits the quest API directly.

---

## Requirements

- **Discord desktop client** (browser is not supported — the script needs `webpackChunkdiscord_app` and `DiscordNative`)
- Either **Vencord** installed, or running the script natively through the developer console (Vencord is strongly recommended — it exposes the Webpack API cleanly)
- At least one quest visible in your Discord Quests tab

---

## Usage

1. Open Discord.
2. Paste the script into the console (or run it through a Vencord plugin wrapper).
3. The quest picker appears in the top-right corner.
4. **Select the quests you want to run** using the checkboxes. All quests are checked by default.
5. Toggle your options (see below).
6. Click **START**.
7. Watch the panel. Each quest shows live progress, and the log at the bottom prints status changes.

Press **Shift + .** at any time to hide or show the panel. Click **STOP** on the panel header to halt everything and clean up.

---

## Options

The options panel (gear icon) exposes:

| Option | Default | Effect |
|---|---|---|
| **Auto-enroll in quests** | On | Enrolls you in quests you haven't accepted yet before running them. Turn off if you want to accept them manually. |
| **Auto-claim rewards** | Off | Claims the reward as soon as a quest completes. Off by default — some claims trigger a captcha that only the Discord UI can solve, so leaving it off is safer. |
| **Sound on completion** | Off | Plays a short chime when a quest finishes. |
| **Random 1–30min delay between cycles** | Off | Adds a random pause between full rescans of the quest list. Useful if you're running many quests and want to look less mechanical. |

Filters at the top of the picker let you narrow the list by **reward type** (in-game, avatar decoration, orbs) and by **quest type** (game, video, activity).

---

## How it works (short version)

- **Module loading** — the script grabs `QuestStore`, `RunningGameStore`, `ApplicationStreamingStore`, `ChannelStore`, `GuildChannelStore`, the Flux `Dispatcher`, and Discord's `RestAPI`. Vencord's `findStore` / `findByProps` / `findByCode` are used when available. A native `webpackChunkdiscord_app` fallback is included for unpatched clients.
- **Game & stream** — a fake game object (with a synthesized PID, exe path, and app id matching the quest) is injected into `RunningGameStore`. Discord sees it as a running process and its quest engine credits progress automatically. The fake is removed the moment the task ends.
- **Activity** — a stream key is generated from a real voice channel id and sent to `/quests/{id}/heartbeat` every ~20 seconds until the target is reached.
- **Video** — the script simulates a playhead by sending advancing `timestamp` values to `/quests/{id}/video-progress` at a realistic cadence (3.5–4.75s between pings, with small jitter).
- **Achievements** — the heartbeat path runs first. If Discord's backend doesn't credit progress, the script gives up gracefully and leaves the quest in the panel with a **GO TO QUESTS** button.

Every task is wrapped in a timeout, a stall detector, and a heartbeat watchdog so a single stuck quest can't hang the whole cycle. Network requests go through a rate-limited queue with 429/5xx retry handling.

---

## What this script does **not** do

- **No OAuth.** It never authorizes any third-party application on your account. The consent modal, `/oauth2/authorize` flow, discordsays proxy, and Vencord helper relay from earlier versions have all been removed.
- **No bypass of achievement verification.** Achievements that require an in-activity check have to be completed manually if the heartbeat path fails.
- **No analytics spoofing.** It does not send fake playtime, fake launches, or fake heartbeat sessions to `/science`.
- **No external servers.** All requests go to Discord's own endpoints, from your own client, with your own session.

---

## Shortcuts

| Key | Action |
|---|---|
| **Shift + .** | Toggle the panel |
| Double-click header | Collapse / expand the panel |
| Drag header | Reposition the panel |

---

## Known limitations

- **Achievement quests** often need manual completion. This is expected. The script tries the heartbeat route first, but Discord's backend frequently rejects synthetic achievement heartbeats.
- **Game quests** depend on your Discord build still routing `RunningGameStore` through the patched methods. If a future Discord update renames them, the patcher will fail to initialize and the script will abort with a clear error.
- **Video quests** are reliable but slow — they take roughly the target duration in real time because the playhead has to advance naturally.
- **Multiple game quests at once** are serialized. Only one fake game can be active at a time, so a second game quest waits until the first completes or fails.
- **Claims that hit a captcha** cannot be auto-solved. Leave Auto-claim off if you want to claim manually with full UI context.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `RunningGameStore patcher could not initialize` | Discord renamed the store methods in a recent update | Wait for a script update or check the module list manually |
| Game quest shows `RUNNING` at 0% forever | Discord didn't emit a heartbeat | The watchdog should fail it after 90 seconds — check the log for `[Heartbeat]` lines |
| Video quest stalls at 0% | The endpoint is rate-limiting | Wait and retry; the retry queue handles 429s automatically |
| Panel doesn't appear | Script already running (`window.orionLock` is `true`) | Run the script again — it re-shows the existing panel |
| Everything stops abruptly | You clicked STOP, or the script hit a fatal error | Check the console — `[Hopeless Fatal]` will print the cause |

---

## Contributing

Pull requests are welcome. If you find a quest type that isn't supported or a heartbeat path that stopped working, open an issue with:
- The quest's task config (`taskConfigV2.tasks` shape)
- The console output while the script was running
- Your Discord build version (`Ctrl+Shift+I` → Console → `DiscordNative.app.getVersion()`)

---

## License

No license is provided. This is provided as-is for educational purposes only. See the disclaimer at the top.
