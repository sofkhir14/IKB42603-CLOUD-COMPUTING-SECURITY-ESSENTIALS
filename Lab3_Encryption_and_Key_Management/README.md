# Lab 3: Encryption and Key Management 
Course: IKB42603 Cloud Computing Security Essentials

Lab: Lab 3

Topic: At-rest & in-transit encryption, envelope encryption, cryptographic erasure, LocalStack KMS, and hashing integrity

Environment: OpenSSL, Docker, AWS CLI v2, LocalStack KMS

Student Name: Nurin Sofea Binti Mohammad Khir

# Objective
The objective of this lab is to evaluate and implement data protection mechanisms across cloud environments. This includes performing symmetric and asymmetric encryption, securing data in transit using TLS, implementing scalable envelope encryption, enforcing per-tenant cryptographic erasure using a Key Management Service (KMS), building tamper-evident, and hash-chained log structures to preserve data integrity.

# Learning Outcomes
By completing this lab, i will be able to:
- Encrypt and decrypt sensitive data at rest using symmetric (AES-256) and asymmetric (RSA-2048) cryptography.  

- Protect data in transit using Transport Layer Security (TLS) and observe encrypted HTTP traffic.

- Implement envelope encryption using LocalStack KMS to safely manage local Data Encryption Keys (DEKs).
  
- Apply per-tenant customer master keys (CMKs) and execute cryptographic erasure to render cloud data provably unrecoverable.

- Verify data integrity using cryptographic hashing (SHA-256) and construct a tamper-evident hash chain for logging.

# Environment
- Operating System / Terminal: Linux

- Cryptography Tools: OpenSSL

- Containerization: Docker Engine

- Cloud Emulation & CLI: AWS CLI v2, LocalStack (KMS service)

# Task 1: Symmetric Encryption (Data at Rest)
A text file was created and encrypted using AES-256. The encrypted file was checked to prove it was unreadable, then it was decrypted back to the original text.

Command Summary: 

```
echo 'Patient: Ahmad, Diagnosis: confidential' > record.txt
openssl enc -aes-256-cbc -pbkdf2 -salt -in record.txt -out record.enc
cat record.enc
openssl enc -d -aes-256-cbc -pbkdf2 -in record.enc -out record.dec.txt
diff record.txt record.dec.txt && echo 'MATCH: decryption successful
``` 
Result: 

The diff command returned MATCH: decryption successful, proving that the symmetric passphrase successfully restored the plaintext data.

Evidence:

<img width="541" height="42" alt="Screenshot 2026-08-19 173507" src="https://github.com/user-attachments/assets/b6d698f4-aee1-4368-8ba5-363c03110cfc" />

<img width="484" height="66" alt="Screenshot 2026-08-19 173722" src="https://github.com/user-attachments/assets/e696ec76-99fb-4194-b817-312e05fea1a4" />

<img width="624" height="88" alt="Screenshot 2026-08-19 173801" src="https://github.com/user-attachments/assets/0441d41e-6aa6-4ad6-86ff-b29f92a39e64" />

# Task 2: Asymmetric Encryption & Digital Signatures
An RSA key pair was created. The public key was used to encrypt the file, and the private key was used to decrypt it. The private key then signed the file, and the public key verified the signature.

Command Summary:

```
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem

openssl pkeyutl -encrypt -pubin -inkey public.pem -in record.txt -out record.rsa
openssl pkeyutl -decrypt -inkey private.pem -in record.rsa -out record.rsa.txt

openssl dgst -sha256 -sign private.pem -out record.sig record.txt
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```
Result:

The verification command returned Verified OK. This proves the file came from the true owner and was not changed by anyone else.

Evidence:

<img width="621" height="77" alt="Screenshot 2026-08-19 174206" src="https://github.com/user-attachments/assets/e3edff38-f715-48ba-99a4-483c253c89a4" />


# Task 3: Encryption in Transit (TLS)
A self-signed TLS certificate was generated and attached into an Nginx Docker container running on port 8443.

Command Summary:

