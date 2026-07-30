# Background Work and Lifecycle

## JobScheduler and delegated work

### Expanded runtime quotas

On Android 16, job runtime quotas also cover:

- jobs in the active standby bucket;
- jobs that start while the app is visible and continue after it becomes invisible;
- jobs running alongside a foreground service.

The behavior affects direct `JobScheduler` work as well as WorkManager and DownloadManager. Use user-initiated data-transfer jobs when their semantics fit, inspect job stop reasons, and call `JobScheduler.getPendingJobReasonsHistory()` to understand why pending jobs have not run.

### Abandoned jobs

If a `JobParameters` instance is collected before `jobFinished()` and the job later times out, Android 16 reports `STOP_REASON_TIMEOUT_ABANDONED`. Repeated abandonment can cause the system to run the app's jobs less often. Keep the parameters reachable through completion and always finish the job correctly.

`setImportantWhileForeground()` no longer raises job importance: it is ignored, and `isImportantWhileForeground()` returns `false`. Do not use it as a quota or scheduling strategy. (`api-36`)

## Background launches and playback

### IntentSender launch privileges

Android 17 extends background-activity-launch hardening to `IntentSender`. Replace legacy `MODE_BACKGROUND_ACTIVITY_START_ALLOWED` use with the narrowest granular mode, such as `MODE_BACKGROUND_ACTIVITY_START_ALLOW_IF_VISIBLE`. Use StrictMode or lint to locate legacy flows.

### Audio lifecycle

On Android 17, invalid background playback and volume calls fail silently, and audio-focus requests return `AUDIOFOCUS_REQUEST_FAILED`. API 37-targeted apps also need a foreground service with while-in-use capability for background audio. The exception is an app that holds exact-alarm permission and is using a `USAGE_ALARM` stream. (`api-37`)

## Alarm and profiling callbacks

### In-process exact idle alarms

Android 17 adds an `OnAlarmListener` overload for `AlarmManager.setExactAndAllowWhileIdle()`. It avoids a `PendingIntent` and the associated long partial wakelock when the exact callback runs in process.

### Profiling triggers

`ProfilingManager` adds triggers for `COLD_START`, `OOM`, and `KILL_EXCESSIVE_CPU_USAGE`. Select triggers that match the lifecycle failure under investigation rather than relying only on manual captures.
