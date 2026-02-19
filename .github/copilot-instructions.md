# Review Comment guidelines
When writing review comments use the following directives to provide more insights and suggest changes accordingly. 

# Instruction Summary

## 1. Provide absolute path for Execution Directives

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

## 2. Define appropriate service type

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

***When to use RemainAfterExit= with oneshot***
- `RemainAfterExit=yes` - Task creates lasting state (mounts, directories, configuration). 
- `RemainAfterExit=no` - Task is temporary cleanup with no persistent state.

**Example:**
```ini
[Service]
Type=oneshot
RemainAfterExit=yes
```

## 3. Dependency management requirement

- Define startup order and dependencies so services start only when prerequisites are ready.
- Without dependencies, services start in random order causing race conditions and there might case where services may try to use resources before they're available (network, database, etc.)
- CPC services must never be added as dependencies to publicly available services.
  
**Requirements:**

***Use `After=` for ordering***

- Controls when service starts relative to others
- Example: After=network.target means "start after network is available"

***Use `Requires=` for strict dependencies***

- Use it when a service cannot start if dependency(strict) fails. 
- If dependency crashes/restarts, this service also restarts.
- Use for critical dependencies.

***Use Wants= for optional dependencies***

- Use it when service can start even if dependency(soft/optional) is missing.
- Use it when service does not restart if dependency restarts.
- It must be paired with `After=` or `Before=` for ordering.
- Use for optional features or platform-specific services.

**Example:**
```ini
[Unit]
After=network.target
Requires=db.service
Wants=logger.service
```

## 4. Enabling of automatic restarts as per need

- It controls whether services can automatically restart after it crashes.
- Defaultly services are not restarted automatically, it stays dead after a crash(`Restart=no`)

**Requirements**

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

## 5. Configure Reloads

Enable configuration updates without stopping the service to avoid disrupting active operations.

**Requirements**

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

**Requirements**

- Use direct commands in `ExecStart=` instead of scripts
- Avoid shell script wrappers for start/stop logic
- Do not use `PIDFile=` unless service forks non-standardly
- it allows systemd to automatically track the main process

**Example:**
```ini
# Instead of a script, use direct ExecStart in the unit file
[Service]
ExecStart=/usr/bin/myapp
```

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

**Requirements**

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

**Requirements**

***Always specify both:***

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

**Requirements**

***Must include `[Install]` section:***

- Add `WantedBy=multi-user.target` for systemd services to ensure auto-start at boot.
- It enables the service to start at boot.

**Example:**
```ini
[Install]
WantedBy=multi-user.target
```

## 10. Add proper description

- Adding proper description helps to understand what the service is about. 

**Example:**
```ini
[Unit]
Description=My service
```
