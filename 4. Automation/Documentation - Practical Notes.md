# **Up:** [[4. Automation]]
---

Two scripts, both running on the K3s host VM (not inside pods) and scheduled with `cron`:

1. **Python Health Checker:** queries Prometheus and Loki, compares results to thresholds, prints a color-coded report, and saves it as HTML.
2. **Bash Resource Monitor:** writes CPU, RAM, and disk usage to a weekly log file.

---

## Setup: Exposing Prometheus and Loki

The script needs these metrics in Prometheus:

| Metric | Source | Used for |
|---|---|---|
| `node_cpu_seconds_total`, `node_memory_*` | `node-exporter` | CPU, I/O wait, memory |
| `http_server_requests_seconds_count` | Micrometer, through the OTel collector | HTTP error rate |
| `process_resident_memory_bytes` | `kubernetes-services` job | Memory leaks |

The host can't resolve cluster DNS names (`*.svc.cluster.local`), so I exposed both APIs as NodePorts:
- **Prometheus:** `http://localhost:30090`
- **Loki:** `http://localhost:31100`

---

## Phase 2: Python Health Checker

### The Script (`cluster_health_check.py`)

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

### What It Checks

| Check | Query | Healthy if |
|---|---|---|
| HTTP error rate | `rate(http_server_requests_seconds_count[5m])`, counting `4xx`/`5xx` statuses | < 5% |
| CPU usage | `100 * (1 - avg by(instance) (rate(node_cpu_seconds_total{mode='idle'}[5m])))` | < 95% |
| I/O wait | `100 * avg by(instance) (rate(node_cpu_seconds_total{mode='iowait'}[5m]))` | < 5% |
| Memory usage | `100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))` | < 90% |
| Memory leaks | `delta(process_resident_memory_bytes{job='kubernetes-services'}[24h]) > 0` | No results |
| Error logs (Loki) | `{service_name=~"simple-app-sender\|simple-web-app"} \|~ "(?i)ERROR\|FAIL"` | No results |

Things worth remembering:
- Prometheus returns values as `[timestamp, "value"]` strings, so convert them with `float(...[1])`.
- Use `MemAvailable`, not `MemFree`. Linux uses free RAM for cache, so `MemFree` makes usage look higher than it is.
- The Loki call is wrapped in `try/except` so a Loki outage doesn't crash the whole script.
- `rich`'s `Console(record=True)` + `save_html()` turns the terminal output into an HTML report.

### Cron: Every 5 Minutes

```bash
# Install dependencies
sudo apt install -y python3 python3-pip
pip3 install requests rich

# Deploy
sudo mkdir -p SOME-PATH/scripts
sudo cp cluster_health_check.py SOME-PATH/scripts/
sudo chmod +x SOME-PATH/scripts/cluster_health_check.py
```

`crontab -e`:
```bash
*/5 * * * * /usr/bin/python3 SOME-PATH/scripts/cluster_health_check.py >> /var/log/cluster_health_check.log 2>&1
```

Verify: `tail -n 30 /var/log/cluster_health_check.log`

---

## Phase 3: Bash Resource Monitor

### The Script (`vm_weekly_report.sh`)

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

### What It Does

- **Rotation:** On Sunday, `>` overwrites the file. On other days, `>>` appends to it. This keeps the file from growing forever.
- **CPU:** `sar` (from `sysstat`) prints the column header and the daily `Average:` line.
- **RAM:** `free -h`
- **Disk:** `df -h /dev/sda2` (root partition)

### Cron: Sundays at 3:00 PM

```bash
# sar needs sysstat installed and enabled
sudo apt install -y sysstat
sudo sed -i 's/ENABLED="false"/ENABLED="true"/' /etc/default/sysstat
sudo systemctl enable --now sysstat

# Deploy
sudo cp vm_weekly_report.sh SOME-PATH/scripts/
sudo chmod +x SOME-PATH/scripts/vm_weekly_report.sh
```

`crontab -e`:
```bash
0 15 * * 0 SOME-PATH/scripts/vm_weekly_report.sh
```

### Sample Output

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
