# **Up:** [[4. Automation]]
---

In this part of the bootcamp, I focused on automating cluster health checks and host monitoring. Rather than manually opening Grafana dashboards or running ad-hoc terminal commands, I developed two dedicated automation scripts:

1. **Python Cluster Health Checker:** Connects directly to Prometheus and Loki HTTP APIs, executes PromQL and LogQL queries, parses the metrics and log streams, evaluates thresholds, and generates both color-coded terminal status output and a standalone HTML report.
2. **Bash VM Weekly Resource Monitor:** Collects native OS-level CPU (`sar`), memory (`free`), and disk (`df`) statistics, manages automated weekly log rotation, and appends formatted reports to a log file.

Both scripts are deployed to the host OS of the K3s virtual machine and automated using the Linux `cron` daemon.

---

## Exposing Metrics and Reconfiguring Prometheus

Before I could write the Python script, I needed to ensure that Prometheus was actually collecting and exposing the exact metrics I wanted to query. 

In the previous monitoring lab, I had instrumented my Spring Boot microservices (`simple-web-app` and `simple-app-sender`) with Micrometer and OpenTelemetry. However, querying comprehensive cluster and host health required more than just application metrics. I had to reconfigure what Prometheus scrapes and exposes:

1. **Host & Node Metrics:**
   To calculate total CPU usage, I/O wait, and memory consumption, I needed metrics from `node-exporter` (such as `node_cpu_seconds_total`, `node_memory_MemAvailable_bytes`, and `node_memory_MemTotal_bytes`). I ensured `node-exporter` was running on the host and that Prometheus included a dedicated scrape job for it.
2. **Application HTTP Request Metrics:**
   To calculate error rates, I needed `http_server_requests_seconds_count` labeled with HTTP `status` codes (e.g., `200`, `400`, `404`, `500`). I verified that Micrometer was exporting these counters properly through the OpenTelemetry collector.
3. **Resident Memory Tracking:**
   To detect potential memory leaks across services over time, Prometheus had to scrape `process_resident_memory_bytes` under the `kubernetes-services` job.
4. **Exposing APIs to the Host OS:**
   Because the automation script runs on the K3s host VM (outside of the Kubernetes pod network), it cannot access internal cluster DNS names like `prometheus.bootcamp-v1.svc.cluster.local`. I exposed the services on dedicated NodePorts:
   - **Prometheus API:** Accessible at `http://localhost:30090`
   - **Loki API:** Accessible at `http://localhost:31100`

Once the metrics were exposed and targets confirmed healthy in the Prometheus targets UI, I learned how to structure PromQL queries, execute them over HTTP endpoints, and parse the resulting JSON payloads.

---

## Phase 2: Python Health Checker

### The Python Script

Here is the complete script (`cluster_health_check.py`):

