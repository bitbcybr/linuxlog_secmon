# openSUSE Leap 16 - auditd → remote audit server (Debian 13)

---

## Host roles
- Sender: openSUSE Leap 16 (default auditd)  
- Receiver: Debian 13 (clean install + install and running auditd listener)

---

## Assumptions
- Local network reachable: Receiver IP 192.168.122.19  
- Temporary test/POC only — transport is unencrypted. For some production env's, other secured forwarders will come into play  

---

## References 
- https://doc.opensuse.org/documentation/leap/security/html/book-security/cha-audit-setup.html  
- https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/security_guide/sec-understanding_audit_log_files
- And some stuff around Debian 13

---

## Receiver (Debian 13) - enable auditd TCP listener

1. Install auditd:
```bash
sudo apt update
sudo apt -y install auditd
```

2. Open auditd.conf with nano and edit manually:
```bash
sudo nano -l /etc/audit/auditd.conf
```
- Locate the line for tcp_listen_port (around line 27 in this installation).  
- Remove the leading "#" if present and set:
tcp_listen_port = 60

3. Save and exit

4. Restart auditd:
```bash
sudo systemctl restart auditd
sudo systemctl status auditd
```

5. Check for listen port 60:
```bash
sudo ss -a
```

Logs received by the Debian auditd will be written to /var/log/audit/audit.log (which is ok only for this testing purposes)

---

## Sender (openSUSE Leap 16) - enable audisp remote plugin

1. Install required packages:
```bash
sudo zypper -n install audit audit-audispd-plugins
```

2. Edit the au-remote plugin config for first activating it:
```bash
sudo nano -l /etc/audit/plugins.d/au-remote.conf
```
- Ensure these lines (or similar) are present and set:
active = yes
direction = out
path = /sbin/audisp-remote
type = always
format = string

- Save and exit.

3. Edit audisp-remote configuration:
```bash
sudo nano -l /etc/audit/audisp-remote.conf
```
- Set the remote server and port (replace values as needed, maybe your receiver is able to parse these logs even when receiving it on another port):
remote_server = 192.168.122.19
port = 60

- Save and exit.

4. Restart auditd / audispd:
```bash
sudo systemctl restart auditd
```
---

Looking at the Receiver logs coming in:
![Screenshot](./assets/suseleap16_auditd_capture_1.png)


---

## Additional Infos
### Open and inspect default rules manually:
```bash
sudo less /etc/audit/rules.d/audit.rules
```
In this test case default lines only include:
-D
-a task,never

---

### Check rotation and auditd config
Open and inspect rotation/retention in auditd config:
```bash
sudo nano -l /etc/audit/auditd.conf
```
Check settings such as max_log_file, num_logs, and related parameters. Save/exit when done.

---

### Quick test (generate an audit event)

On sender:
```bash
sudo su
# or hostnamectl worked for me also
```

On receiver:
```bash
# watch incoming audit log
tail -f /var/log/audit/audit.log
# or search for the hostname via cat and grep on whole file
```

---

## Notes & caveats 
- Ensure firewall on receiver allows TCP port 60 (or chosen port).  
- Verify audispd plugin path and service names on Leap 16 before editing.  

```

---

## Status
- If configured as above, logs should arrive on Debian 13 at /var/log/audit/audit.log — verify with tail or ausearch.
