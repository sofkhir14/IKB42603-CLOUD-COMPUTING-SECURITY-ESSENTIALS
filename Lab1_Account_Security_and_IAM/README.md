Lab 1: Account Security and IAM (Report)
- 

Objective
- 
To construct secure cloud operations by applying the Principle of Least Privilege across local cloud infrastructures (LocalStack for AWS IAM) and container environments (Kubernetes RBAC).

Learning Outcomes
- 

- Apply least privilege principles by moving away from root usage and implementing scoped IAM users, groups, and policies. 
- Create and test fine-grained permissions (distinguishing allowed vs. denied actions).
- Implement and test Role-Based Access Control (RBAC) in Kubernetes using roles, service accounts, and namespaces.
-  Audit identities and reason about MFA, access keys and credential hygiene

Environment
- 

- Operating Systems: Docker Desktop and Kali Linux
- Local Cloud Emulator: LocalStack via Docker container (port 4456)
- Kubernetes cluster: kind (Kubernetes-in-Docker) local cluster named ccse-lab1
- CLI Tools: AWS CLI v2


Session A: Cloud Identity with LocalStack (AWS IAM)
- 

- <img width="612" height="222" alt="Screenshot 2026-07-29 004157" src="https://github.com/user-attachments/assets/11342120-ec2f-408d-8ce1-447a12d32276" />

Purpose: Runs the LocalStack container in the background (-d), names it localstack, and maps local port 4566 to container port 4566 so the AWS CLI can talk to local emulated AWS APIs

- <img width="627" height="236" alt="Screenshot 2026-07-29 010254" src="https://github.com/user-attachments/assets/b2a2220b-396d-459e-8eed-c56920fd46de" />

Purpose: Sends an HTTP request to LocalStack's health check endpoint to verify that all emulated cloud services are running properly.

- <img width="565" height="125" alt="Screenshot 2026-07-29 194639" src="https://github.com/user-attachments/assets/73abb81b-fa56-4fbf-b839-63198bafff36" />

Purpose: Overrides the default AWS endpoint to direct the request to LocalStack, querying the Security Token Service (STS) to display the active AWS identity.

Step-by-Step Implementations
- 
Task 1: Cloud Identity Landscape Table
- 

<img width="766" height="515" alt="Screenshot 2026-08-02 234643" src="https://github.com/user-attachments/assets/604aa488-f358-443b-9d84-cff8796df83e" />


Task 2: Least-Privilege Admin Setup
- 

- <img width="462" height="181" alt="Screenshot 2026-07-29 194946" src="https://github.com/user-attachments/assets/465ab467-c84e-4890-8d93-b1dfb4c255fe" />

Purpose: Creates an IAM Group names Admins to aggregate administrative permissions.

- <img width="737" height="67" alt="Screenshot 2026-07-29 200204" src="https://github.com/user-attachments/assets/0e4411db-83af-461c-a240-3dd9d6bf1fab" />

Purpose: Attaches the AWS managed policy AdministratorAccess to the Admins group which granting administrative rights to all members in that group.

- <img width="536" height="182" alt="Screenshot 2026-07-29 200117" src="https://github.com/user-attachments/assets/87f75e60-651b-496d-bcd0-8eb444229960" />

Purpose: Creates a dedicated user account for administrative tasks so it dont use the root account directly.

- <img width="667" height="42" alt="Screenshot 2026-07-29 200221" src="https://github.com/user-attachments/assets/1fe1fd5b-c014-4761-9579-8059505a701c" />

Purpose: Adds my admin user to the Admins group so it automatically inherits administrative rights through group inheritance. 

- <img width="571" height="337" alt="Screenshot 2026-07-29 200308" src="https://github.com/user-attachments/assets/d82dfd08-405c-4b8a-9a5e-17fe2e5ad1d4" />

Purpose: Audits the group membership to confirm that CloudAdmin_Sofea was added successfully.

Task 3: Scoped Read-Only Analyst
- 

- <img width="527" height="186" alt="Screenshot 2026-07-29 200813" src="https://github.com/user-attachments/assets/7758b505-f4c5-4baa-b9fc-a85783eeec97" />

Purpose: Creates a personal user account for a team member who only needs limited access.

- <img width="742" height="60" alt="Screenshot 2026-07-29 201006" src="https://github.com/user-attachments/assets/9f2c1c71-3e97-4e1d-9990-8097b55d0508" />