```python
import requests
import json
import datetime
from rich.console import Console



# 1. Supply multi-tenant header
headers = {
    "X-Scope-OrgID": "your-tenant-id"
}

# 2. Filter server-side in LogQL; let requests handle URL encoding
params = {
    "query": '{service_name="simple-app-sender"} |~ "(?i)ERROR|FAIL"',
    "limit": 100
}


r = requests.get("http://localhost:30090/api/v1/query?query=rate(http_server_requests_seconds_count[5m])")
dump = r.json()
requests_dict = dump['data']['result']
errors_count = 0
requests_count = 0 

for request in requests_dict:
    if request['metric']['status'].startswith('4') or request['metric']['status'].startswith('5'):
        errors_count+=1
    requests_count+=1

error_rate = 0 if requests_count == 0 else errors_count / requests_count 

# --------------------------------------------- #

r = requests.get("http://localhost:30090/api/v1/query?query=100*(1-avg by(instance) (rate(node_cpu_seconds_total{mode='idle'}[5m])))")
dump = r.json()
requests_dict = dump['data']['result']

node_cpu_total = 0

for instance in requests_dict:
    node_cpu_total+= float(instance['value'][1])

# --------------------------------------------- #

r = requests.get("http://localhost:30090/api/v1/query?query=100 * avg by(instance) (rate(node_cpu_seconds_total{mode='iowait'}[5m]))")
dump = r.json()
requests_dict = dump['data']['result']

iowait = 0
for instance in requests_dict:
    iowait += float(instance['value'][1])

# --------------------------------------------- #

r = requests.get("http://localhost:30090/api/v1/query?query=100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))")
dump = r.json()
requests_dict = dump['data']['result']

memory_usage = 0

for instance in requests_dict:
    memory_usage += float(instance['value'][1])

# --------------------------------------------- #

r = requests.get("http://localhost:30090/api/v1/query?query=delta(process_resident_memory_bytes{job='kubernetes-services'}[24h]) > 0")
dump = r.json()
requests_dict = dump['data']['result']

potential_leaks = []

for instance in requests_dict:
    potential_leaks.append(instance)

# ------------------------------- #

url = "http://localhost:31100/loki/api/v1/query_range"


headers = {
    
}

params = {
    "query": '{service_name=~"simple-app-sender|simple-web-app"} |~ "(?i)ERROR|FAIL"',
    "limit": 100
}
problems = []

try:
    response = requests.get(url, headers=headers, params=params, timeout=10)
    response.raise_for_status() 
    
    data = response.json().get("data", {}).get("result", [])
    
    problems = [
        line 
        for stream in data 
        for _, line in stream.get("values", [])
    ]
except requests.exceptions.RequestException as e:
    print(f"Loki request failed: {e}")  
    
# -------------------------------------------- #
#
#


console = Console(record=True)


HEALTHY = "[bold green][HEALTHY][/bold green]"
UNHEALTHY = "[bold red][NOT HEALTHY][/bold red]"
WARNING = "[bold yellow][Warning][/bold yellow]"

console.print(f"Prometheus + Loki Health Check -> date: {datetime.datetime.now()}")
console.print("Processing...")
console.rule()

console.print(
    f"Error Rate of HTTP requests for the last 5 minutes: {error_rate * 100:.2f}% "
    f"{HEALTHY if error_rate < 0.05 else UNHEALTHY}"
)
console.print(
    f"Total CPU Usage: {node_cpu_total}% "
    f"{HEALTHY if node_cpu_total < 95 else UNHEALTHY}"
)
console.print(
    f"IOWAIT percent: {iowait}% "
    f"{HEALTHY if iowait < 5 else UNHEALTHY}"
)
console.print(
    f"Memory Usage(percent): {memory_usage}% "
    f"{HEALTHY if memory_usage < 90 else UNHEALTHY}" 
)

console.print("\n[bold]Potential Leaks:[/bold]")
for leak in potential_leaks:
    service = leak["metric"]["service"]
    mem_mb = float(leak["value"][1]) / 1000
    console.print(f"  {service} Memory up: {mem_mb:.2f} MB")
console.print(HEALTHY if len(potential_leaks) == 0 else WARNING)

console.print("\n[bold]Loki Logs:[/bold]")
console.print(f"Found {len(problems)} matching logs:")
for entry in problems:
    console.print(f"  {entry}")
console.print(HEALTHY if len(problems) == 0 else WARNING)


console.save_html("cluster_metrics.html")
```

---

### Deep Dive: How the Script Works

The script queries Prometheus and Loki REST APIs, parses the responses, and formats a report.

#### 1. HTTP Request Error Rate
- **PromQL Query:** `rate(http_server_requests_seconds_count[5m])`
- **Mechanism:**
  - This query retrieves the per-second rate of HTTP requests per endpoint and status over a 5-minute sliding window.
  - The script iterates through the resulting metric series:
    ```python
    for request in requests_dict:
        if request['metric']['status'].startswith('4') or request['metric']['status'].startswith('5'):
            errors_count += 1
        requests_count += 1
    ```
  - It checks the `status` label of each series. If it begins with `'4'` (client error) or `'5'` (server error), it increments `errors_count`.
  - It safely calculates `error_rate = errors_count / requests_count` (guarding against division by zero if there are no requests).
  - **Threshold:** An error rate under 5% (`< 0.05`) is considered `[HEALTHY]`; anything equal or higher is flagged as `[NOT HEALTHY]`.

#### 2. Total CPU Usage
- **PromQL Query:** `100 * (1 - avg by(instance) (rate(node_cpu_seconds_total{mode='idle'}[5m])))`
- **Mechanism:**
  - `node_cpu_seconds_total` records seconds spent by each CPU core in different modes (`idle`, `system`, `user`, `iowait`).
  - Calculating `rate(...{mode='idle'}[5m])` yields the fraction of time the CPU spent idle.
  - Subtracting this from `1` gives the active CPU load, which is multiplied by `100` to convert to a percentage.
  - Prometheus returns metric values as strings in a two-item list `[timestamp, "value"]`. The script converts the value using `float(instance['value'][1])` and accumulates the total across instances.
  - **Threshold:** Total CPU usage under 95% is marked `[HEALTHY]`.

