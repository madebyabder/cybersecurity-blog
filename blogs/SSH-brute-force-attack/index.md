# What Actually Happens During an SSH Brute-Force Attack?

SSH is one of the most common ways administrators remotely access Linux servers. It is also one of the services that attackers routinely probe when a server is exposed to the Internet.

A brute-force attack against SSH is simple in concept: repeatedly attempt authentication with different usernames, passwords, or password combinations until one works.

But what actually happens during the attack?

This article breaks the process down from the network connection to the authentication logs, then looks at what a defender can do about it.

---

## 1. First: What Is SSH?

**SSH (Secure Shell)** is a cryptographic network protocol used to securely access remote systems.

A typical connection looks like this:

```text
SSH Client                         SSH Server
    |                                  |
    | ---- TCP connection -----------> |
    |                                  |
    | <--- SSH negotiation ------------|
    |                                  |
    | ---- Authentication ------------>|
    |                                  |
    | <--------- Access ---------------|
```

SSH commonly listens on **TCP port 22**, although administrators can configure it to use another port.

Once authenticated, the user can interact with the remote system according to the permissions of their account.

The security of that authentication process therefore matters a lot.

---

## 2. What Is an SSH Brute-Force Attack?

A brute-force attack attempts to gain access by repeatedly trying authentication credentials.

A basic attack might look like:

```text
admin     : password123
admin     : admin123
admin     : qwerty
admin     : letmein
root      : password
root      : 123456
...
```

A more realistic automated attack can test large lists of usernames and passwords.

The important point is that the attacker does not necessarily know which credentials are valid beforehand.

They are testing possibilities.

There are several related approaches:

- **Brute force:** systematically trying many password combinations.
- **Dictionary attack:** trying passwords from a prepared wordlist.
- **Password spraying:** trying one common password against many accounts to avoid triggering account-specific protections.
- **Credential stuffing:** trying username/password pairs obtained from previous breaches.

These techniques are related, but they are not identical.

---

## 3. Step 1 — Finding an SSH Service

Before attempting authentication, an attacker needs to find systems exposing SSH.

For example, a network scan may reveal:

```text
PORT   STATE SERVICE
22/tcp open  ssh
```

That does not mean the system has been compromised.

It simply means that an SSH service is reachable and responding.

From a defender's perspective, this is the first important distinction:

> **An exposed service is not automatically a compromised service.**

Exposure increases the attack surface, however, especially when the service is accessible from the public Internet.

---

## 4. Step 2 — Establishing the Connection

The attacker connects to the SSH service.

At a high level, the client and server negotiate the parameters required for an encrypted SSH session.

The server identifies itself and the SSH protocol establishes the secure communication channel.

Only after this does the attacker reach the authentication stage.

This is important because people sometimes imagine a brute-force attack as simply sending passwords directly to port 22.

There is more happening underneath.

---

## 5. Step 3 — Repeated Authentication Attempts

The attacker then starts testing credentials.

A simplified sequence looks like this:

```text
                SSH Server
                    |
        +-----------+-----------+
        |           |           |
     Attempt 1   Attempt 2   Attempt 3
        |           |           |
     Failed      Failed      Failed
                                |
                         Attempt N
                                |
                         Successful?
                          /                                No          Yes
                        |            |
                     Continue    Access gained
```

If the password is incorrect, the server rejects the authentication attempt.

The attacker can then try again.

Automated tools make this process much faster than manually attempting passwords.

---

## 6. What Does This Look Like in Logs?

This is where the attack becomes particularly interesting for defenders.

On Linux systems, failed SSH authentication attempts are commonly recorded by the system's logging infrastructure.

For example, Red Hat documents entries similar to:

```text
sshd: Failed password for illegal user admin from 172.16.59.10
```

Repeated failures from the same source can be a strong indicator that someone is attempting to guess credentials. citeturn0search11

A defender might therefore see something like:

```text
Failed password for invalid user admin from 203.0.113.50
Failed password for invalid user admin from 203.0.113.50
Failed password for root from 203.0.113.50
Failed password for root from 203.0.113.50
Failed password for test from 203.0.113.50
```

One failed login is not necessarily suspicious.

Hundreds of failed attempts against multiple usernames from the same source are a very different story.

---

## 7. When Does Brute Force Become Dangerous?

The attack itself does not guarantee successful access.

The real danger comes when the attacker eventually discovers valid credentials.

For example:

