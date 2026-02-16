## Review Comment guidelines
When writing review comments use the following directives to provide more insights and suggest changes accordingly. 

## Instruction Summary

**1. Provide absolute path for Execution Directives**

Always specify full absolute paths for Execution directives(service lifecycle commands) to avoid reliance on environment variables like `$PATH`, which systemd does not inherit.

**Requirements:**
- All Execution directives such as `ExecStart`, `ExecStartPre`, `ExecStartPost`, `ExecStop`, `ExecReload` are required to use absolute paths.
- Relative paths and bare commands are not allowed.

**Example:**
```ini
[Service]
[Service]
ExecStart=/usr/bin/mydaemon --option
ExecStartPre=/usr/bin/mkdir -p /var/run/myapp
ExecStop=/bin/kill -TERM $MAINPID
```

**2. Define appropriate service type**

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

**3. Dependency management requirement**

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

**4. Enabling of automatic restarts as per requirements**

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

**5. Configure Reloads**

- It support reloading configuration without needing the full service to restart.
- It reduces downtime for long running services.
- Configuration changes can be applied with minimal interruption without needing a full restart.

**Requirements**

***Use ExecReload= for long-running services***

- Send proper signal to main process(typically SIGHUP) which can reload the config without stopping
- It should be only used for `Type=notify` or `Type=simple` services
- It should not be used for `Type=oneshot` services

**Example:**
```ini
[Service]
Type=notify
ExecStart=/usr/bin/mydaemon
ExecReload=/bin/kill -HUP $MAINPID
```