#### 3. I/O Wait Percentage
- **PromQL Query:** `100 * avg by(instance) (rate(node_cpu_seconds_total{mode='iowait'}[5m]))`
- **Mechanism:**
  - Measures the percentage of CPU time spent waiting for outstanding disk I/O requests.
  - High I/O wait indicates storage bottlenecks where CPU cores are stalled waiting for disk reads/writes.
  - **Threshold:** Under 5% is `[HEALTHY]`; higher values indicate disk bottlenecks and trigger `[NOT HEALTHY]`.

#### 4. Node Memory Usage Percentage
- **PromQL Query:** `100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))`
- **Mechanism:**
  - In Linux, unused memory is utilized for caching and buffers. Therefore, querying `node_memory_MemFree_bytes` gives a misleadingly high usage estimate.
  - Using `node_memory_MemAvailable_bytes` accurately represents how much memory can be allocated without causing the system to swap.
  - Dividing available bytes by total bytes gives the available ratio; subtracting from 1 and multiplying by 100 provides the actual memory utilization percentage.
  - **Threshold:** Under 90% is `[HEALTHY]`.

#### 5. Detecting Memory Leaks
- **PromQL Query:** `delta(process_resident_memory_bytes{job='kubernetes-services'}[24h]) > 0`
- **Mechanism:**
  - `process_resident_memory_bytes` tracks the resident set size (RSS) memory currently allocated to the process.
  - The `delta(...[24h])` function compares the memory footprint from 24 hours ago to now. A positive delta (`> 0`) indicates steady memory growth, which is a symptom of a potential memory leak.
  - For each detected service, the script calculates growth in megabytes:
    ```python
    mem_mb = float(leak["value"][1]) / 1000
    ```
  - **Threshold:** If no services show positive growth, it reports `[HEALTHY]`; otherwise, it raises a `[Warning]` and displays the affected service and growth amount.

#### 6. Loki Log Streaming & Error Filtering
- **Endpoint:** `http://localhost:31100/loki/api/v1/query_range`
- **LogQL Query:** `{service_name=~"simple-app-sender|simple-web-app"} |~ "(?i)ERROR|FAIL"`
- **Mechanism:**
  - Queries Loki over HTTP, using regex to match both applications and scanning for case-insensitive `ERROR` or `FAIL` tokens.
  - Response parsing unpacks nested streams and log entries:
    ```python
    problems = [
        line 
        for stream in data 
        for _, line in stream.get("values", [])
    ]
    ```
  - Wrapped in a `try...except requests.exceptions.RequestException` block to ensure that a Loki timeout or connection glitch does not crash the entire health checker.
  - **Threshold:** If `len(problems) == 0`, logs are marked `[HEALTHY]`; otherwise, it outputs a `[Warning]` with the count and lines.

#### 7. Terminal Styling & HTML Export with `rich`
- Initializes `Console(record=True)` from the `rich` library.
- Color codes output using semantic markup:
  - `[bold green][HEALTHY][/bold green]`
  - `[bold red][NOT HEALTHY][/bold red]`
  - `[bold yellow][Warning][/bold yellow]`
- Generates an HTML report via `console.save_html("cluster_metrics.html")`, producing a self-contained web page of the health check.

---

### Task Automation with Cron

To satisfy the bootcamp objective of running this check every 5 minutes on the K3s host VM (outside of K3s pods):

1. **Deploy Script on Host VM:**
   ```bash
   sudo mkdir -p SOME-PATH/scripts
   sudo cp cluster_health_check.py SOME-PATH/scripts/
   sudo chmod +x SOME-PATH/scripts/cluster_health_check.py
   ```

2. **Install Required Python Dependencies on VM:**
   ```bash
   sudo apt update
   sudo apt install -y python3 python3-pip
   pip3 install requests rich
   ```

3. **Configure User Crontab:**
   Edit the crontab using `crontab -e`:
   ```bash
   # Run cluster health check every 5 minutes and log output
   */5 * * * * /usr/bin/python3 SOME-PATH/scripts/cluster_health_check.py >> /var/log/cluster_health_check.log 2>&1
   ```

4. **Verify Execution:**
   Inspect the log file after 5 minutes:
   ```bash
   tail -n 30 /var/log/cluster_health_check.log
   ```

---

## Phase 3: Bash Resource Monitor

### The Bash Script

Here is the complete script (`vm_weekly_report.sh`):

