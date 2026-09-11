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

The terminal displayed the three uploaded objects with their key prefixes and confirmed that the tag ```classification=confidential``` was successfully applied.

Evidence:


# Task 2: Reproduce the Archetypal Breach
A overly permissive resource policy granting ```Principal: "*"``` access to ```s3:GetObject``` was applied to simulate an unauthenticated public breach.

Command Summary:
```
cat > public-policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadEverything",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/*"
  }]
}
JSON

aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://public-policy.json

# Test unauthenticated access via curl
curl -s -o leaked.txt -w 'HTTP %{http_code}\n' http://localhost:4566/$BUCKET/confidential/record.txt
cat leaked.txt
```
Result:

The ```curl``` command returned ```HTTP 200``` and printed the confidential patient record without supplying any AWS credentials. This illustrates an unauthenticated bucket breach.

Evidence:


# Task 3: Remediate with Block Public Access
The public policy was removed, S3 Block Public Access guardrails were enabled, and a least-privilege resource policy was applied.

Command Summary:
```
# 1. Remove public bucket policy
aws $EP s3api delete-bucket-policy --bucket $BUCKET

# 2. Apply Block Public Access guardrails
aws $EP s3api put-public-access-block --bucket $BUCKET \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws $EP s3api get-public-access-block --bucket $BUCKET

# 3. Apply least-privilege bucket policy scoped to internal prefix
cat > least-privilege-policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AccountReadInternalOnly",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::000000000000:root"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/internal/*"
  }]
}
JSON

aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://least-privilege-policy.json
```
Result:

```get-public-access-block``` confirmed that all four public access block settings were enabled. This secure the bucket against public access configurations.

Evidence:


# Task 4: Identity Policy vs Resource Policy
An IAM user (```DataAnalyst```) with full read permissions was created. A bucket policy containing an explicit ```Deny``` on ```confidential/*``` was applied to verify policy evaluation logic.

Command Summary:
```
# Create IAM user and attach full S3 read policy
aws $EP iam create-user --user-name DataAnalyst

cat > analyst-iam.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": "*"
  }]
}
JSON

aws $EP iam put-user-policy --user-name DataAnalyst \
  --policy-name S3ReadAll --policy-document file://analyst-iam.json

# Create credentials for DataAnalyst
KEYS=$(aws $EP iam create-access-key --user-name DataAnalyst --query 'AccessKey.[AccessKeyId,SecretAccessKey]' --output text)
ANALYST_KEY_ID=$(echo $KEYS | awk '{print $1}')
ANALYST_SECRET=$(echo $KEYS | awk '{print $2}')

aws configure --profile analyst set aws_access_key_id "$ANALYST_KEY_ID"
aws configure --profile analyst set aws_secret_access_key "$ANALYST_SECRET"
aws configure --profile analyst set region us-east-1

# Apply bucket policy with explicit Deny on confidential/*
cat > deny-confidential.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAnalystInternal",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::000000000000:user/DataAnalyst"},
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::$BUCKET/internal/*"
    },
    {
      "Sid": "DenyAnalystConfidential",
      "Effect": "Deny",
      "Principal": {"AWS": "arn:aws:iam::000000000000:user/DataAnalyst"},
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::$BUCKET/confidential/*"
    }
  ]
}
JSON

aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://deny-confidential.json

# Test access under analyst profile
AWS_PROFILE=analyst aws $EP s3api get-object --bucket $BUCKET --key internal/roster.txt analyst-internal.txt && echo "internal: ALLOWED"
AWS_PROFILE=analyst aws $EP s3api get-object --bucket $BUCKET --key confidential/record.txt analyst-conf.txt || echo "confidential: DENIED"

# Cleanup bucket policy before Session B
aws $EP s3api delete-bucket-policy --bucket $BUCKET
```
Result:

Reading ```internal/roster.txt``` succeeded (```Allowed```), whereas reading ```confidential/record.txt``` was rejected (```Denied```), proving that an explicit ```Deny``` in a resource policy overrides an ```Allow``` in an identity policy.

Evidence:


# Task 5: Default Encryption at Rest (SSE-KMS)
A Customer Managed Key (CMK) was generated in KMS, and default SSE-KMS bucket encryption was enforced.

