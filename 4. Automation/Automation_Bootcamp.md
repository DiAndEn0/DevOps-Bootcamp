# Automation & Scripting Bootcamp

## 📖 Phase 1: Theoretical Foundation
* **Scripting Languages:** Review the strengths and ideal use-cases for Python, Bash, and PowerShell in infrastructure automation.
* **Task Scheduling:** Understand how the Linux `cron` daemon works, cron expressions, and how to schedule recurring tasks.
* **API Interaction:** Learn how to programmatically interact with REST APIs, specifically how to query Prometheus (PromQL via HTTP API) and Loki (LogQL via HTTP API).

---

## 💻 Phase 2: Practical Lab - Python Health Checker
**Objective:** Use Python to automate cluster health verification based on telemetry data.

1.  **Script Development:**
    * Write a Python script that makes HTTP requests to your Prometheus and Loki APIs.
    * *Prometheus Query:* Check for high error rates or resource bottlenecks.
    * *Loki Query:* Check for "ERROR" or "FATAL" strings in the recent log streams.
    * Output a clear status message indicating whether the monitored component is "Healthy" or "Unhealthy" based on these queries.
2.  **Task Automation:**
    * Move this Python script to the host OS of your K3s VM (not running inside a K3s pod).
    * Configure a `cronjob` on the Linux VM to execute this Python script every five minutes.
    * Log the script's output to a local health-check log file.

---

## 🚀 Phase 3: Practical Lab - Bash Resource Monitor
**Objective:** Use Bash to automate native OS monitoring and reporting.

1.  **Script Development:**
    * Write a Bash script that collects core VM health metrics:
        * CPU Usage.
        * RAM Usage.
        * Disk Usage.
    * Format the output neatly and append it to a log file (e.g., `/var/log/vm_weekly_report.log`).
2.  **Task Automation:**
    * Configure a `cronjob` on the VM to run this Bash script exactly at 3:00 PM every Sunday.