Purpose: Directly attaches a scoped policy allowing read-only access to Amazon S3. This prevent any write or delete privileges.

- <img width="602" height="172" alt="Screenshot 2026-07-29 201049" src="https://github.com/user-attachments/assets/fa6ec8ae-61b8-4082-9a2b-54612086e7d6" />

Purpose: Audits the Analyst user account to verify which policies are attached. 

Task 4: Credential Hygiene & Access Keys
- 

- <img width="583" height="187" alt="Screenshot 2026-07-29 201224" src="https://github.com/user-attachments/assets/ab04f817-f39e-46fa-aad2-bbedaef6480c" />

Purpose: Generates a programmatic Access Key ID and Secret Access Key for CLI/API interactions. 

- <img width="498" height="200" alt="Screenshot 2026-07-29 201310" src="https://github.com/user-attachments/assets/c192f43f-7f3d-43a1-b56b-542867b6d7e6" />

Purpose: Lists all active and inactive access keys with the Analyst account.

- <img width="742" height="65" alt="Screenshot 2026-07-29 201735" src="https://github.com/user-attachments/assets/5a4501c1-93c7-4fe0-b2e2-33efd7c1497d" />

Purpose: Deactivates an existing access key during credential rotation to revoke access without deleting the key history.

Session B: Kubernetes RBAC
- 

1. Cluster & Namespace Setup
- <img width="392" height="255" alt="Screenshot 2026-08-02 195009" src="https://github.com/user-attachments/assets/9d1a45c0-9cf8-4529-880e-b3decdc9025c" />

Purpose: Provisions a local Kubernetes cluster inside Docker containers named ccse-lab1 using kind.


- <img width="627" height="140" alt="Screenshot 2026-08-02 195715" src="https://github.com/user-attachments/assets/65e9881d-7bf9-49d3-93b4-1f978e33ff80" />

 <img width="541" height="68" alt="Screenshot 2026-08-02 195750" src="https://github.com/user-attachments/assets/7279dfae-f14b-4611-8190-ef0f7b8d5254" />

Purpose: Confirms that kubectl is communicating with the newly created cluster control plane and verifies node health.

- <img width="262" height="55" alt="Screenshot 2026-08-02 195828" src="https://github.com/user-attachments/assets/8bc3ca07-ece8-49b3-9044-b7804a510b9a" />

 <img width="273" height="58" alt="Screenshot 2026-08-02 195850" src="https://github.com/user-attachments/assets/43a79533-a335-4498-b73c-61f82f6676ea" />

Purpose: Creates two isolated logical environment (dev and prod) inside the same physical cluster. 

- <img width="280" height="177" alt="Screenshot 2026-08-02 195915" src="https://github.com/user-attachments/assets/9b093ada-b8ac-4c65-83af-f22db4c93385" />

Purpose: List all logical namespaces currently created inside the Kubernetes cluster.

2. Task 6: RBAC Configuration
- <img width="402" height="58" alt="Screenshot 2026-08-02 200044" src="https://github.com/user-attachments/assets/745328ca-a4b8-4fbb-a5d1-ce7ac88e7e2a" />

Purpose: Creates a programmatic identity (Service Account) named dev-user specifically inside the dev namespace.

- <img width="621" height="96" alt="Screenshot 2026-08-02 200239" src="https://github.com/user-attachments/assets/43a417b0-3206-40ee-9f34-dd59c7632cbc" />

Purpose: Creates a namespace-scoped Role object named pod-reader in dev that defines what actions are allowed (get,list,watch on pods).

- <img width="623" height="82" alt="Screenshot 2026-08-02 200414" src="https://github.com/user-attachments/assets/17fac402-8572-4e72-802d-0d40140f551f" />

Purpose: Binds the dev-user Service Account to the pod-reader Role within the dev namespace, granting it those specific permissions.

3. Task 7 & Verification: Testing Authorization Boundaries
- <img width="410" height="61" alt="Screenshot 2026-08-04 212659" src="https://github.com/user-attachments/assets/b8945601-bcbe-4680-8501-407a91b0fa37" />

Purpose: Simulates an API request as the Service Account to test if listing pods in dev is permitted.

