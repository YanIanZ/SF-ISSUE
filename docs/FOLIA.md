# Folia support

Slimefun4 schedules exclusively through SuDo's `SchedulerService` (Folia-aware), obtained from the
`SuDo` context on the `Slimefun` singleton: `Slimefun.instance().scheduler()`. There are no direct
`Bukkit.getScheduler()` / `BukkitRunnable` calls in the scheduling path. Automated tests run on
Paper via MockBukkit (Folia cannot be emulated).

Before release, smoke-test on a real **Folia 26.2** server with Slimefun installed:

1. Boot Folia 26.2 with Slimefun — confirm it enables without error.
2. Place and run a machine (e.g. an Electric Furnace) — verify it ticks and processes items.
3. Trigger auto-save (or wait for the `AutoSavingService` interval) — verify no
   `IllegalStateException` about wrong-region-thread / asynchronous access.
4. Open the Slimefun guide search (chat input) — verify the callback fires and the search runs.

**Pass criteria:** no wrong-region-thread / "cannot be run asynchronously" errors; machines,
auto-save, and chat input all function.
