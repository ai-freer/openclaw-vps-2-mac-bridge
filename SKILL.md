---
name: apple-apps
description: |
  Control Apple Notes, Calendar, and Reminders on a remote Mac via node host (remote osascript).
  Use when: (1) creating/reading/searching Apple Notes, (2) creating/reading calendar events in Apple Calendar,
  (3) creating/reading/completing reminders in Apple Reminders, (4) user mentions "Apple Notes", "备忘录",
  "Apple Calendar", "日历", "Apple Reminders", "提醒事项", or asks to save something to Mac local apps.
  NOT for: Feishu calendar/tasks (use feishu-calendar/feishu-task), NOT for: cron-based bot alerts (use cron tool),
  NOT for: project task management (use feishu-task or GitHub Issues).
  Clarify with user if "remind me" means Apple Reminders (syncs to iPhone) or bot alert (message in chat).
---

# Apple Apps (macOS Node)

Execute AppleScript on MacBook via node host. Zero dependencies — no CLI tools needed.

## When to Use vs NOT

| User intent | Use this skill? | Alternative |
|---|---|---|
| "记到备忘录" / "save to Apple Notes" | ✅ | — |
| "加个日历" / Apple Calendar event | ✅ | feishu-calendar for 飞书日历 |
| "加个提醒" / Apple Reminders | ✅ | cron tool for bot alerts |
| "提醒我两小时后..." (ambiguous) | ❓ Ask user | cron if bot alert |
| 飞书任务/日程 | ❌ | feishu-task / feishu-calendar |

## Execution Pattern

```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e '<applescript>'
```

For multi-line scripts, chain multiple `-e` args. Use `exec` tool with the full command.

## Apple Notes

### List notes (recent 20)
```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Notes" to get name of notes 1 thru 20'
```

### List folders
```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Notes" to get name of every folder'
```

### Read note by name
```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Notes" to get plaintext of note "Note Title"'
```

### Create note
```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Notes" to make new note at folder "Notes" with properties {name:"Title", body:"<p>Content here</p>"}'
```

Body uses HTML: `<br>` for line breaks, `<b>` for bold, `<ul><li>` for lists.

### Search notes by keyword
```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Notes" to get name of (notes whose name contains "keyword")'
```

## Apple Calendar

### List today's events
```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'set today to current date
set time of today to 0
set tomorrow to today + 1 * days
tell application "Calendar"
set results to {}
repeat with c in calendars
repeat with e in (events of c whose start date >= today and start date < tomorrow)
set end of results to (summary of e & " | " & (start date of e as string))
end repeat
end repeat
return results
end tell'
```

### Create event
```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Calendar"
tell calendar "Home"
make new event with properties {summary:"Meeting", start date:date "2026-03-15 14:00", end date:date "2026-03-15 15:00", description:"Notes here"}
end tell
end tell'
```

### List calendars
```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Calendar" to get name of every calendar'
```

## Apple Reminders

### List incomplete reminders
```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Reminders" to get name of reminders whose completed is false'
```

### List reminder lists
```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Reminders" to get name of every list'
```

### Create reminder
```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Reminders"
tell list "Reminders"
make new reminder with properties {name:"Buy milk", body:"From the store"}
end tell
end tell'
```

### Create reminder with due date
```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Reminders"
tell list "Reminders"
make new reminder with properties {name:"Submit report", due date:date "2026-03-20 09:00"}
end tell
end tell'
```

### Complete a reminder
```
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Reminders" to set completed of (reminders whose name is "Buy milk") to true'
```

## Tips

- First run of each app may trigger macOS automation permission dialog on MacBook
- AppleScript date parsing is locale-dependent; if date fails, use `current date` + arithmetic
- Notes body uses HTML; Reminders/Calendar use plain text
- All three apps sync with iCloud automatically
