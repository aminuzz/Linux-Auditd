# Linux-Auditd
A host-based file integrity monitoring project using Linux Auditd. Configured watch rules for protected files, simulated attacks using ELF binaries, and analyzed audit logs to attribute each modification, demonstrating compliance monitoring and incident response skills.

## Overview
This project involves designing and implementing a host-based file integrity 
monitoring system using **Linux Auditd** which is the Linux kernel's built-in audit 
framework. Watch rules were configured for a set of protected files to detect 
unauthorized modifications at the kernel level, achieving 100% detection 
coverage across all monitored files. Three simulated attack executables were 
then run against the protected directory, and audit logs were analyzed 
exclusively using **ausearch** and filter keys to correlate each attack binary 
with the file it modified, demonstrating a practical incident response and 
forensic analysis workflow.


### How Auditd Works
| Component | Role |
|-----------|------|
| Kernel audit system | Generates audit events when a watched file is accessed or modified |
| auditd | Userspace daemon that writes events to `/var/log/audit/audit.log` |
| auditctl | Sets and lists rules in the running kernel (temporary; lost on restart) |
| `/etc/audit/rules.d/audit.rules` | Persistent rules file, loaded on boot via `sudo augenrules --load` |
| ausearch / aureport | Tools for querying and summarizing the audit log |


## Environment

| Component | Details |
|-----------|---------|
| OS | Kali Linux |
| Virtualization | VMware |
| Shell | Bash |
| Audit Daemon | auditd |
| Key Tools | auditctl, ausearch, aureport, augenrules |

### Directory Structure
The attack binaries hardcode a specific file path for the protected files 
directory. This was discovered by inspecting the binaries with `strings`:

```bash
strings attack-a | grep -i protected
```
<img width="998" height="181" alt="image" src="https://github.com/user-attachments/assets/ee5220ab-bba8-4e97-bd47-113018456ea9" />


The expected path was then recreated to ensure the binaries could locate 
and modify the target files:

```bash
sudo mkdir -p /home/codepath/project2-main
sudo cp -r ~/protected_files /home/codepath/project2-main/protected_files
sudo chown -R kali:kali /home/codepath
```

> **Note:** The `chown` command was necessary because files copied with 
> `sudo cp` default to root ownership, which would have prevented the 
> attack binaries from writing to the protected files.


## Setup

### 1. Starting Auditd
Auditd was started and its status was verified to confirm it was 
actively listening for events:

```bash
sudo systemctl start auditd
sudo systemctl status auditd
```
<img width="2552" height="1060" alt="image" src="https://github.com/user-attachments/assets/162a2ceb-6ce2-40e4-b9ef-0883a51c77c2" />



The output confirms the service is `active (running)` and that auditd 
4.1.2 is listening for events. Note the `disabled` flag in the loaded 
line so if you want it to persist across reboots, the service should also be enabled:

```bash
sudo systemctl enable auditd
```


### 2. Project Files
The project files were downloaded and extracted into the working directory:

```bash
wget https://github.com/codepath/project2/archive/main.zip
unzip main.zip
```

<img width="1912" height="919" alt="step1" src="https://github.com/user-attachments/assets/3bbb5566-3239-4f6a-8587-edfc944dc698" />
<img width="1913" height="544" alt="step2" src="https://github.com/user-attachments/assets/613f2ca6-e8ef-47ec-8d95-53cec815031a" />



The extracted directory contains two key components:
- **Attack binaries** — three compiled ELF executables (`attack-a`, `attack-b`, `attack-c`) 
  that each modify a different protected file
- **Protected files** — the 10 files that will be monitored by auditd
<img width="1912" height="90" alt="protected_files_step_3" src="https://github.com/user-attachments/assets/2965f398-04fd-47ab-8fd7-b8a58d8decee" />


### 3. Writing the Watch Rules
Watch rules were written for each of the 10 protected files in 
`/etc/audit/rules.d/audit.rules`. Each rule follows this syntax:
```bash
-w <absolute path> -p <permissions> -k <key>
```
- `-w` — the file to watch (absolute path only, `~` is not expanded)
- `-p w` — monitor write operations only
- `-k` — a unique key per file for easy log filtering
<img width="2553" height="1278" alt="audit-rules" src="https://github.com/user-attachments/assets/f6174965-b259-4c5d-a144-db22360916a2" />



### 3. Loading the Rules
Rules were loaded permanently using:

```bash
sudo augenrules --load
```

And verified with:

```bash
sudo auditctl -l
```
<img width="2548" height="1075" alt="image" src="https://github.com/user-attachments/assets/890c684b-f6af-4551-8e14-09e7d8fe2042" />


## Running the attacks
Before running each attack, the audit log was cleared to ensure only 
events from that binary appeared in the output:

```bash
sudo truncate -s 0 /var/log/audit/audit.log
```

Each binary was then executed and the log was queried immediately after 
using the attack's path as a filter:

```bash
./attack-a
sudo ausearch -i -x ./attack-a
```

### Attack A
<img width="2554" height="1230" alt="image" src="https://github.com/user-attachments/assets/dc64c7ee-ee92-40b8-98f7-626a649727e0" />

Let's filter out the noise:
<img width="2547" height="192" alt="image" src="https://github.com/user-attachments/assets/24ebf97d-2ce5-431b-9047-076eda8be5e6" />

The ausearch output shows:
- `key=cloudia` — the watch rule for `cloudia.txt` fired
- `exe=./attack-a` — the responsible binary
- `a2=O_WRONLY|O_CREAT|O_APPEND` — a write operation (not a read)

### Attack B
