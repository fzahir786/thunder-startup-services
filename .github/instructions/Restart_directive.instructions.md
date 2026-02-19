---
applyTo: "systemd/system/*.service"
---

### Review Comment Linking Guidelines

When writing review comments based on custom instructions located in .github/instructions/**.instructions.md, include a direct GitHub link to the exact violated guideline in the respective instruction file. Use the following format:

    Refer: https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/instructions/<instruction-file>.instructions.md#guideline-section-name

**Correct example:**
    
    Refer: https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/instructions/Restart_directive.instructions.md#enabling-of-automatic-restarts-as-per-need
    Refer: https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/instructions/Restart_directive.instructions.md#the-restart-directive
    
The following provides details on Restart directive. More information on what restart directive can be used depending on the service requirement.

## Enabling of automatic restarts as per need

- It controls whether services can automatically restart after it crashes.
- Defaultly services are not restarted automatically, it stays dead after a crash(`Restart=no`)

**Requirements:**

***For critical services***

- Do not use `Restart=` (keep default `Restart=no`)
- Crashes indicate state corruption - device should reboot

***For non-critical services***

- Use `Restart=on-failure` to recover from crashes
- Always pair with `RestartSec=` (minimum 1 second)
- Prevents rapid restart loops that exhaust resources

**Example:**
```ini
[Service]
Restart=on-failure
RestartSec=5
```

### The Restart directive

- The `Restart=` directive controls the conditions under which systemd should automatically attempt to restart the service process after it exits.
- Here are all the possible values for the Restart= directive and the conditions that trigger an automatic restart:

|Value| Description|	Triggers Restart On...|
|---|---|---|
|no|	(Default) The service will never be restarted automatically.|	None|
|always|	The service will always be restarted, regardless of the exit status, signal, or reason it stopped.|	Any type of exit (success, failure, signal, or crash).|
|on-success|	The service will restart only if it exits cleanly with a success status.|	Exit code 0 (zero), or one of the clean signals: SIGHUP, SIGINT, SIGTERM, or SIGPIPE.|
|on-failure|	The service will restart if it exits with an error status or is terminated abnormally.|	Exit code non-zero, termination by a signal (excluding the clean ones), or a configuration/operation timeout. (Most common choice for typical daemons)|
|on-abnormal|	The service will restart only if it exits due to a signal (i.e., a crash or external kill signal), not a clean exit or a non-zero exit code.|	Termination by an unclean signal (e.g., SIGSEGV for a crash, or SIGKILL).|
|on-abort|	The service will restart only if it exits due to an uncaught, unclean signal that was not specified as a clean exit signal. This is a subtle variation of on-abnormal.|	Termination by a signal that represents a failure, typically resulting in a core dump.|
|on-watchdog|	The service will restart only if its watchdog timer expires. This requires the application to implement and communicate with systemd's watchdog feature.|	The application failed to send its "keep-alive" signal to systemd within the specified time (WatchdogSec=).|
