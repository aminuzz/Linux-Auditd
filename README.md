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

## Project Files
| File | Description |
|------|-------------|
| `attack-a`, `attack-b`, `attack-c` | Compiled ELF binaries that each modify a different protected file |
| `protected_files/` | The 10 files monitored by auditd watch rules |
| `audit.rules` | Watch rules loaded into auditd via `augenrules` |
| `audit.log` | Audit log captured during the attack runs |

## Environment

| Component | Details |
|-----------|---------|
| OS | Kali Linux |
| Virtualization | VirtualBox |
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
- `a2=O_WRONLY|O_CREAT|O_APPEND` — a write operation 

### Attack B
<img width="2549" height="225" alt="image" src="https://github.com/user-attachments/assets/408717c6-d836-4ae5-9938-b984e437c76c" />

The ausearch output shows:
- `key=oakley` & `key=squeaky` — the watch rule for `oakley.txt` and `squeaky.txt` fired
- `exe=./attack-b` — the responsible binary
- `a2=O_WRONLY|O_CREAT|O_APPEND` — a write operation 

### Attack C
<img width="2542" height="141" alt="image" src="https://github.com/user-attachments/assets/7710a101-956e-4e52-a44a-b12890d940a6" />

The ausearch output shows:
- `key=precipitation` — the watch rule for `precipitation.csv` fired
- `exe=./attack-c` — the responsible binary
- `a2=O_WRONLY|O_CREAT|O_APPEND` — a write operation

### Attribution Summary
| Attack Binary | Modified File(s) | Operation |
|---------------|------------------|-----------|
| `attack-a` | `cloudia.txt` | `O_WRONLY\|O_CREAT\|O_APPEND` |
| `attack-b` | `oakley.txt`, `squeaky.txt` | `O_WRONLY\|O_CREAT\|O_APPEND` |
| `attack-c` | `precipitation.csv` | `O_WRONLY\|O_CREAT\|O_APPEND` |

## Lessons Learned
- The first set of rules I wrote used `~/protected_files/...` as the watch path. The rules loaded without error but never fired so I knew auditd treats the tilde literally. Switching to the full absolute path fixed it.

- At one point a rule had an incorrect path which caused `auditctl -R` to exit with `res=no` and stop processing the file entirely, meaning none of the rules below it loaded. I didn't catch it until I ran `sudo auditctl -l` and noticed the list was shorter than expected. Now I verify loaded rules after every reload.

- I initially ran the binaries from my working directory and got no log entries. Running `strings attack-a | grep protected` revealed they expected files at a completely different hardcoded path. Once I recreated that directory structure the attacks worked.

- After copying the protected files with `sudo cp`, they were owned by root so the binaries couldn't write to them — again no log entries. A `sudo chown -R kali:kali /home/codepath` resolved it.

- In a real environment, an attacker with root access can wipe the audit log entirely. Forwarding logs to a remote SIEM in real time ensures the record survives even if the host is compromised.

## Tools Used 🛠️

- **auditd** — Linux kernel audit daemon that writes events to `/var/log/audit/audit.log`
- **auditctl** — Set and verified watch rules in the running kernel
- **augenrules** — Loaded the persistent rules file into the running kernel without a restart
- **ausearch** — Queried the audit log by executable (`-x`) and used `grep` to filter output by attack file name
- **strings** — Inspected the attack binaries to discover the hardcoded protected files path
