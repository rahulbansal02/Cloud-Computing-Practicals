# Practical 10 — Cloud Security

---

## 📌 Objective

Implement cloud security best practices including IAM roles and policies with the principle of least privilege, data encryption for storage, and firewall rules — and test each configuration.

---

## 🧠 Conceptual Background (Know-How)

### The Shared Responsibility Model

This is the foundation of all cloud security:

```
┌─────────────────────────────────────────────────────────┐
│              Shared Responsibility Model                │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │            CUSTOMER Responsibility               │  │
│  │                                                  │  │
│  │  • Customer data                                 │  │
│  │  • Identity & Access Management (IAM)            │  │
│  │  • Application configuration                     │  │
│  │  • OS patches (for IaaS/EC2)                     │  │
│  │  • Network traffic protection                    │  │
│  │  • Firewall configuration                        │  │
│  │  • Encryption of data at rest and in transit     │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │                AWS Responsibility                 │  │
│  │                                                  │  │
│  │  • Physical hardware                             │  │
│  │  • Data center security                          │  │
│  │  • Hypervisor                                    │  │
│  │  • Managed service patching (RDS, S3, etc.)      │  │
│  │  • Global infrastructure                         │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### IAM Core Concepts

```
┌──────────────────────────────────────────────────────────┐
│                    AWS IAM Structure                     │
│                                                          │
│    User                  Role                           │
│  (Long-term           (Short-term                       │
│   credentials)         credentials)                     │
│     │                     │                             │
│     └──────┬──────────────┘                             │
│            │                                            │
│         Policy ──── Permissions                         │
│         (JSON        (Allow/Deny                        │
│          doc)         on Resources)                     │
│                                                          │
│  Group → Contains multiple Users                        │
│  Role  → Assumed by Users, Services, or EC2 instances   │
└──────────────────────────────────────────────────────────┘
```

### Principle of Least Privilege

**"Grant only the minimum permissions required to perform a task"**

```
❌ BAD — Overly permissive:
   Effect: Allow
   Action: "*"
   Resource: "*"
   (This gives full admin access to EVERYTHING)

✅ GOOD — Least privilege:
   Effect: Allow
   Action: ["s3:GetObject", "s3:PutObject"]
   Resource: "arn:aws:s3:::my-bucket/*"
   (Only allows reading/writing to ONE specific bucket)
```

---

## 🛠️ Step-by-Step Guide

### Step A — Implement IAM Roles and Policies

#### Part 1: Create IAM Users Without Console Access (Programmatic Only)

```
AWS Console → IAM → Users → Create user
  User name:     s3-reader-user
  Access type:   ✅ Access key – Programmatic access
  (No console access)
→ Next
```
<img width="1919" height="937" alt="upload3" src="https://github.com/user-attachments/assets/7066fa85-77e4-410d-9a81-46f7e6586165" />


#### Part 2: Create Least-Privilege Policies

**Example 1: Read-Only S3 Policy for a Specific Bucket**

```
IAM → Policies → Create policy → JSON tab
```

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ListSpecificBucket",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": "arn:aws:s3:::my-specific-bucket"
        },
        {
            "Sid": "ReadObjectsInBucket",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion"
            ],
            "Resource": "arn:aws:s3:::my-specific-bucket/*"
        }
    ]
}
```

```
Policy name:        S3ReadOnly-MyBucket
Description:        Read-only access to my-specific-bucket only
→ Create policy
```
<img width="1917" height="851" alt="upload4" src="https://github.com/user-attachments/assets/2d18e341-3735-4f6e-baa4-a3dfad76b835" />


**Example 2: EC2 Read-Only with Region Restriction**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "EC2ReadOnlyUsEast1",
            "Effect": "Allow",
            "Action": [
                "ec2:Describe*",
                "ec2:Get*",
                "ec2:List*"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": "us-east-1"
                }
            }
        }
    ]
}
```
<img width="1897" height="641" alt="upload5" src="https://github.com/user-attachments/assets/e756c004-a03c-4348-921b-02c9107a07cb" />



**Example 3: Deny Expensive Instance Types**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyExpensiveInstances",
            "Effect": "Deny",
            "Action": "ec2:RunInstances",
            "Resource": "arn:aws:ec2:*:*:instance/*",
            "Condition": {
                "ForAnyValue:StringNotLike": {
                    "ec2:InstanceType": [
                        "t2.*",
                        "t3.*",
                        "t3a.*"
                    ]
                }
            }
        }
    ]
}
```

#### Part 3: Create an IAM Role for EC2

IAM Roles allow EC2 instances to access AWS services **without hardcoding credentials**.