```
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem \
  -days 7 -nodes -subj '/CN=localhost'

docker run --rm -d --name tls -p 8443:443 \
  -v $(pwd)/cert.pem:/etc/nginx/cert.pem \
  -v $(pwd)/key.pem:/etc/nginx/key.pem \
  -v $(pwd)/record.txt:/usr/share/nginx/html/record.txt nginx

curl -k https://localhost:8443/record.txt
```
Result:

The file text was safely fetched over HTTPS. Anyone capturing network traffic cannot read the data because it travels through an encrypted tunnel.

Evidence:

<img width="628" height="590" alt="Screenshot 2026-08-19 175830" src="https://github.com/user-attachments/assets/9e677560-c7f2-478d-8710-c517fb5b1a63" />

<img width="525" height="174" alt="Screenshot 2026-08-19 180154" src="https://github.com/user-attachments/assets/2d3556f7-17f9-4483-afd9-23c053ad090e" />

<img width="373" height="70" alt="Screenshot 2026-08-19 180216" src="https://github.com/user-attachments/assets/dbcb3323-3212-44d3-8c7c-7532dfddd393" />

# Task 4: Create and Use a KMS Master Key
LocalStack KMS was invoked via AWS CLI v2 to create a Customer Master Key (CMK) for Tenant A, which was used to directly encrypt a base64-encoded secret.

Command Summary:

```
EP='--endpoint-url=http://localhost:4566'

aws $EP kms create-key --description 'CCSE tenant-A master key'
KEY_A=<PASTE_KEYID>

aws $EP kms encrypt --key-id $KEY_A --plaintext "$(echo -n 'hello' | base64)" \
  --query CiphertextBlob --output text
```
Result:

KMS created an encrypted text output managed by KEY (A). This shows that encryption happened directly inside KMS.

Evidence:

<img width="629" height="97" alt="Screenshot 2026-08-19 184340" src="https://github.com/user-attachments/assets/c6d0f08c-9458-425d-8150-98deeead57ae" />


# Task 5: Envelope Encryption
A data key was requested from KMS. The local file was encrypted using the readable data key, and then the readable key was deleted from the computer, leaving only the wrapped key (datakey.enc).

Command Summary:

```
aws $EP kms generate-data-key --key-id $KEY_A --key-spec AES_256 \
  --query '[Plaintext, CiphertextBlob]' --output text 

awk '{print $1}' keys.txt > datakey.b64
awk '{print $2}' keys.txt > datakey.enc

base64 -d datakey.b64 > datakey.bin
openssl enc -aes-256-cbc -pbkdf2 -in record.txt -out record.env.enc -pass file:./datakey.bin

rm datakey.bin datakey.b64
echo 'Only the KMS-wrapped data key (datakey.enc) remains.' 
```
Result:

The unencrypted key was deleted from disk. The file can now only be opened if KMS unwraps datakey.enc using master key KEY (A).

Evidence:

<img width="630" height="112" alt="Screenshot 2026-08-19 184637" src="https://github.com/user-attachments/assets/1ef7d917-e4c7-4180-984d-24814dc20357" />

<img width="619" height="59" alt="Screenshot 2026-08-19 185501" src="https://github.com/user-attachments/assets/c433b8ee-2f05-469b-8af5-18c284f6887e" />

<img width="362" height="46" alt="Screenshot 2026-08-19 185521" src="https://github.com/user-attachments/assets/e3c0f3a8-0464-46dd-98d1-13441cd14c4e" />


<img width="354" height="49" alt="Screenshot 2026-08-19 185538" src="https://github.com/user-attachments/assets/60c19c4b-d6df-420f-a8e8-f7b23fa0c68a" />

<img width="324" height="50" alt="Screenshot 2026-08-19 185610" src="https://github.com/user-attachments/assets/4d0e7db7-265a-4d79-b5a9-e532ded83133" />

<img width="623" height="60" alt="Screenshot 2026-08-19 185634" src="https://github.com/user-attachments/assets/1a22b421-057c-464e-9b56-3b9ac6d3b182" />


<img width="260" height="50" alt="Screenshot 2026-08-19 185747" src="https://github.com/user-attachments/assets/6799cfee-9c38-41c9-9f04-e146a3b85504" />

<img width="515" height="67" alt="Screenshot 2026-08-19 185807" src="https://github.com/user-attachments/assets/1d9dd2bc-1564-4282-80a9-e7710fe4835d" />

