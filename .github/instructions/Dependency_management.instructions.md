The following information provide more insights towards how a dependency could be framed. And How much it is important to use the right directive on systemd services.

## Dependency management Guidenlines

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

The following are some scenario's which defines how a configuration works based on the above guidelines, which can be essential while providing review comments.

### Part 1: Startup Behaviour (Dependent Service Fails or Succeeds)


|Combination in A|	Service B's Startup Outcome|	Service A's Startup Outcome|	Explanation|
|---|---|---|---|
|Wants=B|	B Succeeds|	A Starts|	B starts first (implied ordering), A follows immediately after B reaches its `ready state`<sup>1</sup>.|
|Wants=B|	B Fails|	A Starts|	Weak Dependency: A's startup proceeds despite B's failure. There is no implicit ordering wait, but B's failure is finalized before A is `fully active`<sup>1</sup>.|
|Requires=B|	B Succeeds|	A Starts|	Strong Dependency: B starts, and since it succeeds, A is allowed to start immediately.|
|Requires=B|	B Fails|	A Fails|	Strong Dependency: The core requirement is unmet, so A's startup is immediately aborted.|
|Wants=B, After=B|	B Succeeds|	A Starts|	Standard Robust Setup: A waits for B to be `fully ready`<sup>1</sup> (After), then starts (permitted by Wants).|
|Wants=B, After=B|	B Fails|	A Starts|	Weak Dependency & Ordering Met: B's failure satisfies the After ordering constraint. The weak Wants allows A to proceed despite the failure.|
|Requires=B, After=B|	B Succeeds|	A Starts|	Standard Critical Setup: A waits for B to be `fully ready`<sup>1</sup> (After), then starts (permitted by Requires).|
|Requires=B, After=B|	B Fails|	A Fails|	Strong Dependency: A waits for B to finalise (After), sees the failure, and aborts its own startup due to the unmet Requires constraint.|

<sup>1</sup>Refer Part 3

### Part 2: Runtime Behaviour (Crashes and Restarts)

- This scenario occurs after both services have successfully started and are running. 
- The behaviour here is primarily governed by the strength of the dependency (Wants vs. Requires) and the "Restart="<sup>2</sup> setting in the service files.

|Combination in A|	Service B's Runtime Outcome|	Service A's State|	Explanation|
|---|---|---|---|
|Wants=B|	B Crashes and Restarts|	A Unaffected|	Weak Dependency: A is not monitoring B's ongoing status. A continues to run while B restarts.|
|Wants=B| B Crashes and Stops|	A Unaffected|	Weak Dependency: A continues to run.|
|Requires=B|	B Crashes and Restarts|	A Stops (and restarts, if configured)|	Strong Dependency: When B crashes, systemd recognizes the strong dependency and stops A. If A has `Restart=`<sup>2</sup> configured, it will restart immediately after being stopped.|
|Requires=B|	B Crashes and Stops|	A Stops|	Strong Dependency: A stops because its required service is `no longer active`<sup>1</sup>. A remains stopped unless an external trigger restarts it.|

<sup>1</sup>Refer Part 3

<sup>2</sup>Refer Part 4


## Part 3: Fully Ready Vs Fully Active

In the context of systemd:

|Term|	Meaning (Operational State)|	Implication for Dependent Services (After=)|
|---|---|---|
|Fully Active|	The service's main process has successfully been launched and is running. Systemd has confirmed that the initial startup command succeeded.|	Systemd knows the process is running, but it does not guarantee that the application is ready to handle requests (e.g., ports might still be binding, configuration loading).|
|Fully Ready|	The service is running and has completed all its internal initialization tasks (e.g., loaded config, connected to database, opened sockets, accepted connections).|	The service is guaranteed to be ready to perform its function, and any service waiting with an After= constraint can safely start using it.|

**Fully Active (Systemd State)**

- "Fully Active" refers to the state reported by systemd itself, typically seen as Active: `active (running)` in the output of systemctl status <unit>.
- ***Fully Active state*** is reached as soon as the service's primary execution command (usually specified by `ExecStart=`) returns successfully, and the process is established.
- Systemd only manages the process lifecycle. When the process starts, the unit is active. By default Systemd is blind to the internal initialization logic of the application.

**Fully Ready (Application State)**

- "Fully Ready" refers to the state where the application running inside the process is capable of doing its job.
- **Fully Ready state** is reached only after the application finishes internal tasks (e.g., A plugin completed all its initialisation sequence and it is ready to service the clients).
- Systemd usually relies on Type=notify units to know when a service is truly ready. With Type=notify, the service actively sends a signal back to systemd via the sd_notify() function (or an equivalent) once it's done initialising. Only then does systemd consider the unit's startup sequence complete.