```
IAM → Roles → Create role
  Trusted entity type:    AWS service
  Use case:              EC2
→ Next

Permissions:
  Search and add: AmazonS3ReadOnlyAccess
→ Next

Role details:
  Role name:     EC2-S3-ReadOnly-Role
  Description:   Allows EC2 instances to read from S3
→ Create role
```
<img width="1913" height="878" alt="upload6" src="https://github.com/user-attachments/assets/cbb6b698-2cc9-42d3-ac6f-4600af40443f" />



**Attach the role to an EC2 instance:**
```
EC2 → Instances → Select instance → Actions →
Security → Modify IAM role → Select EC2-S3-ReadOnly-Role →
Update IAM role
```
<img width="1912" height="353" alt="upload7" src="https://github.com/user-attachments/assets/4da7ce72-81da-41eb-920e-0775e77e0fd3" />



**Test from inside the instance:**
```bash
# SSH into the instance
ssh -i keypair.pem ec2-user@<ip>

# Test S3 access (should work — role grants permission)
aws s3 ls s3://my-specific-bucket/

# Test creating a bucket (should fail — no permission)
aws s3 mb s3://new-bucket-test
# Error: An error occurred (AccessDenied) ...
```
<img width="815" height="475" alt="image" src="https://github.com/user-attachments/assets/b5cc862f-0c0a-4526-94c8-914b38bd29c0" />




---

### Step B — Create and Assign Least-Privilege Roles to Users

#### Create IAM User Groups with Specific Permissions

```
IAM → User groups → Create group
  Group name:   developers
  
  Attach policies:
    - EC2ReadOnlyAccess (AWS managed)
    - S3ReadOnly-MyBucket (custom policy created above)
→ Create group
```
<img width="1917" height="474" alt="upload8" src="https://github.com/user-attachments/assets/238d6dc2-fe57-4194-85f4-84a2719e3860" />



```
IAM → User groups → Create group
  Group name:   data-engineers
  
  Attach policies:
    - AmazonS3FullAccess (scoped by condition)
    - AmazonRDSReadOnlyAccess
→ Create group
```

**Add Users to Groups:**
```
IAM → Users → Select user → Groups tab → Add user to groups →
Select "developers" → Add to groups
```
<img width="1548" height="396" alt="upload9" src="https://github.com/user-attachments/assets/eeee43ab-cc29-41c6-a988-8b59d10ebcb3" />


#### IAM Policy Testing with IAM Policy Simulator

```
AWS Console → IAM → Policy Simulator (https://policysim.aws.amazon.com/)

1. Select a user or role
2. Select service (e.g., S3)
3. Select actions (e.g., PutObject)
4. Click "Run Simulation"
→ See if the action is Allowed or Denied and why
```

#### MFA Enforcement Policy (Best Practice)

Force users to enable MFA before accessing any resource:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowViewAccountInfo",
            "Effect": "Allow",
            "Action": [
                "iam:GetAccountPasswordPolicy",
                "iam:ListVirtualMFADevices"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowManageOwnMFA",
            "Effect": "Allow",
            "Action": [
                "iam:CreateVirtualMFADevice",
                "iam:EnableMFADevice",
                "iam:ListMFADevices"
            ],
            "Resource": [
                "arn:aws:iam::*:mfa/${aws:username}",
                "arn:aws:iam::*:user/${aws:username}"
            ]
        },
        {
            "Sid": "DenyAllExceptListedIfNoMFA",
            "Effect": "Deny",
            "NotAction": [
                "iam:CreateVirtualMFADevice",
                "iam:EnableMFADevice",
                "iam:GetUser",
                "iam:ListMFADevices",
                "sts:GetSessionToken"
            ],
            "Resource": "*",
            "Condition": {
                "BoolIfExists": {
                    "aws:MultiFactorAuthPresent": "false"
                }
            }
        }
    ]
}
```

---
<img width="1897" height="649" alt="upload10" src="https://github.com/user-attachments/assets/4a1c2aee-b6f4-4909-9f55-99c0d6582396" />
<img width="1919" height="934" alt="upload11" src="https://github.com/user-attachments/assets/0fc1d122-2cdb-4399-b04a-97e8017e65e8" />



#### Enable Default Encryption on an S3 Bucket

```
S3 → Select bucket → Properties tab → Default encryption → Edit
  Server-side encryption: Enable
  Encryption type: SSE-S3 (Amazon S3-managed keys) [free]
  OR
  Encryption type: SSE-KMS (AWS KMS-managed keys) [more control]
  AWS KMS key: aws/s3 (default AWS-managed key)
