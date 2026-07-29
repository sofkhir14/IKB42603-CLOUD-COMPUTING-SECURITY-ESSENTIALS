Lab 0: Environment Setup
-

Objectives
-
To install and verify all necessary tools and environment components required (Docker, AWS CLI v2, kind, kubectl, and helper tools)

Learning Outcomes
-
Successfully configure and test a local cloud simulation environment using LocalStack and Docker.

Step-by-Step Implementation
-
Step 1: Install Docker
-
- Verify Docker by using *docker --version*
- Run *docker run --rm hello-world* command to test container execution

<img width="628" height="72" alt="Screenshot 2026-07-27 195718" src="https://github.com/user-attachments/assets/2bf77877-02cc-4388-8431-ab9d811b551f" />
<img width="637" height="197" alt="Screenshot 2026-07-27 195645" src="https://github.com/user-attachments/assets/c705abe6-aa10-4bf0-a1e0-8797aa71eb27" />

Step 2: Install AWS CLI v2
- 
- Verify AWS CLI by using *aws --version*
<img width="623" height="75" alt="Screenshot 2026-07-27 200723" src="https://github.com/user-attachments/assets/fa606d22-f5b1-46c0-ae4e-321cc590f194" />

Step 3: Install kind & kubectl
-
- Use *kind --version* and *kubectl version --client* to verify installation
- Verified both binaries respond with client version information
<img width="217" height="105" alt="Screenshot 2026-07-27 202530" src="https://github.com/user-attachments/assets/8075b8d4-8d72-443a-9086-767a58959ab7" />

Step 4: Verify Helper Tools
-
- Use *openssl version* to verify
- Ensured openssl is installed and functional
<img width="642" height="217" alt="Screenshot 2026-07-27 235653" src="https://github.com/user-attachments/assets/1ea0744a-7e68-4d5d-8977-09ce0a5f3c4f" />

Step 5: Test LocalStack & Local Kubernetes Cluster
-
- Started LocalStack container mapping port '4566' and checked the health endpoint
- Created a test Kubernetes cluster named 'ccse' and listed cluster nodes
<img width="612" height="222" alt="Screenshot 2026-07-29 004157" src="https://github.com/user-attachments/assets/63f881fa-cea2-44ff-8710-5df608178430" />
<img width="627" height="236" alt="Screenshot 2026-07-29 010254" src="https://github.com/user-attachments/assets/1645fc06-c371-456b-9aa6-5f8c5d96aa6d" />
<img width="627" height="267" alt="Screenshot 2026-07-29 010910" src="https://github.com/user-attachments/assets/0151e2c9-395d-4419-a994-40b2245810c8" />
<img width="627" height="187" alt="Screenshot 2026-07-29 011023" src="https://github.com/user-attachments/assets/1f9b0bcc-1bac-4dcc-91a1-c68c20017351" />
<img width="312" height="77" alt="Screenshot 2026-07-29 011215" src="https://github.com/user-attachments/assets/9fe13fbc-2150-4d65-9523-3b3df79ce8c4" />

Challenges Encountered
-
- Unable to install Docker daemon at first

Lessons Learned
- 
- Mastered the setup of local tooling required for upcoming security labs
