Lab 2: Secure Isolation and Multitenancy
-

Course Information
- 

Course: IKB42604 Cloud Computing Security Essential

Lab: Lab 2 - Secure Isolation & Multitenancy

Name: Nurin Sofea Binti Mohammad Khir

Date: 8 August 2026

Objective
- 
- To demonstrate compute isolation using Kubernetes namespaces and containers
- To identify the security risks of shared default-open infrastructure
- To implement network isolation using a default-deny Network Policy
- To enforce storage and secret isolation using Role-Based Access Control (RBAC)
- To understand data remanence and perform secure data deletion.

Learning Outcomes
-
- CLO2: Construct secure cloud operations that safeguard data integrity
- Demonstrate compute, network, and storage isolation in a multi-tenant environment
- Apply security best practices to block unauthorized cross-tenant access.

Environment
- 
- Operating System: Linux / macOS / Windows with Admin rights
- Container Engine: Docker Desktop
- Kubernetes Cluster: kind with Calico CNI for NetworkPolicy enforcement
- Command Line Tools: kubectl, docker

Step-by-Step Implementation
- 
Session A: Compute Isolation & Default-Open Risk
- 
Setup Cluster with Policy Environment
- Create kind cluster with default CNI disabled and apply Calico CNI

Evidence & Output:

<img width="515" height="327" alt="Screenshot 2026-08-10 001229" src="https://github.com/user-attachments/assets/031c7355-8362-4ee7-ac44-c2becc0a33e8" />
<img width="620" height="117" alt="Screenshot 2026-08-10 010607" src="https://github.com/user-attachments/assets/3a0620ae-e06b-47f7-aedc-d9cce77bd477" />

No internet in the lab room, so apply kubectl apply -f calico.yaml


Task 1: Create Two Tenants

- Create namespaces and deploy simple web applications for Tenant A and Tenant B

Evidence & Output:

<img width="313" height="96" alt="Screenshot 2026-08-10 010800" src="https://github.com/user-attachments/assets/f8d1c3db-b4e0-44e7-a812-433e6a054dc8" />
<img width="573" height="260" alt="Screenshot 2026-08-10 010855" src="https://github.com/user-attachments/assets/7bd4d8d3-cdb1-446d-b053-2ecc992337fe" />

Task 2: Observe the Default-Open Risk

- Get Tenant B's service IP address and test connectivity from Tenant A

Evidence & Output:

<img width="618" height="60" alt="Screenshot 2026-08-10 070657" src="https://github.com/user-attachments/assets/460117f7-22ea-4d31-8483-06c457b866aa" />
<img width="626" height="115" alt="Screenshot 2026-08-10 070304" src="https://github.com/user-attachments/assets/4121d720-2df6-4e30-86ba-0db1f933c3fb" />

Task 3: Apply Resource Quotas

- Limit resources for Tenant A to prevent resource exhaustion

Evidence & Output:

<img width="310" height="236" alt="Screenshot 2026-08-10 071022" src="https://github.com/user-attachments/assets/1ba67d2b-31d3-4576-a1e3-d60a545b4458" />

Session B: Network & Storage Isolation
- 
Task 4: Default-Deny Network Isolation

- Apply a default-deny ingress policy to Tenant B and test connectivity again

Evidence & Output:

<img width="501" height="215" alt="Screenshot 2026-08-10 071209" src="https://github.com/user-attachments/assets/3382e5be-7cb4-4d6d-be01-e16e96cf0d7b" />

<img width="626" height="115" alt="Screenshot 2026-08-10 070304" src="https://github.com/user-attachments/assets/31cc3501-22cd-45f4-a8a5-67924c57372a" />  <img width="627" height="108" alt="Screenshot 2026-08-10 071407" src="https://github.com/user-attachments/assets/232e2fcb-22b3-4d61-91e5-5f017c21f516" />

Based on these two pictures which the output is HTTP 200 (before) and timeout (after), this is the strongest evidence of enforced network isolation

Task 5: Storage & Secret Isolation

- Create secrets and test access permissions using RBAC

Evidence & Output:

<img width="626" height="122" alt="Screenshot 2026-08-10 071512" src="https://github.com/user-attachments/assets/73ba0a44-f301-4f0c-be2a-42072907f01f" />
<img width="625" height="145" alt="Screenshot 2026-08-10 071656" src="https://github.com/user-attachments/assets/abc579ab-564d-4fde-873b-f2bc3552f0b2" />
<img width="622" height="105" alt="Screenshot 2026-08-10 072401" src="https://github.com/user-attachments/assets/7d2199af-4108-478d-8fff-3d2962bb6e43" />
<img width="456" height="167" alt="Screenshot 2026-08-10 072227" src="https://github.com/user-attachments/assets/edb7604f-bf03-4513-a0a3-fbf2952080c6" />

Task 6: Data Remanence & Secure Deletion

- Demonstrate data remanence and overwrite data 

Evidence & Output:

<img width="622" height="85" alt="Screenshot 2026-08-10 072655" src="https://github.com/user-attachments/assets/11e2c97b-873b-42f2-b88c-ec563abf543d" />
<img width="630" height="145" alt="Screenshot 2026-08-10 073017" src="https://github.com/user-attachments/assets/fa47793c-3d08-4f55-9712-7948c5111c7f" />

Commands Used and Their Purposes
- 
Setup & Task 1 Commands

- kind create cluster --name ccse-lab2 --config=-: Creates a local Kubernetes cluster named ccse-lab2
- kubectl apply -f calico.yaml: Installs Calico network provider to enable NetworkPolicy enforcement
- kubectl -n kube-system rollout status daemonset/calico-node: Checks if Calico is running correctly
- kubectl create namespace <name>: Creates isolated logical spaces for tenant-a and tenant-b
- kubectl -n <namespace> create deployment web --image=nginx: Deploys an Nginx web application for a tenant
- kubectl -n <namespace> expose deployment web --port=80: Creates a Kubernetes service to give access to port 80
- kubectl get pods, svc -n tenant-a: Displays running pods and services in tenant-a

Task 2 & Task 3 Commands

- kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}': Retrieves the internal IP address of Tenant B's service
- kubectl -n tenant-a run probe --rm -it --image curlimages/curl --restart=Never -- curl ...: Runs a temporary container in Tenant A to send a web request to Tenant B
- kubectl apply -f -: Creates a resource quota to restrict CPU, memory, and pod limit for Tenant A
- kubectl describe resourcequota tenant-a-quota -n tenant-a: Shows resource limits and current usage in Tenant A

Task 4, Task 5 & Task 6 Commands

- cat <<EOF (NetworkPolicy): * - -f -n <namespace Applies B. Tenant
  kubectl a all apply blocks incoming into kubectl network policy that traffic |> create secret generic data --from-literal=value=<val>: Stores a secret value safely inside a namespace
- kubectl -n tenant-a create serviceaccount app-a: Creates an identity for an app running in Tenant A
- kubectl -n tenant-a create role reader --verb=get --resource=secrets: Defines permissions to allow reading secrets
- kubectl -n tenant-a create rolebinding rb --role=reader --serviceaccount=tenant-a:app-a: Grants the permission role to the service account
- kubectl auth can-i get secrets -n <namespace> --as=$SA: Checks if Tenant A's service account can access secrets in a namespace
- docker run --rm -v ccse-vol:/data alpine sh -c '...': Runs an Alpine container to test file deletion and binary memory scanning

Short-Answer Questions
- 

Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?

Answer: By default, Kubernetes uses an open flat network model where all pods can communicate across namespaces. This is dangerous in a multi-tenant cloud because a compromised tenant can scan, attack, or access private data of another tenant

Q2. Explain the default-deny principle and how your Network Policy implements it.

Answer: The default-deny principle blocks all network traffic unless it is explicitly allowed. The applied NetworkPolicy uses an empty podSelector: {} for ingress, which blocks all incoming connections to pods in tenant-b

Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?

Answer: Virtual machines share physical hardware and run separate kernel instances, providing strong isolation. Containers share the underlying host operating system kernel, which provides weaker isolation. A VM boundary should be added when hosting untrusted code, processing sensitive compliance data, or separating completely different customers

Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?

Answer: Data remanence is the residual data that remains on a storage disk after a file is deleted. Cryptographic erasure is preferred in the cloud because users do not have physical access to hard drives. Deleting the encryption key makes the stored data completely unreadable instantly

Q5. Which of the three isolation dimensions (compute, network, storage) did each task exercise?

Answer:
- Task 1 & Task 3: Compute Isolation (Namespaces, Pods, and Resource Quotas).
- Task 2 & Task 4: Network Isolation (Default network behavior and NetworkPolicy).
- Task 5 & Task 6: Storage Isolation (Secrets RBAC, Data Remanence, and Secure Wipe).

Verification Outputs
- 
Run and include the verification commands output:

1. Verify Network Policy:
<img width="438" height="81" alt="Screenshot 2026-08-10 073156" src="https://github.com/user-attachments/assets/485b5ea5-6a92-434c-af67-574a8d2fb0c1" />

2. Verify Resource Quota:
<img width="502" height="157" alt="Screenshot 2026-08-10 073223" src="https://github.com/user-attachments/assets/67e78a63-eabc-42b5-8f29-26f098669aca" />

Challenges Encountered
- 
Issue: Calico CNI took extra time to start up. This cause network policy failure initially.

Solution: Used kubectl rollout status command to wait until all Calico pods were completely ready before testing.

Lessons Learned
- 

- Kubernetes namespaces provide logical compute boundary but do not provide network isolation by default.
- Implementing a default-deny NetworkPolicy is mandatory for securing multi-tenant applications.
- RBAC ensures strict storage and secret isolation between different tenants.

References
- 
• Course lecture — Week 3 (Secure Isolation of Physical & Logical Infrastructure)

• Kubernetes Network Policies — kubernetes.io/docs/concepts/services-networking/network
policies 

• Calico documentation — docs.tigera.io 

• CSA Security Guidance v5 — Infrastructure & Networking domain
