---
{"dg-publish":true,"permalink":"/40-ctf-and-events/breach-lab/","tags":["ctf","securityBrigade","BreachLab"]}
---

# Executive Summary of the Lab




![BreachLab-HomepageLook-20260307.png](/img/user/40-CTF-&-Events/_attachments/BreachLab-HomepageLook-20260307.png)

---
# Phase 1: Reconnaissance

1. Target Type: Web Application
2. Target Name: BreachLab
3. Target url:
   ``` URL
   http://breachlab.securitybrigade.com/
   ```
4. Target IP: 172.105.120.202

The application utilizes the insecure HTTP (Cleartext) protocol instead of the industry-standard HTTPS (TLS/SSL), exposing sensitive user data to potential interception.

> [!note] Logic Train
> While reviewing the source code, I identified a custom JavaScript implementation for DOM manipulation. This indicates a potential entry point for **T1059.007 - JavaScript** execution via DOM-based XSS if user input is handled by 'sink' functions like `innerHTML` without sanitization.
### Methodology: Network Discovery
Network discovery is the automated process of identifying, mapping, and monitoring devices (computers, routers, printers, IoT) on a network to maintain an accurate, up-to-date inventory and improve security. It uses protocols like SNMP, LLDP, and ICMP (ping) to map network topology and detect unauthorized or new devices.
#### Nmap
Nmap (Network Mapper) was utilized to perform active reconnaissance against the target IP `172.105.120.202`. The objective was to identify open ports, active services, and potential version fingerprints that could lead to known vulnerabilities.

Scanning the Target using following command:
``` Zsh
nmap -sV 172.105.120.202
```

The Following is the proof of the same

![BreachLab-NmapScanning-20260307.png](/img/user/40-CTF-&-Events/_attachments/BreachLab-NmapScanning-20260307.png)

We get information as follows:

| Port       | State | Service | Version                        |
| :--------- | :---- | :------ | :----------------------------- |
| **80/tcp** | Open  | http    | Apache httpd 2.4.52 ((Ubuntu)) |

> [!note] Logic Train
> Using the Nmap scan we have identified the server service use for the target system that is Apache httpd 2.4.52 ((Ubuntu)). A Quick Google search on any existing CVE for this server service we get the Vulnerabilities for that specific server service and for that version of the server. 

#### CVE for Apache httpd 2.4.52 ((Ubuntu)) [^1]

| **Severity**     | **CVE ID**         | **Vulnerability Name**       | **Impact**                                              | **Fixed In** |
| ---------------- | ------------------ | ---------------------------- | ------------------------------------------------------- | ------------ |
| 🔴 **Important** | **CVE-2022-22720** | HTTP Request Smuggling       | Request body discard failure; allows smuggling attacks. | 2.4.53       |
| 🔴 **Important** | **CVE-2022-23943** | mod_sed: Out-of-bounds Write | Heap memory overwrite with attacker-provided data.      | 2.4.53       |
| 🟡**Moderate**   | **CVE-2022-22719** | mod_lua: Uninitialized Value | Read random memory; potential process crash.            | 2.4.53       |
| 🔵 **Low**       | **CVE-2022-22721** | LimitXMLRequestBody Overflow | Integer overflow on 32-bit systems (>350MB).            | 2.4.53       |

> [!attention]
> Since the Latest version of the server is Apache httpd 2.4.66 Released which was released on 04th December 2025. with the CVE found for 2.4.52 Being Fixed in version 2.4.53. This is the case of Vulnerable and Outdated Components from OWASP Top 10 2021[^2].

> [!success] Prevention Checklist: Vulnerable & Outdated Components> 
> * **Minimize Attack Surface:** Remove unused dependencies, features, files, and documentation.
> * **Continuous Inventory:** Track all client/server-side components and versions (SCA tools like *OWASP Dependency Check* or *retire.js*).
> * **Monitor Vulnerabilities:** Automate CVE/NVD monitoring and subscribe to security alerts for your specific tech stack.
> * **Secure Sourcing:** Only pull from official sources via secure links; prioritize **signed packages**.
> * **Patch Lifecycle:** Retire unmaintained libraries. If a patch isn't available, deploy **virtual patching** (WAF) to mitigate risks.
> * **Governance:** Establish an ongoing plan for triaging and applying updates throughout the application's lifetime.
### Methodology: Directory Fuzzing
Directory fuzzing is a web enumeration technique that uses automated tools to discover hidden files, directories, or subdomains on a web server by testing thousands of potential paths from a wordlist. It acts like brute-forcing, checking for HTTP status codes (e.g., 200 OK, 301 Redirect) to map out the application's structure. The following is Table of Status Code along with meaning.

