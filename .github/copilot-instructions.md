## Review Comment guidelines
When writing review comments use the following directives to provide more insights and suggest changes accordingly. 

## Instruction Summary

**1. Absolute path requirement for Execution Directives**

Always specify full absolute paths for Execution directives(service lifecycle commands) to avoid reliance on environment variables like `$PATH`, which systemd does not inherit.

## Requirements
- All Execution directives such as `ExecStart`, `ExecStartPre`, `ExecStartPost`, `ExecStop`, `ExecReload` are required to use absolute paths.
- Relative paths and bare commands are not allowed

**Example:**
```ini
[Service]
[Service]
ExecStart=/usr/bin/mydaemon --option
ExecStartPre=/usr/bin/mkdir -p /var/run/myapp
ExecStop=/bin/kill -TERM $MAINPID
```

**2. Define proper service types**

**Prefer `Type=notify` and `Type=oneshot`** over other types like `Type=simple`... 

