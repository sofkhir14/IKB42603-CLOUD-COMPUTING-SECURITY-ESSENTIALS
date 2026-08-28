# Lab 4: Access Control and Network Security

Course: IKB42603 Cloud Computing Security Essentials 

Lab: Lab 4 

Topic: Authentication vs. Authorization, MFA, RBAC, Network Segmentation, Firewall Rules, and Container Hardening

Environment: Docker, Kubernetes (kind), kubectl, oathtool, Trivy Scanner

Name: Nurin Sofea Binti Mohammad Khir

# Objective
The objective of this lab is to configure access control and network security across cloud containers and Kubernetes. Learn how to set up password authentication, add Multi-Factor Authentication (MFA), restrict developer roles using Role-Based Access Control (RBAC), isolate database networks using segmentation, set default-deny firewall rules, and harden container security profiles.

# Learning Outcomes
By completing this lab, I will be able to:

- Distinguish between authentication and authorization.

- Implement a time-based one-time password (TOTP / MFA) second factor.

- Configure Kubernetes RBAC to enforce least privilege access.

- Segment Docker networks so web services cannot directly reach back-end databases.

- Apply host-level default-deny firewall policies using iptables

- Harden Docker containers with non-root accounts, dropped capabilities, and read-only filesystems, and scan images for vulnerabilities.

# Environment
- Operating System / Terminal: Linux

- Containerization & Orchestration: Docker Engine, kind (Kubernetes in Docker)

- Command Line Tools: kubectl, oathtool (TOTP), iptables, curl

- Security Scanner: Aquasecurity Trivy

# Task 1: Authentication & Authorization (Password-Protected Service)
A password file was created using htpasswd and mounted into an Nginx container. Unauthenticated requests were tested alongside valid credential requests.

Command Summary: 
```
docker run --rm httpd:alpine htpasswd -nbB student 'P@ssword!' > htpasswd.txt

cat > default.conf <<'EOF'
server { 
    listen 80;
    location / { 
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;
        return 200 'Authenticated OK\n'; 
    } 
}
EOF

docker run --rm -d --name authsvc -p 8080:80 \
  -v "$(pwd)/default.conf:/etc/nginx/conf.d/default.conf" \
  -v "$(pwd)/htpasswd.txt:/etc/nginx/.htpasswd" nginx

curl -s -o /dev/null -w 'no-creds: %{http_code}\n' http://localhost:8080
curl -s -u student:'P@ssword!' http://localhost:8080
```
Result:

Sending no credentials returned no-creds: 401, while sending correct credentials returned Authenticated OK with HTTP status 200. This proves that authentication blocks unverified users.

Evidence:


<img width="621" height="67" alt="Screenshot 2026-08-28 213027" src="https://github.com/user-attachments/assets/20a6aadc-f53a-4395-8bb1-798b2acdd87b" />


<img width="460" height="58" alt="Screenshot 2026-08-28 213035" src="https://github.com/user-attachments/assets/d6268893-9fa5-4545-a0b3-c7166362eaaa" />


# Task 2: Add a Second Factor (MFA / TOTP)
A base32 secret key was generated to create a time-based one-time password (TOTP). The generated code was then verified using oathtool.

Command Summary:
```
SECRET=$(head -c20 /dev/urandom | base32)
echo "Enrol this secret in an authenticator app: $SECRET"
oathtool --totp -b "$SECRET"

# Simulate user entering code
CODE=$(oathtool --totp -b "$SECRET")
[ "$CODE" = "$(oathtool --totp -b "$SECRET")" ] && echo 'MFA OK' || echo 'MFA FAILED'
```
Result:

The terminal printed MFA OK. This shows that two-factor authentication successfully validates "something you have" alongside "something you know".

Evidence:

<img width="627" height="315" alt="Screenshot 2026-08-28 214610" src="https://github.com/user-attachments/assets/6b84847e-ca0f-4585-a45e-43f62b9b6b99" />

# Task 3: Authorization: RBAC Roles
A Kubernetes cluster was created using kind. A developer ServiceAccount was created and limited to only viewing pods in the app namespace using RBAC.

Command Summary:
```
kind create cluster --name ccse-lab4
kubectl create namespace app
kubectl create serviceaccount dev -n app

kubectl create role dev-role -n app --verb=get,list --resource=pods
kubectl create rolebinding dev-rb -n app --role=dev-role --serviceaccount=app:dev

SA=system:serviceaccount:app:dev
kubectl auth can-i list pods -n app --as=$SA
kubectl auth can-i create deploy -n app --as=$SA
kubectl auth can-i delete pods -n app --as=$SA
```
Result:

The can-i list pods check returned yes, while create deploy and delete pods returned no. This proves that authorization restricts actions even after a user is authenticated.

Evidence:

<img width="431" height="187" alt="Screenshot 2026-08-28 215138" src="https://github.com/user-attachments/assets/fa21e9cf-962c-4339-909a-5a27ecab7072" />

# Task 4: Network Segmentation (Three-Tier)
Two Docker networks (frontend-net and backend-net) were created. A database container was placed on backend-net, a web container on frontend-net, and an app container on both.

Command Summary:
```
docker network create frontend-net
docker network create backend-net

docker run -d --name db --network backend-net redis:alpine
docker run -d --name app --network backend-net nginx
docker network connect frontend-net app
docker run -d --name web --network frontend-net nginx

# Test web -> db (Should fail)
docker exec web sh -c 'apk add -q curl; curl -m 3 db:6379 || echo BLOCKED'

# Test app -> db (Should work)
docker exec app sh -c 'apk add -q curl; nc -z -w3 db 6379 && echo REACHABLE'
```
Result:

The connection from web to db output BLOCKED, while app to db output REACHABLE. This proves network isolation stops front-end web servers from accessing database servers directly.

Evidence:

<img width="628" height="90" alt="Screenshot 2026-08-28 215624" src="https://github.com/user-attachments/assets/4470f198-cafc-43e2-bb9e-ade17ea014c7" />


<img width="627" height="112" alt="Screenshot 2026-08-28 215640" src="https://github.com/user-attachments/assets/57130584-b50c-4452-8f0d-694ebb07947f" />


# Task 5: Firewall Rules (Default-Deny)
A default-deny firewall rule was configured using iptables inside a container to drop all incoming traffic except HTTPS on port 443.

Command Summary:
```
docker run --rm --cap-add=NET_ADMIN alpine sh -c '
apk add -q iptables; 
iptables -P INPUT DROP; 
iptables -A INPUT -p tcp --dport 443 -j ACCEPT; 
iptables -A INPUT -i lo -j ACCEPT; 
iptables -L INPUT -n'
```
Result:

The iptables -L command showed Chain INPUT (policy DROP) with an explicit ACCEPT rule for port 443. This models cloud security groups where all incoming traffic is blocked by default.

Evidence:

<img width="628" height="145" alt="Screenshot 2026-08-28 220211" src="https://github.com/user-attachments/assets/a46c1fd2-64e2-4003-a946-ba26666e1e2b" />

# Task 6: Container / Host Hardening
An unprivileged Nginx container was run with non-root privileges, a read-only filesystem, dropped capabilities, and no new privileges. Its settings were inspected and its image was scanned for vulnerabilities using Trivy.

Command Summary:
```
docker run -d --name hardened \
  --user 1000:1000 \
  --read-only \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --tmpfs /tmp \
  nginxinc/nginx-unprivileged

docker inspect hardened --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}}'

docker run --rm aquasec/trivy image --severity HIGH,CRITICAL nginx:alpine | head -20
```
Result:

The docker inspect command printed User=1000:1000 ReadOnly=true. The Trivy scan generated a list of High and Critical security vulnerabilities found in the base image.

Evidence:


<img width="627" height="77" alt="Screenshot 2026-08-28 221022" src="https://github.com/user-attachments/assets/d8b09a80-6c87-4148-b303-217116b46bf1" />


<img width="626" height="377" alt="Screenshot 2026-08-28 221122" src="https://github.com/user-attachments/assets/d034d519-8b4a-4c06-aa7d-97f4702ea9fa" />



# Commands Used
```
# Authentication & MFA Commands
docker run --rm httpd:alpine htpasswd -nbB student 'P@ssword!'
oathtool --totp -b "$SECRET"

# Kubernetes RBAC Commands
kind create cluster --name ccse-lab4
kubectl create serviceaccount dev -n app
kubectl create role dev-role -n app --verb=get,list --resource=pods
kubectl create rolebinding dev-rb -n app --role=dev-role --serviceaccount=app:dev
kubectl auth can-i <action> <resource> -n app --as=system:serviceaccount:app:dev

# Network Segmentation & Firewall Commands
docker network create <network_name>
docker network connect <network_name> <container_name>
iptables -P INPUT DROP
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Container Hardening & Scanning Commands
docker run -d --read-only --cap-drop ALL --security-opt no-new-privileges nginxinc/nginx-unprivileged
docker inspect <container> --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}}'
docker run --rm aquasec/trivy image --severity HIGH,CRITICAL <image_name>
```

