---
id: E01
epic: E — Native notifications
title: P0 NextAction desktop notification
size: M
requires: [B10]
novel: false
---

## What
When a P0 (top-priority) task's NextAction time arrives, the app fires a native
macOS desktop notification. The notifier watches scheduled NextAction times and
raises an OS-level alert at the moment a P0 task is due back on the radar.

## Why it exists
The author runs many tasks and cannot watch the board continuously; a P0 task
coming due is exactly the kind of thing that must reach him even when the app is
in the background. Native notifications are the push channel that surfaces it.

> In the author's words: *"Notify me via my TaskApp."* And, on the mechanism
> working at all: *"I also don't have any app notification for some reason — can
> you check why?"*

## Acceptance criteria (EARS)
- When a P0 task's NextAction time (B10) is reached, the system shall post a
  native desktop notification for that task.
- The system shall raise the notification only for P0 tasks, not lower
  priorities.
- The notification shall identify the task it concerns.
- The system shall schedule the alert from the task's NextAction timestamp.

## Build notes
- Implemented in `notifier.ts` in the Electron main process, driven by the
  NextAction time model (B10).
- Notification delivery depends on a stable, signed app identity so the OS keeps
  the app's notification permission across rebuilds — after the app was renamed
  to "taskman," permission had to be re-granted; sign the dev bundle with a stable
  identity to avoid this recurring.
- Auto-clearing these notifications and handling far-future schedules are separate
  concerns — see E02 and E03.
