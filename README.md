**1\. Executive Summary**

This report is commisioned to have a deep understading of the WantToCry ransomware attack exploits misconfigured or exposed SMB (Server Message Block) services to gain unauthorized access to systems that took place in Log(N) Pacific between **4 March 2026 and 5 March 2026**, the Windows virtual machine **vm-final-lab-wo** experienced a multi-stage cyber intrusion involving unauthorized network authentication, lateral movement through Windows network services, remote command execution, and ransomware deployment. The investigation identified that the attacker initially gained access through **NTLM network authentication**, which allowed the adversary to authenticate remotely without performing an interactive login. This authentication originated from the external IP address **107.140.235.185** and was recorded in the Windows Security logs as **Event ID 4624 (Logon Type 3)**.

Logon Type 3 indicates network authentication performed through services such as **SMB, RPC, WinRM, or WMI**. Because NTLM authentication allows credentials or authentication tokens to be reused across network services, the attacker did not need to know the user's password or actively log into the workstation referenced in the logs (**LAPTOP-U2VMN6LD**). The workstation listed in the event log therefore represents the origin of the authentication token rather than proof that the device was connected at the time of the attack.

After obtaining access, the attacker interacted with another system on the network named **WIN-IGJLHARSET8**, which indicates that **lateral movement through SMB-based administrative access occurred**. This activity suggests that the attacker attempted to expand their access across the environment by leveraging valid credentials and network services.

Subsequent telemetry shows that the attacker executed administrative utilities including **cmd.exe, powershell.exe, and ssh.exe**, demonstrating that they obtained elevated privileges and began interacting directly with the operating system. Later in the attack timeline, the **OpenSSH service was enabled**, which exposed the system to automated SSH scanning and brute-force attempts from several public IP addresses. However, these attempts occurred **after the system had already been compromised** and therefore were not the initial entry point.

The final stage of the attack involved **ransomware deployment**, which encrypted files using the extension **.want_to_cry** and generated a ransom note titled **i_want_to_cry.txt**. These artifacts are associated with **WantToCry/WannaCry-style ransomware variants**, indicating that the attacker's ultimate objective was data encryption and financial extortion.

