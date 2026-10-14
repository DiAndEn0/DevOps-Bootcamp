# **Up:** [[4. Automation]]
---

# Scripting Languages

In DevOps, we use different scripting languages depending on what we need to automate:

- **Bash:** Best for native Linux operations, running system utilities, piping commands (`grep`, `awk`, `sed`), and quick glue code. It requires no extra runtime on Linux, but handling complex data like JSON is clunky and error-prone.
- **Python:** The best choice when dealing with REST APIs, parsing JSON or YAML, and writing more complex logic. With libraries like `requests` and `rich`, it makes querying backends and formatting reports very straightforward.
- **PowerShell:** Great for Windows and Azure environments. Unlike Bash, it passes structured objects through the pipeline rather than raw text.

---

# Cron & Task Scheduling

Linux uses the **cron** daemon (`crond`) to run scripts and commands automatically in the background on a schedule.

Crontabs follow a standard 5-field syntax:
```text
* * * * * <command-to-run>
┬ ┬ ┬ ┬ ┬
│ │ │ │ └──── Day of week (0-6, Sunday=0)
│ │ │ └────── Month (1-12)
│ │ └──────── Day of month (1-31)
│ └────────── Hour (0-23)
└──────────── Minute (0-59)
```

### Common Examples:
- `*/5 * * * *` — Runs every 5 minutes.
- `0 15 * * 0` — Runs every Sunday at 3:00 PM.
- `0 0 * * *` — Runs daily at midnight.

### A Few Important Cron Gotchas:
1. **Limited Environment:** Cron doesn't load your personal `.bashrc` or shell profile. Always use absolute paths for both commands and scripts (e.g. `/usr/bin/python3` instead of just `python3`).
2. **Current Directory:** Cron defaults to the user's home directory, so relative file paths in scripts can break unless you use absolute paths or `cd` first.
3. **Logging Output:** By default, cron sends unhandled output to local mail. Always redirect output and errors to a log file:
   ```bash
   */5 * * * * /usr/bin/python3 /path/to/script.py >> /var/log/my_script.log 2>&1
   ```

For a deeper dive into cron syntax, see this [Cron Guide](https://www.geeksforgeeks.org/linux-unix/how-to-automate-tasks-with-cron-jobs-in-linux/).

---

# Prometheus & Loki HTTP APIs

Both Prometheus and Loki expose HTTP APIs, which lets us query telemetry data programmatically instead of only viewing it in Grafana.

### Prometheus API
- Endpoint: `/api/v1/query?query=<PromQL>`
- Returns a JSON object with a `result` array. Each item has:
  - `metric`: Labels for that series (like status code, instance name, job).
  - `value`: A list containing `[timestamp, "string_value"]`. We convert `"string_value"` to float in Python.
- Reference: [Prometheus Query API Documentation](https://prometheus.io/docs/prometheus/latest/querying/api/)

### Loki API
- Endpoint: `/loki/api/v1/query_range?query=<LogQL>`
- Used to query log streams within a time window.
- In multi-tenant setups, it requires the `X-Scope-OrgID` header.
- The returned data contains `streams` with lists of `[timestamp_ns, "log line"]`.
- Reference: [Loki HTTP API Documentation](https://grafana.com/docs/loki/latest/reference/api/)
