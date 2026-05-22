# Remote Code Execution Through Local File Inclusion in PHP Applications

<img width="1584" height="672" alt="image (1)" src="https://github.com/user-attachments/assets/f844cd52-f81c-4bb4-adf6-892ebbaf723d" /><br>


This report demonstrates how a PHP web application was exploited by chaining Local File Inclusion (LFI) with SSH log poisoning to achieve Remote Code Execution (RCE). Using Burp Suite and Netcat, sensitive files were accessed, commands were executed, and the flag was successfully retrieved.

<b>Introduction</b>

The objective of this Capture The Flag (CTF) challenge was to analyze and exploit vulnerabilities present in a web application in order to retrieve the hidden flag from the target server. During the assessment, multiple security weaknesses were identified and chained together, ultimately leading to remote command execution on the system.

The primary tool used during the exploitation process was Burp Suite. Burp Suite was used to intercept, modify, and replay HTTP requests sent to the vulnerable web application. This allowed detailed analysis of server responses and enabled the testing of various payloads against the target endpoint.

The target application was hosted on an Ubuntu server running Apache 2.4.41 and exposed a vulnerable parameter inside the file profile.php. Through careful manipulation of HTTP requests using Burp Suite, it was discovered that the application failed to properly validate user-controlled file paths, making it vulnerable to a Local File Inclusion (LFI) attack.

The exploitation process eventually evolved from simple file disclosure into full Remote Code Execution (RCE) through the use of log poisoning techniques. The final stage involved locating and reading the flag file stored on the server.

<b>Discovering the Local File Inclusion Vulnerability</b>

The first phase of the assessment focused on testing the img parameter inside the following endpoint:
```
/profile.php?img=
```
Using Burp Suite’s intercept and repeater functionality, specially crafted directory traversal payloads were inserted into the request in an attempt to access files outside the intended application directory.

The following HTTP request was sent to the server:

```
GET /profile.php?img=....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd HTTP/1.1
Host: 10.66.173.153:50000
```
The application responded successfully with the contents of the /etc/passwd file:

<img width="1813" height="1027" alt="1" src="https://github.com/user-attachments/assets/d58b4cd5-15ff-4203-b771-46a273efcad3" />


This behavior confirmed the presence of a Local File Inclusion vulnerability combined with directory traversal. The payload ....// was specifically chosen because it can bypass poorly implemented filters that attempt to block standard traversal sequences such as ../.

The successful disclosure of /etc/passwd demonstrated that arbitrary files on the operating system could be accessed through the vulnerable application. At this point, the server was already considered critically exposed because sensitive files and system information could be retrieved remotely.

<b>Accessing System Log Files</b>

After confirming the LFI vulnerability, the next objective was identifying files that could help escalate the attack further. In many Linux environments, authentication log files are particularly useful because they often contain user-controlled data generated during failed login attempts or malformed connections.

Using Burp Suite, another request was crafted to access the SSH authentication logs located at /var/log/auth.log.

The request used was:
```
GET /profile.php?img=....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//var/log/auth.log HTTP/1.1
Host: 10.66.173.153:50000
```
The response returned valid log entries from the server:
```
May 17 21:39:01 ip-10-66-173-153 CRON[2127]: pam_unix(cron:session): session opened for user root by (uid=0)
May 17 21:39:01 ip-10-66-173-153 CRON[2127]: pam_unix(cron:session): session closed for user root
```
<img width="1813" height="1027" alt="2" src="https://github.com/user-attachments/assets/ca0f38b1-1349-4bab-8a4d-bd4ef661fc65" />

This confirmed that the application could read sensitive system logs directly through the LFI vulnerability.

<b>Testing for Log Poisoning</b>

The next step was determining whether attacker-controlled input could be written into the authentication logs. To test this, a connection was manually established to the SSH service running on port 22 using Netcat.

The following command was executed:
```
nc 10.66.173.153 22
```
The SSH server returned its banner:
```
SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.11
```
<img width="1174" height="651" alt="3" src="https://github.com/user-attachments/assets/2d16cf41-027e-479a-bd94-43ebe09fe98b" />

