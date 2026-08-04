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

- ![alt text](<Screenshot 2026-07-29 004157.png>)
Purpose: Runs the LocalStack container in the background (-d), names it localstack, and maps local port 4566 to container port 4566 so the AWS CLI can talk to local emulated AWS APIs
- ![alt text](<Screenshot 2026-07-29 010254-1.png>)
Purpose: Sends an HTTP request to LocalStack's health check endpoint to verify that all emulated cloud services are running properly.
- ![alt text](<Screenshot 2026-07-29 194639.png>)
Purpose: Overrides the default AWS endpoint to direct the request to LocalStack, querying the Security Token Service (STS) to display the active AWS identity.

Step-by-Step Implementations
- 
Task 1: Cloud Identity Landscape Table
- 

![alt text](image-1.png)

Task 2: Least-Privilege Admin Setup
- 

- ![alt text](<Screenshot 2026-07-29 194946.png>)
Purpose: Creates an IAM Group names Admins to aggregate administrative permissions.

- ![alt text](<Screenshot 2026-07-29 200204.png>)
Purpose: Attaches the AWS managed policy AdministratorAccess to the Admins group which granting administrative rights to all members in that group.

- ![alt text](<Screenshot 2026-07-29 200117.png>)
Purpose: Creates a dedicated user account for administrative tasks so it dont use the root account directly.

- ![alt text](<Screenshot 2026-07-29 200221.png>)
Purpose: Adds my admin user to the Admins group so it automatically inherits adminstrative rights through group inheritance. 

- ![alt text](<Screenshot 2026-07-29 200308.png>)
Purpose: Audits the group membership to confirm that CloudAdmin_SOFEA was added successfully.

Task 3: Scoped Read-Only Analyst
- 

- ![alt text](<Screenshot 2026-07-29 200813.png>)
Purpose: Creates a personal user account for a team member who only needs limited access.

- ![alt text](<Screenshot 2026-07-29 201006.png>)
Purpose: Directly attaches a scoped policy allowing read-only access to Amazon S3. This prevent any write or delete privileges.

- ![alt text](<Screenshot 2026-07-29 201049.png>)
Purpose: Audits the Analyst user account to verify which policies are attached. 

Task 4: Credential Hygiene & Access Keys
- 

- ![alt text](<Screenshot 2026-07-29 201224.png>)
Purpose: Generates a programmatic Access Key ID and Secret Access Key for CLI/API interactions. 

- ![alt text](<Screenshot 2026-07-29 201310.png>) 
Purpose: Lists all active and inactive access keys with the Analyst account.

- ![alt text](<Screenshot 2026-07-29 201735.png>)
Purpose: Deactivates an existing access key during credential rotation to revoke access without deleting the key history.

Session B: Kubernetes RBAC
- 

1. Cluster & Namespace Setup
- ![alt text](<Screenshot 2026-08-02 195009.png>)
Purpose: Provisions a local Kubernetes cluster inside Docker containers named ccse-lab1 using kind.


- ![alt text](<Screenshot 2026-08-02 195715.png>)
 ![alt text](<Screenshot 2026-08-02 195750.png>)
Purpose: Confirms that kubectl is communicating with the newly created cluster control plane and verifies node health.

- ![alt text](<Screenshot 2026-08-02 195828.png>)
 ![alt text](<Screenshot 2026-08-02 195850.png>) 
Purpose: Creates two isolated logical environment (dev and prod) inside the same physical cluster. 

- ![alt text](<Screenshot 2026-08-02 195915.png>) 
Purpose: List all logical namespaces currently created inside the Kubernetes cluster.

2. Task 6: RBAC Configuration
- ![alt text](<Screenshot 2026-08-02 200044.png>)
Purpose: Creates a programmatic identity (Service Account) named dev-user specifically inside the dev namespace.

- ![alt text](<Screenshot 2026-08-02 200239.png>)
Purpose: Creates a namespace-scoped Role object named pod-reader in dev that defines what actions are allowed (get,list,watch on pods).

- ![alt text](<Screenshot 2026-08-02 200414.png>)
Purpose: Binds the dev-user Service Account to the pod-reader Role within the dev namespace, granting it those specific permissions.

3. Task 7 & Verification: Testing Authorization Boundaries
- ![alt text](<Screenshot 2026-08-04 212659.png>)
Purpose: Simulates an API request as the Service Account to test if listing pods in dev is permitted.

- ![alt text](image-2.png)
Purpose: Tests if the Service Account can delete pods in dev, verifying that unauthorized verbs are blocked.

- ![alt text](image-3.png)
Purpose: Tests if the permissions leak into the prod namespace. Proving namespace isolation. 

- ![alt text](<Screenshot 2026-08-02 202347.png>)
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
