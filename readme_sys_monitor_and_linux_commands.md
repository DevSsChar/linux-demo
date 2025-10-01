# README — Simple System Monitor + Linux Commands Reference

This README contains:

1. A **simple system monitoring script** (CPU & memory logging + service checks + auto-restart),
2. A **line-by-line explanation** of that script,
3. Concise reference sections for **Process Management**, **Service Management (systemctl)**, and **I/O Redirection & Pipes** with practical examples.

---

## Simple Script (save as `sys_monitor.sh`)

```bash
#!/bin/bash

# Simple system monitor: log CPU/memory and restart services if down
LOGFILE="sys_monitor.log"
SERVICES=("ssh" "cron")

log_usage() {
    echo "---- $(date) ----" >> "$LOGFILE"
    echo "CPU Load: $(uptime)" >> "$LOGFILE"
    echo "Memory Usage:" >> "$LOGFILE"
    free -h >> "$LOGFILE"
}

check_services() {
    for service in "${SERVICES[@]}"; do
        systemctl is-active --quiet "$service"
        if [ $? -ne 0 ]; then
            echo "$(date): $service is DOWN. Restarting..." >> "$LOGFILE"
            systemctl restart "$service"
        else
            echo "$(date): $service is running." >> "$LOGFILE"
        fi
    done
}

while true; do
    log_usage
    check_services
    sleep 30
done
```

---

## Line-by-line explanation

**`#!/bin/bash`**
- Shebang. Tells the OS to run this script with `/bin/bash`. Enables Bash-specific features and expansions.

**`LOGFILE="sys_monitor.log"`**
- Assigns the filename to the variable `LOGFILE`.
- Quotes protect the string from word splitting (not strictly needed here but good habit).

**`SERVICES=("ssh" "cron")`**
- Declares a Bash array named `SERVICES` containing service names to monitor.
- Add or remove service names as needed (e.g., `"nginx"`, `"mysql"`).

**`log_usage() {`**
- Begins function `log_usage` which gathers and appends resource info to the logfile.

**`echo "---- $(date) ----" >> "$LOGFILE"`**
- `$(date)` runs the `date` command and substitutes its output.
- `echo` prints the timestamp line.
- `>> "$LOGFILE"` appends the output to the logfile. `>>` means append, not overwrite.

**`echo "CPU Load: $(uptime)" >> "$LOGFILE"`**
- `uptime` prints load averages and uptime; inline command substitution places it after `CPU Load:`.

**`echo "Memory Usage:" >> "$LOGFILE"`**
- Writes a label for the memory section.

**`free -h >> "$LOGFILE"`**
- `free -h` prints memory and swap usage in human-friendly units; the output is appended to the logfile.

**`}`**
- Ends the `log_usage` function body.

**`check_services() {`**
- Begins function `check_services` which iterates over `SERVICES` and ensures each is active.

**`for service in "${SERVICES[@]}"; do`**
- `"${SERVICES[@]}"` expands to every element of the array as a separate word.
- The `for` loop runs the body for each `service` value.

**`systemctl is-active --quiet "$service"`**
- Asks `systemctl` if the named service is active. `--quiet` suppresses output; the exit code tells us the result.

**`if [ $? -ne 0 ]; then`**
- `$?` holds the exit status of the previous command. `0` means success; non-zero means failure.
- `-ne` is numeric "not equal". So this condition is true if the service is not active.

**`echo "$(date): $service is DOWN. Restarting..." >> "$LOGFILE"`**
- Appends a log entry stating the service is down and a restart will be attempted.

**`systemctl restart "$service"`**
- Attempts to restart the service. Normally requires appropriate privileges (run script as a user who can restart services or via `sudo`).

**`else`** / **`echo "$(date): $service is running." >> "$LOGFILE"`**
- If `systemctl is-active` returned success (service running), append a "running" message to the log.

**`fi`** and **`done`**
- Close the `if` block and the `for` loop.

**`}`**
- Ends `check_services` function.

**`while true; do`**
- Starts an infinite loop. The loop will run forever unless the process is stopped (e.g., Ctrl+C or kill).

**`log_usage`**
- Calls the `log_usage` function defined earlier.

**`check_services`**
- Calls the `check_services` function to validate and restart services if needed.

**`sleep 30`**
- Pauses execution for 30 seconds before repeating the loop. Adjust as needed.

**`done`**
- Ends the `while` loop.

---

## 1) Process Management Commands
Useful commands to inspect and manage processes.

### `ps` — process snapshots
- `ps aux` — show all processes with full details (user, PID, CPU, MEM, command). Example column meanings:
  - `USER` — owner of process
  - `PID` — process id
  - `%CPU` / `%MEM` — CPU and memory usage
  - `VSZ` / `RSS` — virtual and resident memory sizes