Instead of completing a legitimate SSH handshake, the text hello was sent to the server. As expected, the SSH daemon rejected the malformed connection attempt.

Afterwards, the authentication log file was opened again through the vulnerable endpoint using Burp Suite. A new log entry appeared:
```
sshd[2223]: error: kex_exchange_identification: client sent invalid protocol identifier "hello"
```
<img width="1785" height="399" alt="4" src="https://github.com/user-attachments/assets/0a537f3e-af85-4f89-9dc4-abc8f340bc94" />

This confirmed that arbitrary user input supplied through the SSH connection was being stored inside the authentication logs. Since those logs were also accessible through the LFI vulnerability, the environment became vulnerable to log poisoning.

Log poisoning is a technique where attackers inject executable code into log files and later force the application to include those files. If the included file is interpreted by PHP, the injected payload may execute directly on the server.

<b>Achieving Remote Code Execution</b>

To escalate the attack from Local File Inclusion to Remote Code Execution, PHP code was injected into the SSH authentication logs.

Another Netcat connection was established to the SSH service, but this time the following PHP payload was sent:
```
<?php system("id"); ?>
```
<img width="1188" height="632" alt="6" src="https://github.com/user-attachments/assets/d9af0adb-2d90-482f-84d9-70406838671d" />

Once the payload was written into the log file, Burp Suite was used again to include the poisoned auth.log file through the vulnerable img parameter.

When the application processed the included log file, the embedded PHP code executed successfully and produced the following output:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data),4(adm),27(sudo)
```
<img width="1785" height="399" alt="5" src="https://github.com/user-attachments/assets/a7d3808c-a942-4582-875c-5830f2005184" />

This confirmed successful remote command execution on the server under the privileges of the www-data user, which is commonly used by Apache web servers on Ubuntu systems.

At this stage, the attack had fully compromised the web application environment.

<b>Locating and Retrieving the Flag</b>

With command execution successfully achieved, the next objective was locating the flag file on the server. Since web applications are commonly stored inside /var/www/html, directory enumeration was performed through additional PHP payloads injected into the logs.

The following payload was used:
```
<?php system("ls /var/www/html"); ?>
```
The command output revealed multiple files, including a suspicious text file named:
```
505eb0fb8a9f32853b4d955e1f9123ea.txt
```
<img width="1797" height="614" alt="7" src="https://github.com/user-attachments/assets/a97bb767-5005-4124-a11c-66d215e5c356" />

To retrieve the contents of the file, another payload was injected:
```
<?php system("cat /var/www/html/505eb0fb8a9f32853b4d955e1f9123ea.txt"); ?>
```
<img width="1181" height="619" alt="9" src="https://github.com/user-attachments/assets/de8ce9aa-715e-44d8-b3da-c5fad1b3bc58" />

After including the poisoned log file once more through the LFI vulnerability, the contents of the text file were displayed successfully, revealing the challenge flag.

<img width="1866" height="617" alt="10" src="https://github.com/user-attachments/assets/9dae1dc9-d43e-474e-b15f-9adb987b24f7" /><br>

<b>Security Analysis</b>

This challenge demonstrated how multiple vulnerabilities can be chained together to achieve complete server compromise. The initial Local File Inclusion vulnerability alone already represented a severe issue because it allowed arbitrary file disclosure. However, the presence of writable log files and unsafe file inclusion behavior enabled escalation to Remote Code Execution.
The complete attack chain can be summarized as follows:
```
Local File Inclusion → Authentication Log Access → Log Poisoning → PHP Code Execution → Flag Retrieval
```
The challenge also highlighted the importance of secure server-side input validation and safe handling of dynamically included files. Improperly sanitized file paths, combined with the ability to inject attacker-controlled content into log files, created a highly exploitable environment.

[![TryHackMe](https://img.shields.io/badge/-TryHackMe-3d9970?style=for-the-badge&logo=tryhackme&logoColor=white)](https://tryhackme.com/p/TheRootGod)
[![LinkedIn](https://img.shields.io/badge/-LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/nistor-alexandru-cosmin-a900b5278/)