# Short-Answer Questions
Q1. Explain the difference between authentication and authorization using Tasks 1 and 3.

- Authentication (Task 1): Verifies who you are. Passing a valid username and password proved identity and returned HTTP 200.

- Authorization (Task 3): Determines what you are allowed to do. Even though the developer identity was authenticated, RBAC only allowed reading pods and blocked deleting pods or creating deployments.

Q2. Why is MFA so effective, and which attacks does it defeat?

- MFA requires two different authentication factors like a password and a phone TOTP code. It is extremely effective because an attacker who steals a user password still cannot log in without the physical device. It defeats phishing, credential stuffing, and brute-force password attacks.

Q3. How does network segmentation limit the damage of a compromised web server?

- Network segmentation places web servers and database servers on isolated network segments. If an attacker hacks the internet-facing web server, the isolated network prevents them from moving laterally to access or steal sensitive database records.

Q4. What does a default-deny firewall policy achieve, and how does it relate to cloud security groups?

- A default-deny policy blocks all incoming traffic by default, allowing only specifically configured ports (such as port 443 for HTTPS). Cloud security groups work the exact same way which is they deny all incoming traffic until an explicit rule is created to allow trusted traffic.

Q5. List the hardening measures you applied and the attack surface each one removes.

- Non-Root User (--user 1000:1000): Stops an attacker from getting full root access on the host system if the container is breached.

- Read-Only Filesystem (--read-only): Prevents attackers from downloading malware, modifying web files, or changing core system configurations.

- Drop Capabilities (--cap-drop ALL): Strips administrative Linux kernel privileges, preventing raw network manipulation or kernel exploits.

# Challenges Encountered

- No Major Technical Errors Encountered: All commands for cluster creation, network segmentation, firewall configuration, and container hardening executed as expected without technical failures.

# Lessons Learned
- Authentication alone is not enough. Strong cloud security requires strict authorization (RBAC) to limit actions.

- Multi-Factor Authentication (MFA) is one of the easiest and most effective ways to protect accounts against credential theft.

- Defense-in-depth requires network segmentation and container hardening so that compromising a single container does not lead to a full cluster breach.

# Conclusion
This lab showed how to secure cloud environments using access control and network protection. Authentication and MFA ensure that only verified users gain system access, while RBAC limits user actions based on explicit roles. Segmenting Docker networks prevents attackers from reaching database tiers directly, and default-deny firewall policies block unnecessary incoming traffic. Finally, container hardening and vulnerability scanning minimize the overall attack surface to keep services secure.

# Security Best-Practices Checklist
[/] Service requires authentication (unauthenticated requests rejected).  

[/] MFA / second factor implemented and validated.  

[/] Authorization enforced by RBAC (least privilege; unauthorized actions denied). 

[/] Network segmented so the data tier is unreachable from the front tier. 

[/] Default-deny firewall with explicit allow rules configured.  

[/] Container hardened: non-root, minimal, capabilities dropped, read-only; image scanned. 

# Verification Commands
Command Summary:

```
kubectl get rolebinding dev-rb -n app -o yaml
docker inspect hardened --format '{{json .HostConfig.CapDrop}}'
```
Expected Verification Results:

- YAML output showing dev-role attached to app:dev service account.

- Output showing ["ALL"] for dropped Linux capabilities.

Evidence:


<img width="542" height="365" alt="Screenshot 2026-08-28 221500" src="https://github.com/user-attachments/assets/16162275-e67e-4292-8b65-78e66294a282" />


# Cleanup
Command Summary:

```
docker rm -f authsvc db app web hardened 2>/dev/null
docker network rm frontend-net backend-net 2>/dev/null
kind delete cluster --name ccse-lab4
```
Evidence:


<img width="477" height="283" alt="Screenshot 2026-08-28 221738" src="https://github.com/user-attachments/assets/94445405-1a0a-4caf-b949-bbe137fdfb32" />


# References
- Course Lectures – Week 5 (Access Control); Week 9 (Network Security Patterns).

- Docker Engine Security Guidelines – https://docs.docker.com/engine/security/

- Center for Internet Security (CIS) Benchmarks for Docker & Kubernetes – https://www.cisecurity.org

- Cloud Security Alliance (CSA) Security Guidance v5 – Infrastructure & Networking; IAM.