- <img width="408" height="65" alt="Screenshot 2026-08-04 212833" src="https://github.com/user-attachments/assets/ea0abb1f-655f-486d-a753-63db1919b88e" />

Purpose: Tests if the Service Account can delete pods in dev, verifying that unauthorized verbs are blocked.

- <img width="408" height="56" alt="Screenshot 2026-08-04 212944" src="https://github.com/user-attachments/assets/8dab1727-bba7-49bd-9fe0-1d2b302b5777" />

Purpose: Tests if the permissions leak into the prod namespace. Proving namespace isolation. 

- <img width="477" height="296" alt="Screenshot 2026-08-02 202347" src="https://github.com/user-attachments/assets/c6005779-7f06-4b0d-afa0-b999f592688d" />

Purpose: Exports the full YAML configuration of the RoleBinding to prove and document that RBAC was correctly constructed.

Challenges Encountered
- 

- None

Lessons Learned
- 

- Blast Radius Reduction: Restricting accounts like Analyst to read-only roles limits the damage an attacker can perform if credentials are leaked.
- Authentication vs. Authorization: Authentication verifies identity, but Authorization determines resource access.
- Namespace Isolation: Role and RoleBindings in Kubernetes are namespaced by default. This makes Namespace Isolation effective fro multi-tenant or multi-environment clusters.


References
- 

- Course lectures — Week 1 (Fundamentals), Week 2 (Security Architecture), Week 5 (Access 
Control), Week 7 (Identity Management). 
- LocalStack documentation — docs.localstack.cloud
- Kubernetes RBAC — kubernetes.io/docs/reference/access-authn-authz/rbac 
- CSA Security Guidance v5 — Domain on Identity & Access Management.

Questions & Answers
- 

Task 3: Blast-Radius Reduction
- 

Question: If the Analyst account were stolen, why is the damage limited compared to a stolen admin account ? Connect your answer to blast-radius reduction.

Answer: If the Analyst account is compromised, the attacker can only view data because the account only has AmazonS3ReadOnlyAccess attached. The attacker cannot delete data, modify settings, or access other critical services like IAM configurations.

This directly demonstrates blast-radius reduction by applying the Principle of Least Privilege. 
 
Task 7 Question: Authentication vs Authorization
- 
Question: Relate the three can-i results to authentication versus authorization: which step is the service account passing, and which step is blocking the delete and the prod access?

Answer: 

Authentication: The service account passes authentication in all three checks. Kubernetes successfully recognizes system:servicesaccount:dev:dev-user as a valid identity within the cluster.

Authorization: The Authorization step is what blocks the unauthorized actions.

- It permits listing pods in dev because the pod-reader Role explicitly grants get, list, watch verbs
- It blocks deleting pods in dev because delete was never granted in the Role.
- It blocks listing pods in prod because the RoleBinding is namespaced to dev. This gives zero authorization in the prod namespace.

Short Answer Question
- 

Q1.Why is attaching policies to groups better than attaching them directly to users?

= Attaching policies to groups ensures scalability, consistency, and easier auditing. Instead of updating permission policies individually for every user, updating a single group policy automatically updates permissions for every member in thet group.

Q2.What is the difference between an IAM User and an IAM Role?

- IAM User: A permanent identity tied to a specific persons or service with long-lived credentials like a password or permanent access key.
- IAM Role: A temporary identity intended to be assumed by users or services.

Q3.Explain least privilege using the Analyst account, and how it reduces blast radius if
compromised.

= The Principle of Least Privilege giving an identity only the exact, minimum permissions required to perform its task. The Analyst only needs to inspect storage. So, granting AmazonS3ReadOnlyAccess satisfies their role. If compromised, the blast radius is reduced because an attacker cannot perform escalate privileges, modify the systems, or destroy resources.

Q4.In Kubernetes, what is the difference between a Role and a RoleBinding?

- Role: Defines what actions are allowed (verifying verbs like get or list on resources like pods) within a namespace.
- RoleBinding: Defines who get those permissions by linking a subject (like a user, group, or service account) to a specific Role.

Q5.Why did the developer service account fail to access prod, and which security principle does
that demonstrate?

- Role and RoleBinding objects in Kubernetes are namespaced. The service account was bound to a role inside the dev namespace, so it holds no permissions rights in prod.
- Security Principle:  Demostrates Namespace Isolation and the Principle of Least Privilege.
