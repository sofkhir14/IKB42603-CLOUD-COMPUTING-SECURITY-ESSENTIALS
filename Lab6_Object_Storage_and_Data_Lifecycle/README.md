# Lab 6: Object Storage Security & the Data Security Lifecycle
Course: IKB42603 Cloud Computing Security Essentials

Lab: Lab 6

Topic: Object storage security, bucket resource policies, SSE-KMS default encryption, presigned URLs, versioning remanence, lifecycle rules, and cryptographic erasure

Environment: LocalStack, Docker, AWS CLI v2, Linux shell tools (curl, jq)

Name: Nurin Sofea Binti Mohammad Khir

# Objective
The objective of this lab is to configure, evaluate, and secure cloud object storage throughout the Data Security Lifecycle. Data assests will be clarify, reproduce and remediate a public bucket exposure, evaluate identity-based versus resource-based IAM policies, enforce default SSE-KMS envelope encryption, evaluate versioning data remanence, configure lifecycle retention rules, and execute provable deletion via cryptographic erasure.

# Learning Outcomes
By completing this lab, I will be able to:

- Provision cloud object storage, apply object classification tags, and differentiate object storage access controls from standard file/block systems.

- Reproduce the archetypal public bucket data breach and remediate it using Block Public Access and least-privilege bucket policies.

- Differentiate between identity-based policies (IAM) and resource-based policies (Bucket Policies), predicting access evaluation outcomes when explicit Deny rules exist.

- Enforce mandatory server-side encryption at rest (SSE-KMS) with Customer Managed Keys (CMK).

- Generate and evaluate time-bounded presigned URLs for delegated access.

- Demonstrate object-level data remanence using S3 versioning and delete markers, and achieve provable deletion using cryptographic erasure.

# Environment
- Operating System / Terminal: Kali Linux

- Cloud Emulation & CLI: AWS CLI v2, LocalStack (S3, KMS, IAM)

- Containerization: Docker Engine

- Command Line Utilities: curl, awk, sed, jq

# Data Classification Table

| Classification | Who May Read It | Impact if Leaked | Control Applied |
| :--- | :--- | :--- | :--- |
| **public** | Anyone (Anonymous internet users) | None (Public information) | Public read access / Unrestricted |
| **internal** | Authenticated hospital staff & employees | Low-Medium (Operational exposure) | Least-privilege policy restricted to account root/IAM roles |
| **confidential** | Authorized medical personnel only | Critical (Privacy violation, regulatory penalties under PDPA/GDPR) | Explicit IAM Deny policy, SSE-KMS encryption, and version retention |

# Task 1: Classify the Data Before You Store It
An S3 bucket was created, three objects with varying sensitivity levels were uploaded, and classification tags were assigned.

Command Summary:
```
# Set bucket name
export BUCKET=miit-patient-records-$RANDOM
echo "Bucket Name: $BUCKET"

# Create bucket
aws $EP s3api create-bucket --bucket $BUCKET

# Create local data files
echo 'Ward visiting hours 10am-8pm' > public-notice.txt
echo 'Staff duty schedule, week 12' > internal-roster.txt
echo 'Patient: Ahmad bin Ali, Diagnosis: confidential' > confidential-record.txt

# Upload objects with classification tags
aws $EP s3api put-object --bucket $BUCKET --key public/notice.txt \
  --body public-notice.txt --tagging 'classification=public'
aws $EP s3api put-object --bucket $BUCKET --key internal/roster.txt \
  --body internal-roster.txt --tagging 'classification=internal'
aws $EP s3api put-object --bucket $BUCKET --key confidential/record.txt \
  --body confidential-record.txt --tagging 'classification=confidential'

# Verify uploads and tags
aws $EP s3api list-objects-v2 --bucket $BUCKET --query 'Contents[].[Key,Size]' --output table
aws $EP s3api get-object-tagging --bucket $BUCKET --key confidential/record.txt
```
Result:

The terminal displayed the three uploaded objects with their key prefixes and confirmed that the tag classification=confidential was successfully applied.

Evidence:


# Task 2: Reproduce the Archetypal Breach