→ Save changes
```

Now **every object uploaded to this bucket is automatically encrypted at rest**.

#### Enforce Encryption via Bucket Policy (Deny Unencrypted Uploads)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyUnencryptedObjectUploads",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::my-secure-bucket/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption": "AES256"
                }
            }
        },
        {
            "Sid": "DenyNonSSLRequests",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::my-secure-bucket",
                "arn:aws:s3:::my-secure-bucket/*"
            ],
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        }
    ]
}
``` 
### Phase 1: Create the Target Bucket and Apply the Policy
Before you can test the block, you need a bucket with the strict policy applied.
1. Go to the **S3 Dashboard** in your AWS browser console.
2. Click **Create bucket** and name it something unique (e.g., `kshitiz-secure-bucket-999`).
3. Once created, click on the bucket, go to the **Permissions** tab, scroll down to **Bucket policy**, and click **Edit**.
4. Paste the JSON policy from the guide (the one that denies unencrypted uploads), making sure to change `my-secure-bucket` to your actual bucket name (`kshitiz-secure-bucket-999`) in both resource lines. Click **Save**.

### Phase 2: Create the Test File in PuTTY
Now, bring up your black PuTTY terminal window. You need a file to upload. You can create a simple text file instantly using the `echo` command.

Type this and hit Enter:
```bash
echo "This is top secret encrypted data." > testfile.txt
```
*(You can verify it worked by typing `ls` to see the file listed, or `cat testfile.txt` to read what you just wrote).*

### Phase 3: Run the Encryption Tests

Now you can run the exact commands from the guide. Just remember to swap out `my-secure-bucket` for the actual name of the bucket you just created.

**Test 1: The "Lazy" Upload (Should Fail)**
Try to upload the file normally, without telling AWS to encrypt it:
```bash
aws s3 cp testfile.txt s3://kshitiz-secure-bucket-999/
```
* **Result:** You should get an `upload failed ... AccessDenied` error. This means your bucket policy successfully blocked the unencrypted file!

**Test 2: The Secure Upload (Should Succeed)**
Now, tell AWS to encrypt the file using the `--sse AES256` flag:
```bash
aws s3 cp testfile.txt s3://kshitiz-secure-bucket-999/ --sse AES256
```
* **Result:** You should see `upload: testfile.txt to s3://...`. The upload was allowed because you followed the security rule.

**Test 3: Verify the Encryption**
To prove to yourself that the file is actually encrypted sitting in the bucket, ask AWS to read the file's metadata:
```bash
aws s3api head-object --bucket kshitiz-secure-bucket-999 --key testfile.txt
```
* **Result:** It will spit out a block of JSON text. Look near the bottom of that text, and you will see `"ServerSideEncryption": "AES256"`. 

Your test is complete, and your bucket is successfully enforcing data security!
**Test the Encryption Enforcement:**
```bash
# Upload without encryption header (should fail with our policy)
aws s3 cp testfile.txt s3://my-secure-bucket/ 
# Error: AccessDenied (no encryption specified)

# Upload with encryption header (should succeed)
aws s3 cp testfile.txt s3://my-secure-bucket/ \
  --sse AES256
# Success: upload: testfile.txt to s3://my-secure-bucket/testfile.txt

# Verify object is encrypted
aws s3api head-object \
  --bucket my-secure-bucket \
  --key testfile.txt
# Output shows: "ServerSideEncryption": "AES256"
```
<img width="819" height="523" alt="image" src="https://github.com/user-attachments/assets/c17353a2-aeea-4c5f-87fd-c43aafba1676" />
<img width="1906" height="863" alt="upload12" src="https://github.com/user-attachments/assets/a911ea90-2c3a-43b2-98d2-a5b92a244bfb" />

<img width="827" height="507" alt="image" src="https://github.com/user-attachments/assets/c71ceb7e-52d0-4981-9938-45b25df4116e" />


#### Enable EBS Volume Encryption (EC2 Disk Encryption)

**For new volumes:**
```
EC2 → Volumes → Create volume →
  ✅ Encrypted
  KMS key: (default) aws/ebs
```
<img width="1899" height="249" alt="upload13" src="https://github.com/user-attachments/assets/93bf0b24-9899-4798-8e62-98d1f75f3a75" />

---

### Step D — Set Up a Firewall Rule and Test Functionality

#### Security Group Rules (Instance-Level Firewall)

**Create a restrictive web server security group:**

```
EC2 → Security Groups → Create security group

Name:         webserver-strict-sg
Description:  Strict rules for production web server
VPC:          your-vpc

Inbound rules:
  HTTP    | TCP | 80   | 0.0.0.0/0     | Allow public HTTP
  HTTPS   | TCP | 443  | 0.0.0.0/0     | Allow public HTTPS  
  SSH     | TCP | 22   | 203.0.113.0/24| Allow SSH from office IP range only
  
Outbound rules:
  HTTP    | TCP | 80   | 0.0.0.0/0     | Allow outbound HTTP (updates)
  HTTPS   | TCP | 443  | 0.0.0.0/0     | Allow outbound HTTPS
  DNS     | UDP | 53   | 0.0.0.0/0     | Allow DNS
  (Remove the default "All traffic" outbound rule)

→ Create security group
```
<img width="1593" height="747" alt="image" src="https://github.com/user-attachments/assets/e9c37735-f208-494f-99bb-211bfe20f86d" />

