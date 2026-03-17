# Python Automation, Ansible, Salt, and Monitoring

> HRT labels this role "Systems Automation / Python". Strong Python development skills are required — not just scripting.

## Python for System Automation

```python
import subprocess, os, pathlib
import psutil      # process/system info
import paramiko    # SSH automation

# Run shell commands from Python
result = subprocess.run(['ls', '-la'], capture_output=True, text=True, check=True)
print(result.stdout)
# subprocess.run() = blocking, simple
# subprocess.Popen() = non-blocking, streaming

# File operations
p = pathlib.Path('/var/log')
for f in p.glob('*.log'):
    print(f, f.stat().st_size)

# Process monitoring with psutil
for proc in psutil.process_iter(['pid', 'name', 'cpu_percent']):
    if proc.info['cpu_percent'] > 90:
        print(proc.info)

# SSH with paramiko
client = paramiko.SSHClient()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect('host', username='user', key_filename='/path/to/key')
stdin, stdout, stderr = client.exec_command('df -h')
print(stdout.read().decode())
client.close()
```

## Ansible (HRT Uses This)

```bash
# Ad-hoc commands
ansible all -i inventory.ini -m ping
ansible web -m shell -a 'df -h'
ansible web -m apt -a 'name=nginx state=latest' --become

# Run playbook
ansible-playbook -i inventory.ini playbook.yml --check  # dry-run
ansible-playbook -i inventory.ini playbook.yml -v       # verbose
```

```yaml
# Example playbook
---
- name: Configure trading server
  hosts: trading_servers
  become: yes
  vars:
    app_user: trader
  tasks:
    - name: Install required packages
      apt:
        name: ['python3', 'tcpdump', 'htop']
        state: present
        update_cache: yes

    - name: Copy config file
      template:
        src: templates/app.conf.j2
        dest: /etc/myapp/app.conf
        owner: '{{ app_user }}'
        mode: '0640'
      notify: restart app

    - name: Ensure service is running
      systemd:
        name: myapp
        state: started
        enabled: yes

  handlers:
    - name: restart app
      systemd:
        name: myapp
        state: restarted
```

## SaltStack (HRT's Primary Config Management)

```bash
# Test connectivity
salt '*' test.ping
salt 'web*' test.ping

# Remote execution
salt '*' cmd.run 'df -h'
salt '*' service.restart nginx
salt 'db01' pkg.install mysql-server

# Apply states (dry-run)
salt '*' state.apply nginx test=True
salt '*' state.apply nginx

# Grains (minion metadata)
salt 'web01' grains.items
salt -G 'os:Debian' test.ping   # target by grain

# Pillars (secure per-minion data)
salt '*' pillar.items
```

## systemd Deep Dive

```bash
# Service management
systemctl start|stop|restart|reload|status <service>
systemctl enable|disable <service>       # persist across reboots
systemctl list-units --type=service --state=running
systemctl list-units --failed

# Boot performance analysis
systemd-analyze                          # total boot time
systemd-analyze blame                    # per-unit time
systemd-analyze critical-chain           # boot critical path

# Log management
journalctl -u <service> -f              # follow service logs
journalctl --since '10 min ago'
journalctl -p err -b                    # errors since boot
journalctl --vacuum-size=500M           # trim logs
```

```ini
# Writing a unit file (know this cold)
[Unit]
Description=My Trading App
After=network.target

[Service]
Type=simple
User=trader
ExecStart=/opt/app/bin/myapp
Restart=on-failure
RestartSec=5s
LimitNOFILE=65536
Environment=CONFIG_PATH=/etc/myapp/config.yml

[Install]
WantedBy=multi-user.target
```

## Prometheus + Grafana

```yaml
# Architecture:
# Prometheus server → scrapes /metrics from exporters
# Alertmanager → handles alerts
# Grafana → visualization

# PromQL examples
node_cpu_seconds_total                      # raw counter
rate(node_cpu_seconds_total[5m])            # per-second rate over 5m
100 - avg(rate(node_cpu_seconds_total{mode='idle'}[5m])) * 100  # CPU %

# Alert rule
groups:
  - name: system
    rules:
      - alert: HighCPU
        expr: 100 - avg(rate(node_cpu_seconds_total{mode='idle'}[5m]))*100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: 'CPU usage above 90% for 5 minutes'
```

**Key exporters:**
- `node_exporter` — OS metrics (CPU, mem, disk, network)
- `process_exporter` — per-process metrics
- `blackbox_exporter` — probe endpoints (HTTP, TCP, ICMP)