Although the authentication behavior is consistent with techniques such as **NTLM relay or credential reuse**, the investigation did not uncover definitive artifacts proving relay activity. Nevertheless, the collected evidence clearly demonstrates **unauthorized network authentication, lateral movement, command execution with elevated privileges, and ransomware encryption activity**.
![Attack flow diagram](https://raw.githubusercontent.com/cfazuero1/Dissecting-the-Want-to-Cry-ransomware/main/a_flowchart_titled_attack_flow_diagram_illustrat.png)
**2\. Investigation Scope**

The investigation focused on the compromised Windows system **vm-final-lab-wo**, which operates as a virtual machine in an Azure environment. The analysis examined system activity occurring between **4 March 2026 and 5 March 2026**, beginning with the earliest suspicious authentication events and concluding with the ransomware encryption stage.

Evidence was collected from multiple telemetry sources including **Windows Security Event Logs, SMB operational logs, SSH authentication logs, Microsoft Defender telemetry, and Microsoft Sentinel queries**. These sources provided insight into authentication activity, process execution, network connections, and file system changes occurring during the incident.

By correlating data across these sources and reconstructing a forensic timeline, investigators were able to determine the sequence of attacker actions, identify the systems involved in lateral movement, and evaluate the overall impact of the intrusion.

**3\. Investigation Tools**

Several forensic and security monitoring tools were used during the investigation to analyze system logs and reconstruct the attack timeline.

- Hayabusa - Windows event log threat hunting
- Timesketch - forensic timeline reconstruction
- Microsoft Defender for Endpoint - endpoint telemetry analysis
- Microsoft Sentinel - security event correlation and querying
- Virus Total - Check for Ips and files.

Hayabusa was used to parse the Windows event logs and detect suspicious authentication and process activity. Timesketch enabled investigators to visualize events chronologically, allowing them to identify patterns and correlations between authentication events, command execution, and ransomware deployment.

Microsoft Defender provided endpoint visibility into process execution and file modifications, while Microsoft Sentinel allowed investigators to perform advanced queries across the environment to identify anomalous authentication patterns.

Together, these tools enabled a comprehensive reconstruction of the attack lifecycle.

**4\. Security Log Analysis**

Log analysis performed using **Hayabusa** identified multiple alerts indicating suspicious remote activity and authentication anomalies.
![Hayabusa command](https://raw.githubusercontent.com/cfazuero1/Dissecting-the-Want-to-Cry-ransomware/main/hayabusa%20command.png)
![Hayabusa results](https://raw.githubusercontent.com/cfazuero1/Dissecting-the-Want-to-Cry-ransomware/main/result-hayabusa.png)

### Windows Security Timeline
[timeline_security.csv](https://raw.githubusercontent.com/cfazuero1/Dissecting-the-Want-to-Cry-ransomware/main/timeline_security.csv)

### SMB Timeline Events
[smb-timeline-events-results.csv](https://raw.githubusercontent.com/cfazuero1/Dissecting-the-Want-to-Cry-ransomware/main/smb-timeline-events-results.csv)

### SSH Timeline for Timesketch
[timeline_ssh_for_timesketch.csv](https://raw.githubusercontent.com/cfazuero1/Dissecting-the-Want-to-Cry-ransomware/main/timeline_ssh_for_timesketch.csv)

Key detections included:

- External Remote SMB Logon from Public IP
- Program Executed Using Proxy/Local Command via SSH.EXE
- External Remote RDP Logon from Public IP
- Failed Logon From Public IP
- Process Ran With High Privilege
- Password Guessing Activity

These alerts indicate that the compromised system experienced **multiple remote authentication attempts from external IP addresses** and that attackers attempted to access network services such as SMB and RDP.

The analysis also revealed a large volume of informational events including **NTLM authentication activity (over 11,000 events)**. This volume suggests that network authentication mechanisms were heavily used during the attack and supports the hypothesis that the attacker relied on **credential reuse or NTLM-based authentication techniques** rather than interactive login.

Additional detections included **credential manager enumeration and local account discovery**, which are common reconnaissance activities performed by attackers after gaining initial access.

**5\. Initial NTLM Network Authentication**

The earliest confirmed suspicious activity appears in the Windows Security logs as **Event ID 4624**, indicating successful authentication.

![NTLM authentication diagram](https://raw.githubusercontent.com/cfazuero1/Dissecting-the-Want-to-Cry-ransomware/main/NTML.png)

Key attributes include:

Event ID: 4624  
Logon Type: 3 (Network Logon)  
Authentication Package: NTLMv2  
Source IP: 107.140.235.185  
Workstation Name: LAPTOP-U2VMN6LD  
Elevated Token: Yes

Logon Type 3 represents authentication through network services rather than interactive login sessions. This type of authentication is typically associated with services such as **SMB, RPC, WinRM, and WMI**, which allow remote administrative access to Windows systems.

Because NTLM authentication allows credentials to be reused across services, the attacker could authenticate to the system without knowing the user's password or controlling the workstation listed in the log. The workstation name therefore represents the **source of the authentication token** rather than confirmation that the device was actively involved in the attack.

**6\. Possible NTLM Relay or Credential Reuse**

The authentication behavior observed in the logs is consistent with attacks that exploit NTLM authentication mechanisms. One possible explanation is an **NTLM relay attack**, where an attacker intercepts an NTLM authentication response and forwards it to another system to gain access.

In such scenarios, the attacker impersonates a legitimate user without possessing the user's password. However, the investigation did not identify definitive artifacts confirming that an NTLM relay tool was used.

Because of this limitation, the investigation concludes that **NTLM credential reuse occurred**, while acknowledging that NTLM relay remains a plausible explanation.

**7\. SMB Activity and Lateral Movement**

During the investigation, SMB operational logs revealed communication with another system on the network:

**WIN-IGJLHARSET8**

Evidence suggests that the attacker used SMB connections to interact with this system and perform activities such as software installation and system configuration changes.

Acording to "Exposed SMB: The Hidden Risk Behind ‘WantToCry’ Ransomware Attacks" Written by Umar Khan A, after obtaining access, they enumerate shared drives and network resources to identify valuable data repositories. The ransomware payload is then deployed, encrypting files across accessible systems and network shares while leaving a ransom note demanding payment for decryption. This type of attack highlights how exposed network services and weak credentials can enable attackers to move laterally within a network and rapidly compromise critical data.
![WantToCry ransomware attack diagram](https://raw.githubusercontent.com/cfazuero1/Dissecting-the-Want-to-Cry-ransomware/main/diagram_rasomware.png)
Reference: https://www.seqrite.com/blog/wanttocry-ransomware-smb-vulnerability/

SMB-based lateral movement is a common technique used by attackers after obtaining valid credentials. Once authenticated, attackers can access administrative shares and execute commands on remote systems.

The interaction with **WIN-IGJLHARSET8** indicates that the attacker attempted to **expand their presence within the network** rather than remaining confined to a single host.

**8\. Guest Account Assessment**

The investigation identified that the **Guest account was enabled** on the compromised system. While the presence of an enabled Guest account may raise security concerns, this account has significant restrictions under default Windows configurations.

Guest accounts typically cannot:

- Modify files belonging to other users
- Install software
- Access administrative shares

Instead, Guest accounts are generally limited to basic enumeration activities such as listing accessible network shares or viewing publicly available files.

Because the attack involved administrative command execution and ransomware deployment, the Guest account cannot explain the observed compromise.

**9\. OpenSSH Service Exposure**

System logs show that the **OpenSSH service started on 5 March 2026 at 14:30:50**, exposing port **22** to the internet.

Once the service became visible externally, automated scanning tools began attempting authentication against the server. This behavior is common when SSH services are exposed publicly, as automated bots continuously scan for weak credentials.

The timing of these events indicates that SSH scanning occurred **after the initial compromise**, confirming that SSH brute-force attempts were not the original entry vector.

**10\. Command Execution**

Following the suspicious authentication event, the system executed several administrative utilities including:

- cmd.exe
- powershell.exe
- ssh.exe

These tools are commonly used by attackers to perform reconnaissance, execute commands, and deploy malware. Their execution indicates that the attacker successfully obtained privileges sufficient to interact with the operating system.

This stage represents the transition from **initial access to active exploitation**, where the attacker begins performing actions to achieve their objectives.

**11\. Ransomware Deployment**

The final stage of the attack involved ransomware encryption of files on the compromised system.

Encrypted files were renamed using the extension:

.want_to_cry

A ransom note titled:

i_want_to_cry.txt

was also discovered.

Threat intelligence analysis associates these artifacts with **WantToCry/WannaCry-style ransomware variants**, indicating that the attacker intended to disrupt system availability and demand payment in exchange for decryption.

**12\. Indicators of Compromise**

The investigation identified several suspicious IP addresses associated with scanning and authentication attempts:

107.140.235.185  
142.93.227.179  
161.35.148.68  
80.94.92.186  
193.32.162.145  
104.155.119.125  
185.156.73.59  
92.63.197.9

Malicious artifacts identified during the investigation include:

.want_to_cry  
i_want_to_cry.txt

These indicators should be used to update security monitoring rules and threat detection systems.

**13\. Forensic Attack Timeline**

4 March 2026

15:07:24 - NTLM network authentication (Event ID 4624)  
15:09:10 - External SMB logon detected  
15:12:31 - SMB interaction with **WIN-IGJLHARSET8**  
15:14:02 - Credential enumeration activity  
15:16:45 - Local account discovery  
15:18:17 - Administrative command execution begins

5 March 2026

14:30:50 - OpenSSH service started  
14:31:12 - External SSH scanning begins  
14:33:40 - Program executed via SSH proxy  
14:36:10 - Ransomware execution begins

**14\. MITRE ATT&CK Mapping**

The attack aligns with the following techniques from the **MITRE ATT&CK framework**:

T1550 - Use of Stolen Credentials  
T1021 - Remote Services (SMB / RDP)  
T1087 - Account Discovery  
T1059 - Command and Scripting Interpreter  
T1486 - Data Encrypted for Impact

Mapping attacker activity to ATT&CK techniques allows defenders to understand which adversary tactics were used and to improve detection capabilities for future incidents.

**15\. Security Recommendations**

To prevent similar incidents in the future, several defensive measures should be implemented.

**Authentication Hardening**

Organizations should reduce reliance on NTLM authentication where possible and implement modern authentication mechanisms such as Kerberos or certificate-based authentication. Multi-factor authentication should be enforced for all administrative accounts to prevent credential reuse attacks.

**Network Security Controls**

SMB signing should be enforced to prevent NTLM relay attacks. Additionally, network segmentation should be implemented to limit lateral movement between systems.

**Service Exposure Management**

Services such as SSH should not be exposed directly to the internet unless necessary. When remote administration is required, access should be restricted through VPN or jump hosts and monitored closely.

**Account Security**

The Guest account should be disabled unless explicitly required. Strong password policies and account lockout mechanisms should also be implemented to prevent brute-force attacks.

**Monitoring and Detection**

Security teams should monitor for suspicious authentication events such as **Event ID 4624 (Logon Type 3) from external IP addresses**. Additional monitoring should focus on process execution events involving administrative tools such as PowerShell and cmd.

**Backup and Recovery**

Organizations should maintain offline backups and regularly test recovery procedures to ensure that systems can be restored quickly in the event of ransomware incidents.