Command Summary:
```
# Create KMS key
export KEY_ID=$(aws $EP kms create-key \
  --description 'IKB42603 Lab6 patient records bucket key' \
  --query 'KeyMetadata.KeyId' --output text)

# Configure default bucket encryption
cat > encryption.json <<JSON
{
  "Rules": [{
    "ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "$KEY_ID"
    },
    "BucketKeyEnabled": true
  }]
}
JSON

aws $EP s3api put-bucket-encryption --bucket $BUCKET --server-side-encryption-configuration file://encryption.json

# Upload object without encryption flags
aws $EP s3api put-object --bucket $BUCKET --key confidential/record-v2.txt --body confidential-record.txt

# Inspect object metadata
aws $EP s3api head-object --bucket $BUCKET --key confidential/record-v2.txt \
  --query '[ServerSideEncryption,SSEKMSKeyId,BucketKeyEnabled]' --output text
```
Result:

```head-object``` returned ```aws:kms```, the CMK Key ID, and ```True``` for ```BucketKeyEnabled```, demonstrating default bucket-level envelope encryption.

Evidence:


# Task 6: Delegated Access and the Condition-Key Trap
A presigned URL was generated to grant time-bounded access. Next, a policy requiring ```aws:SecureTransport``` was evaluated.

Command Summary:
```
# Generate presigned URL valid for 60 seconds
URL=$(aws $EP s3 presign s3://$BUCKET/internal/roster.txt --expires-in 60)
curl -s -w '  <-- HTTP %{http_code}\n' "$URL"

# Apply SecureTransport enforcement policy
cat > secure-transport.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyUnencryptedTransport",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": ["arn:aws:s3:::$BUCKET", "arn:aws:s3:::$BUCKET/*"],
    "Condition": {"Bool": {"aws:SecureTransport": "false"}}
  }]
}
JSON

aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://secure-transport.json

# Test API call (Expected lockout over plain HTTP)
aws $EP s3api list-objects-v2 --bucket $BUCKET || echo "Access Denied via HTTP"

# Recover bucket access
aws $EP s3api delete-bucket-policy --bucket $BUCKET
```
Result:

The presigned URL allowed object download. Applying the ```aws:SecureTransport``` policy blocked subsequent local HTTP commands because LocalStack's endpoint (```http:///localhost:4566```) evaluates ```aws:SecureTransport``` as ```false```.

Evidence:


# Task 7: Versioning, Delete Markers & Data Remanence
Bucket versioning was enabled, multiple object versions were created, and object-level data remanence was evaluated after executing a standard delete command.

Command Summary:
```
# Enable bucket versioning
aws $EP s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled

# Upload updated versions
echo 'Patient: Ahmad bin Ali, Diagnosis: hypertension' > rec-v2.txt
echo 'Patient: [REDACTED], Diagnosis: [REDACTED]' > rec-v3.txt

aws $EP s3api put-object --bucket $BUCKET --key confidential/record.txt --body rec-v2.txt
aws $EP s3api put-object --bucket $BUCKET --key confidential/record.txt --body rec-v3.txt

# Execute standard object deletion
aws $EP s3api delete-object --bucket $BUCKET --key confidential/record.txt

# Verify list of versions and delete markers
aws $EP s3api list-object-versions --bucket $BUCKET --prefix confidential/record.txt

# Recover historical version using null VersionId
aws $EP s3api get-object --bucket $BUCKET --key confidential/record.txt --version-id null recovered.txt
cat recovered.txt

# Permanently purge the historical version
aws $EP s3api delete-object --bucket $BUCKET --key confidential/record.txt --version-id null
```
Result:

Issuing ```delete-object``` created a Delete Market without removing historical data. The original confidential file was successfully retrieved using ```--version-id null```. This demonstrates data remanence.

Evidence:


# Task 8: Lifecycle, Retention & Cryptographic Erasure
An automated S3 lifecycle policy was applied. Cryptographic erasure was then executed by disabling and scheduling the deletion of the KMS Customer Managed Key.