**Examples**
```bash
ps aux
ps -u $USER   # processes owned by current user
ps aux | grep apache
```

### `top` / `bashtop` — real-time monitor
- `top` shows processes in real time (updates by default every few seconds). Useful interactive keys:
  - `q` — quit
  - `k` — kill process (asks for PID)
  - `M` — sort by memory
  - `P` — sort by CPU
- `bashtop` (or `htop`) gives a nicer, colored UI.

Example output snippet (columns):
```
PID USER   %CPU %MEM TIME+ COMMAND
1234 john  15.3  3.2  0:15 firefox
```

### `kill` — send signals by PID
- `kill 1234` — send SIGTERM (graceful) to PID 1234.
- `kill -9 1234` — send SIGKILL (force kill).

Example: find and kill firefox
```bash
ps aux | grep firefox
kill 1234
```

### `pkill` — kill by name
- `pkill firefox` — send SIGTERM to all processes named `firefox`.
- `pkill -9 firefox` — force kill.
- `pkill -u john` — target processes by user.

### `w` — who is logged in and what they're doing
- `w` shows logged-in users, their terminals, and active processes.
Example:
```
USER  TTY  FROM    LOGIN@ IDLE JCPU WHAT
john  pts/0 10.0.0.2 14:20  0.00s 0.12s vim file.txt
```

### `lscpu` — CPU architecture/info
- `lscpu` prints CPU topology and properties (cores, threads, model name, MHz). Example fields:
```
Architecture: x86_64
CPU(s): 8
Model name: Intel(R) Core(TM) ...
```

---

## 2) Service Management Commands (systemctl)
Commands to control systemd services.

### Start / Stop / Restart
```bash
sudo systemctl start apache2   # start service now
sudo systemctl stop apache2    # stop service now
sudo systemctl restart apache2 # stop then start (use after config changes)
```
- `start` runs the service immediately but does not change boot-time enabling.

### Enable / Disable at boot
```bash
sudo systemctl enable apache2   # start at boot
systemctl is-enabled apache2    # check if enabled -> "enabled" or "disabled"
sudo systemctl disable apache2  # prevent starting at boot
```
- `enable` affects future boots only; it does not start the service now.

### Check status
```bash
systemctl is-active ssh     # prints active/inactive
systemctl is-failed mysql   # check if service failed
```

**Use in scripts** (quiet checks by exit code):
```bash
if systemctl is-active --quiet ssh; then
    echo "SSH is running"
else
    echo "SSH is stopped"
fi
```

**Auto-restart failed services (example):**
```bash
if systemctl is-failed --quiet mysql; then
    echo "MySQL has failed! Restarting..."
    sudo systemctl restart mysql
fi
```

---

## 3) I/O Redirection & Pipes
How to redirect input/output and chain commands.

### `<` — input redirection
- `sort < unsorted.txt` — `sort` reads from `unsorted.txt` instead of stdin.
- `wc -l < data.txt` — count lines in file.

### `>` — output redirection (overwrite)
- `ps aux > processes.txt` — writes process list to file, replacing existing content.
- `command 2> errors.log` — redirect stderr (fd 2) to a file.

### `>>` — append
- `echo "Backup completed at $(date)" >> backup.log` — append to log file.

### `|` — pipe
- `ps aux | grep apache` — send output of `ps aux` into `grep`.
- `ps aux | sort -k4 -nr | head -5` — show top 5 memory consumers (`-k4` sorts by 4th column, `-nr` numeric reverse).
- `tail -f /var/log/syslog | grep error` — follow a log and show lines containing "error" in real time.

### Examples provided:
```bash
sort < unsorted.txt
wc -l < data.txt
ps aux > processes.txt
lscpu > cpu_info.txt
echo "=== $(date) ===" >> daily_log.txt
ps aux | wc -l
ps aux | grep apache
cat file.txt | grep "error" | sort | uniq
du -h /home | sort -rh | head -10
```

---

## Notes & Best Practices
- Running `systemctl` commands that change service state usually requires root privileges. Use `sudo` or run the script as root.
- When running a long-lived monitoring script in production, prefer running it as a systemd service or using a process manager instead of `nohup` & `&` so it starts on boot and restarts if it fails.
- Consider log rotation for `sys_monitor.log` (e.g., configure `logrotate`) to prevent the log from growing indefinitely.
- Add input validation to the script if you make it accept external input (e.g., ensure numeric sleep interval, check service names exist).

---

If you want, I can:
- convert this README into a downloadable Markdown or PDF file,
- create a `systemd` service unit file to run the monitor on boot,
- add email or Telegram alerts when a service fails to restart.

