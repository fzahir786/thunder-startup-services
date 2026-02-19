### Review Comment Linking Guidelines

When writing review comments based on custom instructions located in .github/instructions/**.instructions.md, include a direct GitHub link to the exact violated guideline in the respective instruction file. Use the following format:

    Refer: https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/copilot-instructions.md#guideline-section-name

## Examples

    Refer: https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/copilot-instructions.md#1-add-proper-service-description 
    Refer: https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/instructions/Dependency_management.instructions.md#dependency-management-guidenlines
    

# Instruction Summary
  1. [Add proper service description](https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/copilot-instructions.md#1-add-proper-service-description)
  2. [Provide absolute path for Execution Directives](https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/copilot-instructions.md#2-provide-absolute-path-for-execution-directives)
  3. [Define appropriate service type](https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/copilot-instructions.md#3-define-appropriate-service-type)
  4. [Service state management(RemainAfterExit)](https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/copilot-instructions.md#4-service-state-managementremainafterexit)
  5. [Configure Reloads](https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/copilot-instructions.md#5-configure-reloads)
  6. [Avoid usage of custom script](https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/copilot-instructions.md#6-avoid-usage-of-custom-script)
  7. [Usage of Drop-in files](https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/copilot-instructions.md#7-usage-of-drop-in-files)
  8. [Set Timeouts and Limits Appropriately](https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/copilot-instructions.md#8-set-timeouts-and-limits-appropriately)
  9. [Boot Integration Requirement](https://github.com/fzahir786/thunder-startup-services/blob/develop/.github/copilot-instructions.md#9-boot-integration-requirement)

## 1. Add proper service description

Always provide a clear, descriptive Description= in the `[Unit]` section to explain what the service does.

**Requirements:**

- It should be Clear and specific about the service's purpose
- Helpful for others to understand what the service is about.
- Try to avoid generic descriptions, repeating filenames or missing descriptions.
  
**Example:**
```ini
#Example 1
[Unit]
Description=Wi-Fi Configuration Manager for Embedded Devices
```

**Incorrect example:**
```ini
#Incorrect example 1
[Unit]
Description=My service  # "My service" tells nothing about what it does

#incorrect example 2
[Unit]
Description=webapp.service  # Just repeating filename
```

## 2. Provide absolute path for Execution Directives

Always specify full absolute paths for Execution directives(service lifecycle commands) to avoid reliance on environment variables like `$PATH`, which systemd does not inherit.

**Requirements:**
- All Execution directives such as `ExecStart`, `ExecStartPre`, `ExecStartPost`, `ExecStop`, `ExecReload` are required to use absolute paths.
- Relative paths and bare commands are not allowed.

**Example:**
```ini
[Service]
ExecStart=/usr/bin/mydaemon --option
ExecStartPre=/usr/bin/mkdir -p /var/run/myapp
ExecStop=/bin/kill -TERM $MAINPID
```

**Incorrect example:**
```ini
[Service]
ExecStart=mydaemon --option          # Missing path to the executable
ExecStartPre=mkdir -p /var/run/myapp  # Relative command
ExecStop=./stop.sh                    # Relative path to the directory
```

## 3. Define appropriate service type

- Prefer `Type=notify` and `Type=oneshot` over other types like `Type=simple`. 
- `Type=simple` doesn't guarantee service readiness, systemd just assumes it. And it causes race conditions where dependent services start before the service is truly operational.

**Requirements:**

***For long running services:***
- Use `Type=notify`.
- Service must send readiness signal to systemd using sd_notify().
- Systemd waits for "I'm ready" signal before marking service as started which prevents dependent services from starting too early.

***For one-time tasks***
- Use `Type=oneshot`.
- Service runs once and exits.
- Systemd waits for command to complete.

**Example:**
```ini
[Service]
Type=oneshot
RemainAfterExit=yes
```

## 4. Service state management(RemainAfterExit)

- Control whether systemd considers a service "active" after its process exits.
- By default, when a service process exits, systemd marks it as "inactive".
- But some services create persistent state (mounted filesystems, created directories, system configuration).
- Other services depend on knowing if that state still exists.
- RemainAfterExit= tells systemd whether the service's "effect" persists after the process exits.

**Requirements:**

***Use with `Type=oneshot` only***

- `RemainAfterExit= `is only valid with `Type=oneshot`
- Not applicable to `Type=notify` or `Type=simple` (they have running processes)
- Attempting to use with other types is an error

***When to use `RemainAfterExit=yes`***

- Service creates persistent state that outlives the process
- Other services depend on this state
- Services that should appear "active" even after the command completes

***When to use `RemainAfterExit=no`***

- Service that performs temporary operation with no lasting state
- Cleanup tasks and one-time operations can use `RemainAfterExit=no`
- Service that are required to be "inactive" after completing

**Example:**
```ini
[Service]
Type=oneshot
RemainAfterExit=yes
# Mount creates persistent state - filesystem remains mounted
ExecStart=/bin/mount /dev/sdb1 /mnt/data
ExecStop=/bin/umount /mnt/data
```

**Incorrect example:**
```ini
#incorrect example 2
[Service]
Type=oneshot
RemainAfterExit=no  # This is wrong - Mount persists but service marked inactive
ExecStart=/bin/mount /dev/sdb1 /mnt/data
ExecStop=/bin/umount /mnt/data

#incorrect example 3
[Service]
Type=oneshot
# Missing RemainAfterExit - defaults to 'no' which is wrong here
ExecStart=/bin/mkdir -p /var/run/myapp
```

## 5. Configure Reloads

Enable configuration updates without stopping the service to avoid disrupting active operations.

**Requirements:**

***Without reload capability***

- Configuration changes require full service restart.
- Active connections/sessions are dropped.
- Users experience service interruption (buffering, disconnections).

***With reload capability***

- Configuration updates applied while service keeps running.
- Active users remain connected.
- Zero downtime for configuration changes.
- Better user experience in embedded environments.

***For long-running daemon services***

- Add `ExecReload=` directive to enable graceful configuration updates.
- Send signal to main process (typically SIGHUP) that tells service to re-read its configuration.
- Service must support signal-based reload.

***Service type restrictions***

- Use with` Type=notify` - long-running daemons with readiness notification.
- Use with `Type=simple` - long-running daemons without notification.
- Do not use with `Type=oneshot` - no running process to signal after task completes.

***Common reload signals***

- `SIGHUP` (signal 1) - Standard "reload configuration" signal.
- `SIGUSR1` (signal 10) - Custom application-defined reload.
- `SIGUSR2` (signal 12) - Alternate custom reload operation.
- Do not use signals like `SIGTERM` (Terminates the service) and `SIGKILL` (Force kills service immediately).

**Example:**
```ini
[Service]
Type=notify
ExecStart=/usr/bin/nginx -g 'daemon off;'
ExecReload=/bin/kill -HUP $MAINPID
# nginx reloads config on SIGHUP without dropping connections
```

**Incorrect example:**
```ini
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/initialize-system.sh
ExecReload=/bin/kill -HUP $MAINPID  #No process is running to receive the signal
```

## 6. Avoid usage of custom script

- Let systemd manage service lifecycle directly instead of using wrapper scripts.
- Scripts add complexity and failure points and increases boot time
- Systemd can't track processes properly through scripts making it harder to debug and maintain

**Requirements:**

- Use direct commands in `ExecStart=` instead of scripts
- Avoid shell script wrappers for start/stop logic
- Do not use `PIDFile=` unless service forks non-standardly
- it allows systemd to automatically track the main process

**Incorrect example:**
```ini
[Service]
Type=simple
ExecStart=/usr/local/bin/start-myapp.sh  #Wrapper script

#content of wrapper script
#!/bin/bash
# Unnecessary wrapper doing simple tasks
cd /opt/myapp
export APP_CONFIG=/etc/myapp.conf
exec /usr/bin/myapp --config $APP_CONFIG
```

**Corrected example:**
```ini
#instead of having it in wrapper script we can have the config in systemd service file directly 
[Service]
Type=notify
WorkingDirectory=/opt/myapp
Environment="APP_CONFIG=/etc/myapp.conf"
ExecStart=/usr/bin/myapp --config /etc/myapp.conf
```

## 7. Usage of Drop-in files 

- Use drop-in files for overrides, which preserves original packaged unit files and avoids editing `/usr/lib/systemd/system/` files.
- Per-device customization without modifying base configuration.
- Easier to track changes and updates.

**Requirements:**

***For RDK Components***

- Do not use drop-in file  files, add content to unit directly.

***For OSS Components***

- OSS components may use drop-in files in `/etc/systemd/system/unit.d/` 
- it is good for per-device customization

**Example:**
```ini
# Create /etc/systemd/system/my.service.d/override.conf
[Service]
Restart=always
ExecStartPre= <Add my per device change>
```

## 8. Set Timeouts and Limits Appropriately

- Define how long systemd waits for service start/stop operations.
- If timeout is not specified, systemd uses default timeout value (90 secs)
- Hung services block boot or shutdown
- Clear expectations on timeout can prevent indefinite waits

**Requirements:**

***Always specify both***

- `TimeoutStartSec=` - How long to wait for service startup (it should be 30 secs or less).
- `TimeoutStopSec=` - How long to wait for graceful shutdown (it should be 10 secs or less).
- If timeout value is greater than 90 seconds then proper justification is required.

**Example:**
```ini
[Service]
TimeoutStartSec=30
TimeoutStopSec=10
```

## 9. Boot Integration Requirement

- Link services to targets like `multi-user.target` for auto-start at boot.
- It is essential because services won't start automatically without `[Install]` section
- It is required to ensure that proper target linkage happens at boot phase

**Requirements:**

***Must include `[Install]` section***

- Add `WantedBy=multi-user.target` for systemd services to ensure auto-start at boot.
- It enables services to start automatically during bootup.
- It can be used for standalone services(not a dependency).

**Example:**
```ini
[Install]
WantedBy=multi-user.target
```