Command Summary:
```
# Configure lifecycle policy
cat > lifecycle.json <<'JSON'
{
  "Rules": [
    {
      "ID": "RetireConfidentialRecords",
      "Filter": {"Prefix": "confidential/"},
      "Status": "Enabled",
      "Expiration": {"Days": 365},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 30}
    },
    {
      "ID": "AbortIncompleteUploads",
      "Filter": {"Prefix": ""},
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }
  ]
}
JSON

aws $EP s3api put-bucket-lifecycle-configuration --bucket $BUCKET --lifecycle-configuration file://lifecycle.json

# Execute Cryptographic Erasure
aws $EP kms disable-key --key-id $KEY_ID
aws $EP kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7

# Verify KMS key status
aws $EP kms describe-key --key-id $KEY_ID --query 'KeyMetadata.[KeyState,DeletionDate]' --output text
```
Result:

The lifecycle rules were applied successfully, and the KMS key state transitioned to ```PendingDeletion```, rendering encrypted ciphertexts unrecoverable.

Evidence:


# Command Used
```
# S3 Bucket & Policy Management
aws s3api create-bucket --bucket <bucket>
aws s3api put-bucket-policy --bucket <bucket> --policy file://<policy.json>
aws s3api put-public-access-block --bucket <bucket> --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Server-Side Encryption & Lifecycle Configuration
aws kms create-key --description <desc>
aws s3api put-bucket-encryption --bucket <bucket> --server-side-encryption-configuration file://encryption.json
aws s3api put-bucket-lifecycle-configuration --bucket <bucket> --lifecycle-configuration file://lifecycle.json

# Versioning, Recovery & Key Erasure
aws s3api put-bucket-versioning --bucket <bucket> --versioning-configuration Status=Enabled
aws s3api get-object --bucket <bucket> --key <key> --version-id <version_id> <output_file>
aws kms schedule-key-deletion --key-id <key_id> --pending-window-in-days 7
```

# Short-Answer Questions
Q1: Which single element of the Task 2 policy caused the exposure, and why is ```Principal: "*"``` more dangerous on a bucket policy than an over-broad IAM policy attached to one user?

- The single JSON key-value element that caused the breach was ```"Principal: "*'```. This element grants unauthenticated access to any entity on the internet. It is significantly more dangerous on a resource policy than on an identity policy because an IAM policy only expands permissions for one authenticated entity, whereas ```"Principal: "*"``` on a resource policy makes the source publicly accessible to the entire internet without credentials.

Q2: Explain the difference between an identity-based policy and a resource-based policy. In Task 4, which one decided each of the analyst's two requests?

- Identity-based policy (IAM): Attached directly to a user, group, or role, defining what that specific identity is permitted to do across cloud resources.

- Resource-based policy (Bucket Policy): Attached directly to a resource such as S3 Bucket, defining who can access that specific resource and under what conditions.

- Task 4 Decision: For ```internal/roster.txt```, access was granted because both the identity policy and bucket policy contained explicit ```allow``` statements. For ```confidential/record.txt```, the resource policy's explicit ```Deny``` statement (```DenyAnalystConfidential```) tool precedence over the identity policy's ```Allow``` which blocking the request.

Q3: Block Public Access is described as a guardrail rather than a control. What is the difference, and why does the distinction matter for an organisation with many engineers?

- A security control remediates or flags a specific risk after or during configuration, whereas a preventive guardrail acts as an absolute policy boundary that overrides and rejects non-compliant configurations regardless of user privilege. For organizations with many engineers, guardrails prevent accidental bucket misconfigurations even if a developer uploads a flawed policy file.

Q4: Your bucket has default SSE-KMS encryption. Does that protect the confidential record from the analyst in Task 4? Explain precisely what server-side encryption does and does not defend against.

- No, SSE-KMS does not protect the record from the analyst if the analyst holds valid IAM and KMS read permissions. Server-Side Encryption protects against physical media theft, unauthorized storage snapshot access, or disk exposure at the cloud provider infrastructure layer. It does not replace logical authorization controls (IAM/Resource policies). Any identity with ```s3:GetObject``` and ```kms:Decrypt``` permissions will transparently receive the decrypted data.

Q5: A patient invokes their right to erasure. Using your Task 7 evidence, explain why ```delete-object``` alone is not compliant, and describe two mechanisms that would make the deletion provable.