| Status Code       | Meaning                              | What to do next                                                 |
| :---------------- | :----------------------------------- | :-------------------------------------------------------------- |
| **200 OK**        | File/Folder exists and is reachable. | Open it in your browser immediately.                            |
| **301/302**       | Redirect.                            | See where it's pointing; it might lead to a login page.         |
| **403 Forbidden** | It exists, but access is denied.     | Try to "bypass" it or look for sub-files inside it.             |
| **405**           | Method Not Allowed.                  | It exists, but maybe it requires a POST request instead of GET. |
> [!note] Note on Service Stability
> To strictly adhere to the Rules of Engagement (RoE) regarding resource preservation, the thread count has been throttled to `-t 40`. This mitigation ensures discovery remains comprehensive without risking a Denial-of-Service (DoS) condition on the target infrastructure.  

For our Test we will be using the following tools:
#### 1. Gobuster
Enumerating the target using following command:

```Zsh
gobuster dir -u 172.105.120.202 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 40 
```

![BreachLab-Gobuster-1-20260307.png](/img/user/40-CTF-&-Events/_attachments/BreachLab-Gobuster-1-20260307.png)

![BreachLab-Gobuster-2-20260307.png](/img/user/40-CTF-&-Events/_attachments/BreachLab-Gobuster-2-20260307.png)

![BreachLab-Gobuster-3-20260307.png](/img/user/40-CTF-&-Events/_attachments/BreachLab-Gobuster-3-20260307.png)





#### 2. Dirsearch
Enumerating the target using following command:

```Zsh
dirsearch -u http://breachlab.securitybrigade.com/ -t 40
```

![BreachLab-Dirsearch-1-20260307.png](/img/user/40-CTF-&-Events/_attachments/BreachLab-Dirsearch-1-20260307.png)

![BreachLab-Dirsearch-2-20260307.png](/img/user/40-CTF-&-Events/_attachments/BreachLab-Dirsearch-2-20260307.png)

After enumeration we have found the `database.sql` directory which is following:

```SQL
-- FooPhones 2.0 Sample Database with Foreign Key Handling

SET FOREIGN_KEY_CHECKS = 0;

DROP TABLE IF EXISTS purchases;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS users;

SET FOREIGN_KEY_CHECKS = 1;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    email VARCHAR(100),
    wallet_balance DECIMAL(10,2) DEFAULT 1000.00,
    is_admin TINYINT(1) DEFAULT 0
);

INSERT INTO users (username, password, email, is_admin) VALUES
('admin', 'admin123', 'admin@foophones.test', 1),
('alice', 'alice123', 'alice@foophones.test', 0),
('bob', 'bob123', 'bob@foophones.test', 0);

CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    price DECIMAL(10,2),
    stock INT DEFAULT 10
);

INSERT INTO products (name, description, price, stock) VALUES
('FooPhone X', 'Latest flagship with insecure features', 499.99, 5),
('FooPhone Mini', 'Smaller and even less secure', 299.99, 10),
('FooPhone Pro', 'With all the bugs unlocked', 799.99, 2);

CREATE TABLE purchases (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    product_id INT,
    purchase_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE
);

```



The Credential Used are as follows

Username: `<script>alert(999999)</script>`
Email: `test2@tmail.edu.rs`
Password: Test@123


![BreachLab-LoginUsingDOMXSS-20260307.png](/img/user/40-CTF-&-Events/_attachments/BreachLab-LoginUsingDOMXSS-20260307.png)



















---
[^1]: Derived from [Apache HTTP Server 2.4 vulnerabilities (Fixed in 2.4.53)](https://httpd.apache.org/security/vulnerabilities_24.html#:~:text=Fixed%20in%20Apache%20HTTP%20Server%202.4.53)

[^2]:Derived from [OWASP Top 10 2021 Vulnerable and Outdated System)](https://owasp.org/Top10/2021/A06_2021-Vulnerable_and_Outdated_Components/)