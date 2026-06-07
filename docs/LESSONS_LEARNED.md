# Lessons Learned

## Issue: GitHub Actions APK build failed during `npm ci`

### Date/Time

2026-06-07 21:11 +01:00

### Symptom Observed

GitHub Actions stopped in the `Install JavaScript dependencies` step before Gradle started.

### Initial Hypothesis

The failure was caused by an incorrect Node.js version or a GitHub Actions runtime mismatch.

### Actual Root Cause

`patch-package` failed while parsing `patches/react-native-webview+13.16.0.patch`.

### Evidence Used

The run log showed:

- `Failed to apply patch for package react-native-webview`
- `patch file ... could not be parsed`
- `npm error command sh -c patch-package`

The failure happened before any Android build task ran.

### Fix Applied

Regenerated `patches/react-native-webview+13.16.0.patch` from the actual `react-native-webview` source edits so `patch-package` could apply it cleanly.

### Why Previous Assumptions Were Incorrect

The first fatal error was not about Node.js, Java, Gradle, or Android SDK setup. Those were downstream concerns and never executed.

### How to Detect It Faster in the Future

Check the first fatal `ERROR` in the log and confirm whether the build failed before Gradle started. If `npm ci` fails, inspect `postinstall` scripts and patch files first.

### Related Files

[`patches/react-native-webview+13.16.0.patch`](../patches/react-native-webview+13.16.0.patch)
[`package.json`](../package.json)
[`gh workflow run build-apk.yml`](../.github/workflows/build-apk.yml)
[`gh run 27102230244`](https://github.com/EngineerFil206/freekiosk/actions/runs/27102230244)
[`gh run 27102203947`](https://github.com/EngineerFil206/freekiosk/actions/runs/27102203947)

## Issue: Gradle daemon exited with JVM garbage collection thrashing

### Date/Time

2026-06-07 21:11 +01:00

### Symptom Observed

The APK workflow progressed past dependency install, then `./gradlew assembleDebug` failed after several minutes with a daemon memory error.

### Initial Hypothesis

The Android SDK package selection or JDK version was incomplete.

### Actual Root Cause

Gradle was running with the default low heap because the repository did not contain a real `android/gradle.properties` file in CI. The project only had `android/gradle.properties.template`, and `.gitignore` excluded the actual file.

### Evidence Used

The build log reported:

- `The currently configured max heap space is '512 MiB'`
- `Gradle build daemon has been stopped: since the JVM garbage collector is thrashing`

The repo also had:

- `android/gradle.properties.template`
- `.gitignore` entries for `android/gradle.properties`

### Fix Applied

Updated the workflow to generate `android/gradle.properties` during CI with explicit Gradle memory settings and disabled unnecessary parallel/configure-on-demand behavior.

### Why Previous Assumptions Were Incorrect

The build was not failing because of missing SDK packages or the wrong JDK. It was failing because Gradle could not keep enough heap under the default configuration.

### How to Detect It Faster in the Future

Inspect Gradle logs for heap and daemon warnings first. If you see JVM thrashing, check `org.gradle.jvmargs` and whether CI is actually creating the repo-local `gradle.properties` file.

### Related Files

[`android/gradle.properties.template`](../android/gradle.properties.template)
[`android/build.gradle`](../android/build.gradle)
[`android/app/build.gradle`](../android/app/build.gradle)
[`android/gradle.properties`](../android/gradle.properties)
[`.github/workflows/build-apk.yml`](../.github/workflows/build-apk.yml)
[`gh run 27102865267`](https://github.com/EngineerFil206/freekiosk/actions/runs/27102865267)
[`gh run 27103002310`](https://github.com/EngineerFil206/freekiosk/actions/runs/27103002310)

## Build Summary: APK workflow succeeded after two failures

### Date/Time

2026-06-07 21:11 +01:00

### What Actually Prevented Success

Two separate blockers prevented the workflow from succeeding:

1. A malformed `react-native-webview` patch broke `npm ci` during `postinstall`.
2. Gradle memory defaults caused the daemon to thrash and exit during `assembleDebug`.

### Which Hypotheses Were Wrong

The early assumptions that Node.js, Java, or Android SDK versions were the main problem were wrong. Those components were not the first fatal issue.

### Which Debugging Steps Were Useful

- Reading the first failing step in each run.
- Pulling the failed job logs with `gh run view --log-failed`.
- Verifying the patch file with `patch.exe --dry-run`.
- Re-running `npm ci` locally to prove the patch fix before touching Gradle.

### Which Debugging Steps Wasted Time

- Treating deprecation warnings as if they were the cause.
- Looking at Gradle before the patch parser failure was resolved.

### What Future Developers Should Check First

1. The first fatal log line, not the last warning.
2. `postinstall` hooks and patch files before Android toolchain versions.
3. Whether CI is generating the same `android/gradle.properties` behavior that local builds rely on.

### Related Files, Commits, Workflows, or Logs

[`patches/react-native-webview+13.16.0.patch`](../patches/react-native-webview+13.16.0.patch)
[`package.json`](../package.json)
[`android/gradle.properties.template`](../android/gradle.properties.template)
[`android/build.gradle`](../android/build.gradle)
[`android/app/build.gradle`](../android/app/build.gradle)
[`docs/LESSONS_LEARNED.md`](./LESSONS_LEARNED.md)
[`0e5ed02`](https://github.com/EngineerFil206/freekiosk/commit/0e5ed02)
[`25a52cc`](https://github.com/EngineerFil206/freekiosk/commit/25a52cc)
[`gh run 27103002310`](https://github.com/EngineerFil206/freekiosk/actions/runs/27103002310)