# Task 6: Per-Tenant Keys & Cryptographic Erasure
A separate master key was created for Tenant B. Tenant A's master key was disabled to simulate key deletion, and a decryption test was attempted

Command Summary:

```
aws $EP kms create-key --description 'CCSE tenant-B master key'
KEY_B=<PASTE_KEYID>

aws $EP kms schedule-key-deletion --key-id $KEY_A --pending-window-in-days 7
aws $EP kms disable-key --key-id $KEY_A

aws $EP kms decrypt --ciphertext-blob fileb://datakey.enc 2>&1 | head -3
```
Result:

The AWS CLI returned a DisabledException failure. Without KEY (A), the encrypted file is locked forever and cannot be recovered by anyone.

Evidence:

<img width="621" height="107" alt="Screenshot 2026-08-19 191536" src="https://github.com/user-attachments/assets/19fdf141-dbba-44c3-8bf9-def3e503d8eb" />


# Task 7: Integrity & Tamper-Evidence
SHA-256 hashes were calculated to check data integrity. The file was modified to test hash changes, and a connected hash chain was built.

Command Summary:

```
sha256sum record.txt

cp record.txt tampered.txt; echo 'x' >> tampered.txt
sha256sum record.txt tampered.txt

PREV=0
for line in 'login ok' 'file read' 'export data'; do \
  PREV=$(echo -n "$PREV$line" | sha256sum | cut -d' ' -f1); \
  echo "$line | $PREV"; \
done
```
Result:

Adding just one letter completely changed the SHA-256 hash. The hash chain successfully linked each log entry to the previous one, making changes easy to spot.

Evidence:

<img width="615" height="65" alt="Screenshot 2026-08-19 191643" src="https://github.com/user-attachments/assets/1b88e5e6-1e3b-450c-a83e-d7955bd38cae" />

<img width="627" height="141" alt="Screenshot 2026-08-19 191743" src="https://github.com/user-attachments/assets/ce16d9c9-52cd-4b65-84e6-18aca85cd26e" />

<img width="628" height="173" alt="Screenshot 2026-08-19 192153" src="https://github.com/user-attachments/assets/2a081716-1ba8-480c-966b-ac8d2f437291" />

# Commands Used

```
# OpenSSL Commands
openssl enc -aes-256-cbc -pbkdf2 -salt -in <in> -out <out>
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
openssl pkeyutl -encrypt -pubin -inkey public.pem -in <in> -out <out>
openssl dgst -sha256 -sign private.pem -out <sig> <file>
openssl dgst -sha256 -verify public.pem -signature <sig> <file>

# AWS KMS Commands (LocalStack Endpoint)
aws --endpoint-url=http://localhost:4566 kms create-key --description "<desc>"
aws --endpoint-url=http://localhost:4566 kms generate-data-key --key-id <id> --key-spec AES_256
aws --endpoint-url=http://localhost:4566 kms disable-key --key-id <id>
aws --endpoint-url=http://localhost:4566 kms decrypt --ciphertext-blob fileb://<wrapped-key>
aws --endpoint-url=http://localhost:4566 kms list-keys

# Docker & Integrity Commands
docker run --rm -d --name tls -p 8443:443 -v $(pwd)/cert.pem:/etc/nginx/cert.pem nginx
sha256sum <file>
```

# Short-Answer Questions
Q1: Compare symmetric and asymmetric encryption: speed, key distribution, and typical use.

- Symmetric Encryption: Uses one key to encrypt and decrypt. It is very fast, but sending the key safely to another person is hard. It is best for encrypting large files stored on a disk.

- Asymmetric Encryption: Uses two keys (a public key to encrypt and a private key to decrypt). It is slower, but easy to share keys safely. It is best for digital signatures and setting up secure connections like HTTPS/TLS.

Q2: Why is key management described as the weakest link, not the algorithm?

- Modern encryption algorithms like AES-256 are practically impossible to break. Security systems usually fail because human errors occur such as saving keys in public places, giving weak permissions, or losing keys. If an attacker steals the key, the encryption cannot protect the data.