- Explicit Version Purging: Iteratively issue ```delete-object``` referencing every specific ```versionID```.

- Cryptographic Erasure: Destroy or revoke the KMS Customer Managed Key encrypting the target data, rendering stored ciphertexts unrecoverable noise.

Q6: You are the auditor in Week 11. Name three commands from this lab whose output you would collect as compliance evidence, and state which control each one evidences.

- ```aws s3api get-public-access-block --bucket <bucket>```: Evidence preventative guardrail controls blocking public exposure.

- ```aws s3api get-bucket-encryption --bucket <bucket>```: Evidences mandatory default encryption at rest using SSE-KMS.

- ```aws s3api get-bucket-lifecycle-configuration --bucket <bucket>```: Evidences automated data retention and retirement rules meeting regulatory compliance standards.

# Challenges Encountered


# Lessons Learned
- Public storage breaches stem from misconfigured resource policies containing ```"Principal: "*"``` rather than complex software exploits.

- AWS authorization evaluation follows an explicit precedence model where an explicit ```Deny``` overrides any ```Allow```.

- Standard deletion commands on versioned S3 buckets create Delete Markers rather than purging underlying data. This require explicit version removal or cryptographic key destruction.

# Security Best-Practices Checklist
- [x] Every object carries a classification tag before access decisions are made.

- [x]  No bucket policy contains ```"Principal: "*"```; Anonymous access is blocked.

- [x]  Access permissions follow least privilege and are scoped to explicit key prefixes.

- [x]  Default server-side encryption is set to ```aws"kms``` using a Customer Managed Key.

- [x]  S3 Versioning is enabled, with lifecycle policy automation configured.

# Verification Command
Execute the following verification block to prove the final bucket security posture:

Command Summary:
```
echo "=== IKB42603 Lab 6 verification: $BUCKET ==="
aws $EP s3api get-public-access-block --bucket $BUCKET \
  --query 'PublicAccessBlockConfiguration' --output text
aws $EP s3api get-bucket-versioning --bucket $BUCKET --output text
aws $EP s3api get-bucket-encryption --bucket $BUCKET \
  --query 'ServerSideEncryptionConfiguration.Rules[0].ApplyServerSideEncryptionByDefault.[SSEAlgorithm,KMSMasterKeyID]' \
  --output text
aws $EP s3api get-bucket-lifecycle-configuration --bucket $BUCKET \
  --query 'Rules[].[ID,Status]' --output text
aws $EP kms describe-key --key-id $KEY_ID --query 'KeyMetadata.KeyState' --output text
```

# Cleanup 
```
# Delete bucket policy and all object versions/delete markers
aws $EP s3api delete-bucket-policy --bucket $BUCKET 2>/dev/null

aws $EP s3api delete-objects --bucket $BUCKET --delete "$(aws $EP s3api \
  list-object-versions --bucket $BUCKET --output json \
  --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}')" 2>/dev/null

aws $EP s3api delete-objects --bucket $BUCKET --delete "$(aws $EP s3api \
  list-object-versions --bucket $BUCKET --output json \
  --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}')" 2>/dev/null

# Delete bucket and IAM test user
aws $EP s3api delete-bucket --bucket $BUCKET
aws $EP iam delete-user-policy --user-name DataAnalyst --policy-name S3ReadAll
aws $EP iam delete-user --user-name DataAnalyst
```

# Conclusion
This lab evaluated the complete Data Security Lifecycle across cloud object storage environments. Implementing data classification tagging, enabling account-wide Block Public Access guardrails, enforcing default SSE-KMS encryption, and managing explicit ```Deny``` resource policies mitigates common storage misconfiguration risks. Furthermore, understanding object versioning remanence and leveraging lifecycle policies and cryptographic erasure ensures compliant data retention and destruction workflows.

# References
1. Course Lectures – Week 4 (Data Protection); Week 10 (Policy, Compliance & Risk); Week 11 (Compliance Assessment & Reporting).

2. Amazon S3 Security Best Practices – https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html

3. Amazon S3 Versioning and Lifecycle Management – https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html

4. Cloud Security Alliance (CCSK v5) – Domain 4 (Organization Management) & Domain 5 (Data Security).