```text
Attacker
   |
   | username: admin
   | password: ********
   v
SSH Server
   |
   | Authentication successful
   v
Remote Shell
```

At that point, the problem is no longer simply an authentication attack.

The attacker may now have an initial foothold on the system.

What they can do next depends on the privileges of the compromised account and the security controls surrounding the system.

This is where other attack techniques can become relevant, including:

- Privilege escalation
- Persistence
- Credential harvesting
- Lateral movement
- Data theft
- Command execution

So an SSH brute-force attack is often better understood as a possible **initial-access technique**, rather than the complete attack.

---

## 8. How Can Defenders Detect It?

A basic detection strategy is to monitor authentication failures.

Useful indicators include:

### High number of failures

A large number of failed SSH logins in a short period can indicate automated password guessing.

### Multiple usernames from one source

For example:

```text
root
admin
test
ubuntu
guest
administrator
```

Trying many common usernames can indicate automated enumeration.

### Repeated activity over time

An attacker may not always perform thousands of attempts immediately.

Some campaigns deliberately slow down their attempts to make detection harder.

### Successful login after many failures

This deserves particular attention:

```text
Failed
Failed
Failed
Failed
SUCCESS
```

A successful authentication following a large number of failures should trigger investigation, especially if the source or account is unusual.

---

## 9. How Can You Defend Against SSH Brute Force?

There is no single magic setting that solves the problem.

A layered approach is much stronger.

### Use SSH keys instead of passwords

Public-key authentication is generally preferable to password-based authentication for administrative SSH access.

### Disable password authentication where appropriate

If your environment supports key-based authentication and does not require passwords, disabling password authentication can remove an entire class of password-guessing attacks.

### Disable direct root login

Administrators can use individual accounts and controlled privilege escalation instead of allowing direct root authentication.

### Restrict SSH exposure

If SSH only needs to be accessible from a corporate network, VPN, bastion host, or specific administrative IP ranges, do not expose it unnecessarily to the entire Internet.

### Use rate limiting and blocking controls

Tools such as Fail2ban can react to repeated authentication failures and temporarily block abusive sources.

### Monitor authentication logs

Detection is essential.

Even a well-configured server should be monitored so that suspicious authentication activity can be investigated.

---

## 10. What Should a SOC Analyst Look For?

Imagine you are investigating an alert for unusual SSH activity.

A useful investigation could include:

```text
1. Identify the source IP
        ↓
2. Count failed authentication attempts
        ↓
3. Identify targeted usernames
        ↓
4. Check whether authentication eventually succeeded
        ↓
5. Identify the successful account
        ↓
6. Check the source location/reputation
        ↓
7. Review commands and activity after login
        ↓
8. Determine whether the account was compromised
```

This is where SSH brute force becomes more than a simple "hacking technique."

It becomes a **detection and investigation problem**.

A SOC analyst is not only asking:

> "Did someone try to log in?"

They are asking:

> "Was the activity malicious, did it succeed, and what happened afterward?"

---

## 11. The Bigger Lesson

SSH brute-force attacks demonstrate an important cybersecurity principle:

**An attack does not need to be sophisticated to be dangerous.**

SSH itself can be securely designed and encrypted, yet weak credentials, excessive exposure, poor monitoring, or weak access controls can still create opportunities for attackers.

Security therefore cannot depend on one control.

You want multiple layers:

```text
Strong authentication
        +
Limited exposure
        +
Least privilege
        +
Rate limiting
        +
Logging
        +
Monitoring
        +
Incident response
```

If one layer fails, the others should make exploitation harder and help detect what happened.

---

## Conclusion

An SSH brute-force attack is essentially a repeated authentication attack against a remote service.

The technical process is straightforward:

**Discover SSH → Connect → Attempt credentials → Observe responses → Repeat → Gain access if successful**

The more interesting part for a cybersecurity professional is what happens around that process.

Can you recognize the attack in logs?

Can you distinguish normal failed logins from automated activity?

Can you determine whether an authentication attempt succeeded?

And if it did, can you investigate what happened next?

Understanding those questions takes you beyond simply knowing what a brute-force attack is — and toward thinking like a security analyst.

---

## References

- Red Hat — *How to secure the SSHD daemon*
- OpenSSH documentation
- MITRE ATT&CK — Valid Accounts / Brute Force techniques

*This article is intended for educational and defensive cybersecurity purposes. Any testing should be performed only on systems you own or are explicitly authorized to assess.*