```bash
#!/bin/bash
file='vm_weekly_report.log'
day=$(date +"%a")

case $day in
Mon | Tue | Wed | Thu | Fri | Sat)
  date >>$file
  ;;

Sun)
  date >$file
  ;;
esac

sar | grep %user >>$file
sar | grep '^Average:' >>$file
echo "" >>$file
free -h >>$file
echo "" >>$file
df -h '/dev/sda2' >>$file
echo "#-----------------#" >>$file
```

---

### Deep Dive: How the Script Works

This script collects native OS-level performance metrics and manages weekly file rotation.

#### 1. Weekly Log Rotation Mechanism
```bash
file='vm_weekly_report.log'
day=$(date +"%a")

case $day in
Mon | Tue | Wed | Thu | Fri | Sat)
  date >>$file
  ;;

Sun)
  date >$file
  ;;
esac
```
- `date +"%a"` returns the short day name (`Mon`, `Tue`, `Wed`, `Thu`, `Fri`, `Sat`, `Sun`).
- **Rotation Logic:**
  - When running Monday through Saturday, `date >> $file` **appends** to the existing log file.
  - When running on **Sunday**, `date > $file` **overwrites** (truncates) the file, resetting the weekly report with Sunday's initial timestamp.
  - This simple rotation mechanism ensures the report file never grows indefinitely.

#### 2. CPU Metrics via `sar` (System Activity Reporter)
- `sar` is part of the `sysstat` package. It reads historical CPU utilization samples gathered by the system activity collector:
  ```bash
  sar | grep %user >>$file
  sar | grep '^Average:' >>$file
  ```
  - `grep %user` captures the table header row, detailing columns like `%user`, `%nice`, `%system`, `%iowait`, `%steal`, and `%idle`.
  - `grep '^Average:'` grabs the final summary calculation line representing average CPU load across the day.

#### 3. Memory Statistics via `free`
- `free -h >>$file` appends human-readable memory statistics (megabytes/gigabytes), displaying total, used, free, shared, buff/cache, and available RAM, as well as Swap utilization.

#### 4. Disk Partition Utilization via `df`
- `df -h '/dev/sda2' >>$file` captures the disk capacity, used blocks, available space, and percentage utilization for the main root OS filesystem (`/dev/sda2`).

#### 5. Section Delimiters
- `echo "#-----------------#" >>$file` appends a clean separator at the end of each run to clearly separate successive snapshots.

---

### Task Automation with Cron

The objective requires running this Bash monitor at **3:00 PM every Sunday**.

1. **Deploy Script on Host VM:**
   ```bash
   sudo cp vm_weekly_report.sh /opt/monitoring/scripts/
   sudo chmod +x SOME-PATH/vm_weekly_report.sh
   ```

2. **Ensure `sysstat` is Installed & Enabled:**
   Since the script uses `sar`, `sysstat` must be installed and active:
   ```bash
   sudo apt install -y sysstat
   sudo sed -i 's/ENABLED="false"/ENABLED="true"/' /etc/default/sysstat
   sudo systemctl enable --now sysstat
   ```

3. **Configure User Crontab:**
   Edit crontab using `crontab -e`:
   ```bash
   # Run VM resource monitor every Sunday at 3:00 PM (15:00)
   0 15 * * 0 /opt/monitoring/scripts/vm_weekly_report.sh
   ```

4. **Sample Output (`vm_weekly_report.log`):**
   ```text
   Sun Oct 18 15:00:01 UTC 2026
   12:00:01 AM     CPU     %user     %nice   %system   %iowait    %steal     %idle
   Average:        all      3.42      0.00      1.15      0.22      0.00     95.21

                  total        used        free      shared  buff/cache   available
   Mem:           7.7Gi       2.1Gi       3.8Gi        18Mi       1.8Gi       5.3Gi
   Swap:          2.0Gi          0B       2.0Gi

   Filesystem      Size  Used Avail Use% Mounted on
   /dev/sda2        49G   14G   33G  30% /
   #-----------------#
   ```

---

## Reference Links

- [Prometheus and Grafana Bottleneck Analysis](https://beefed.ai/en/bottleneck-analysis-prometheus-grafana)
- [How to Automate Tasks with Cron Jobs in Linux](https://www.geeksforgeeks.org/linux-unix/how-to-automate-tasks-with-cron-jobs-in-linux/)
- [Prometheus HTTP API Documentation](https://prometheus.io/docs/prometheus/latest/querying/api/)
- [Grafana Loki HTTP API Documentation](https://grafana.com/docs/loki/latest/reference/api/)
- [Rich Python Library Documentation](https://rich.readthedocs.io/en/stable/)