**Test the Security Group:**
```bash
# Test HTTP access (should work if web server running)
curl http://<ec2-public-ip>
```
<img width="820" height="396" alt="image" src="https://github.com/user-attachments/assets/485f2dba-1439-4765-ad72-354d5de7a403" />
```
# Test SSH from your IP (should work)
ssh -i keypair.pem ec2-user@<ec2-public-ip>

# Test a port that's NOT in the rules (should timeout)
nc -zv <ec2-public-ip> 8080
# Connection timed out (blocked by security group)

# Test from unauthorized IP for SSH (should be blocked)
# From a different IP than what's in the rule:
ssh ec2-user@<ec2-public-ip>
# ssh: connect to host X.X.X.X port 22: Connection timed out
```

#### Network ACL Rules (Subnet-Level Firewall)

```
VPC → Network ACLs → Create network ACL
  Name:  webserver-nacl
  VPC:   your-vpc
→ Create

Inbound rules (add):
  Rule 100 | HTTP  | TCP | 80  | 0.0.0.0/0 | ALLOW
  Rule 110 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW
  Rule 120 | SSH   | TCP | 22  | 203.0.113.0/24 | ALLOW
  Rule 130 | Custom | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW (return traffic!)
  Rule *   | All traffic | | | DENY (default)

Outbound rules (add):
  Rule 100 | All traffic | All | All | 0.0.0.0/0 | ALLOW
  Rule *   | All traffic | | | DENY (default)

Associate with subnet:
  NACL → Subnet associations → Edit → Select your public subnet
```

> ⚠️ **NACL is stateless** — you must explicitly allow return traffic (ephemeral ports 1024-65535). Security Groups are stateful — they automatically allow return traffic.

---

## 📊 Security Layers Summary

```
┌────────────────────────────────────────────────────────┐
│              AWS Security Defense in Depth             │
│                                                        │
│  Layer 1: AWS Account Security                         │
│   → MFA on root, IAM users/roles, CloudTrail           │
│                                                        │
│  Layer 2: Network Security                             │
│   → VPC, Network ACLs, Security Groups                 │
│                                                        │
│  Layer 3: Host Security                                │
│   → OS patches, host-based firewall (iptables)         │
│                                                        │
│  Layer 4: Application Security                         │
│   → WAF (Web Application Firewall), input validation   │
│                                                        │
│  Layer 5: Data Security                                │
│   → Encryption at rest (S3 SSE, EBS encryption)        │
│   → Encryption in transit (TLS/SSL)                    │
│                                                        │
│  Layer 6: Detection & Response                         │
│   → CloudWatch Alarms, GuardDuty, CloudTrail           │
└────────────────────────────────────────────────────────┘
```

---

## 🔐 IAM Best Practices Checklist

```
Root Account:
  ✅ Enable MFA on root account
  ✅ Never use root for daily tasks
  ✅ Delete root access keys

IAM Users:
  ✅ Create individual IAM users (no shared accounts)
  ✅ Use groups to assign permissions
  ✅ Enforce MFA for all users
  ✅ Rotate access keys every 90 days

Permissions:
  ✅ Apply least privilege principle
  ✅ Use IAM roles instead of access keys for EC2
  ✅ Review and remove unused permissions (use IAM Access Analyzer)
  ✅ Use permission boundaries for delegated admins

Monitoring:
  ✅ Enable CloudTrail in all regions
  ✅ Set up Config Rules for compliance
  ✅ Enable GuardDuty for threat detection
```

---

## ✅ Learning Outcomes

After completing this practical, you should be able to:

- [ ] Explain the Shared Responsibility Model
- [ ] Create IAM users, groups, roles, and policies
- [ ] Write custom IAM policies using JSON
- [ ] Apply the principle of least privilege
- [ ] Enforce MFA using IAM policies
- [ ] Configure S3 bucket encryption (SSE-S3 and SSE-KMS)
- [ ] Create and apply bucket policies to enforce encryption and HTTPS
- [ ] Configure Security Groups with restrictive rules
- [ ] Understand the difference between Security Groups and NACLs
- [ ] Test firewall rules and verify access is correctly blocked/allowed

---

## 📚 Further Reading

- [AWS IAM User Guide](https://docs.aws.amazon.com/iam/latest/userguide/)
- [AWS Security Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
- [AWS GuardDuty](https://docs.aws.amazon.com/guardduty/latest/ug/what-is-guardduty.html)
- [AWS Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html)
- [OWASP Cloud Security Guidelines](https://owasp.org/www-project-cloud-security/)