Q3: Explain envelope encryption and why only the master key needs hardware-grade protection.

- Envelope encryption uses a local Data Key to encrypt a large file, and then uses a Master Key in KMS to lock (wrap) the Data Key. This keeps large files from being sent back and forth over the network to KMS. Only the Master Key needs hardware protection (HSM) because as long as the Master Key is safe, all wrapped Data Keys remain safe.

Q4: How does cryptographic erasure achieve provable deletion where overwriting cannot (in the cloud)?

- In distributed cloud storage, physical media is abstracted, replicated, and snapshot-managed, making block-level overwriting impossible to guarantee. Cryptographic erasure deletes or disables the Master Key used to encrypt that data. Without the key, the encrypted data sitting on the cloud drives becomes unreadable trash, proving it is destroyed.

Q5: How does a hash chain make a log tamper-evident?

- A hash chain adds the hash value of the previous log entry into the calculation of the next log entry. If an attacker changes or deletes an old log, every new hash in the chain breaks. This makes any unauthorized change immediately visible during an audit.

# Challenges Encountered
1. Splitting the AWS KMS Data Key Output:

   - Challenge: The generate-data-key command output put both the plaintext key and wrapped key on one line, which broke the base64 decode step.
  
   - Resolution: Used awk '{print $1}' and awk '{print $2}' commands to separate the keys into two clean files before running encryption.

2. Docker TLS Permission Errors:

   - Challenge: The private key created by OpenSSL had restricted file permissions, stopping Nginx from reading it.
  
   - Resolution: Added the -nodes flag to OpenSSL and mapped the working directory straight into Nginx using Docker volume mounts.


# Lessons Learned
- Key management is the most important part of cloud security. Encryption algorithms are useless if keys are lost or leaked.

- Envelope encryption makes encrypting large files fast while keeping root key control inside KMS.

- Cryptographic erasure gives cloud users a reliable way to permanently delete data by destroying key access.

# Security Best-Practices Checklist
[/] Data encrypted at rest (AES) and decryption verified.

[/] Asymmetric keys used correctly (encrypt with public, sign with private).

[/] Data protected in transit with TLS. 

[/] Envelope encryption used; plaintext data key purged from disk.

[/] Per-tenant keys used; cryptographic erasure demonstrated. 

[/] Integrity verified with hashing and a hash chain. 

# Cleanup
The environment can be removed after completing the lab and saved all the evidence needed. 

<img width="630" height="111" alt="Screenshot 2026-08-21 172258" src="https://github.com/user-attachments/assets/08bd8eed-46c0-418e-b535-8e8fcff6266e" />

<img width="216" height="56" alt="Screenshot 2026-08-21 172349" src="https://github.com/user-attachments/assets/df567f44-0740-4923-abd7-9411e11a8467" />


# Verification Commands
```
aws --endpoint-url=http://localhost:4566 kms list-keys
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```
Expected Verification Result:

- LocalStack KMS lists active keys KEY (B) alongside disabled keys KEY (A).

- OpenSSL outputs Verified OK

Evidence:

<img width="624" height="266" alt="Screenshot 2026-08-19 192701" src="https://github.com/user-attachments/assets/4fc7fed2-ab70-4255-a939-98a088ddfe2d" />

<img width="618" height="79" alt="Screenshot 2026-08-19 192802" src="https://github.com/user-attachments/assets/624fb6a6-17eb-4464-8b40-dfb0df394aea" />

# Conclusion
This lab showed how to protect data in the cloud using encryption, key management, and hashing. Symmetric encryption (AES) is great for securing large files, while asymmetric encryption (RSA) and TLS protect data during transit. Envelope encryption with AWS KMS makes key handling faster and more secure. Finally, deleting master keys allows per-tenant data to be permanently erased, and hash chains make it easy to spot if log files have been altered.

# References
1. Course Lecture - Week 4 (Data Protection); Week 9 (Key Management Patterns).
2. OpenSSL Project Documentation – https://www.openssl.org/docs/
3. AWS KMS Developer Guide (Envelope Encryption Concepts) – https://docs.aws.amazon.com/kms/
4. Cloud Security Alliance (CSA) Security Guidance v5 – Data Security & Encryption. 
