# AWS IAM Role Enumeration & S3 Account Discovery Walkthrough

## Lab Overview

This lab demonstrates how an AWS IAM user can enumerate permissions, discover assumable roles, inspect role policies, and leverage the `s3-account-search` tool to identify the AWS account associated with an S3 bucket.

## Objective

Identify the AWS Account ID associated with the bucket:

```text
s3://img.cybrlabs.io
```

and obtain the lab flag.

---

# Step 1: Configure AWS Credentials

Configure the provided credentials:

```bash
aws configure
aws sts get-caller-identity
```

Output confirmed access as:

```text
arn:aws:iam::014498641646:user/S3User-AssumeRole
```

---

# Step 2: Enumerate IAM Roles

```bash
aws iam list-roles --query "Roles[?RoleName=='S3AccessImages']"
```

Discovered role:

```text
arn:aws:iam::014498641646:role/S3AccessImages
```

The trust policy showed that the current IAM user could assume the role.

---

# Step 3: Enumerate Inline Policies

```bash
aws iam list-role-policies --role-name S3AccessImages
aws iam get-role-policy --role-name S3AccessImages --policy-name AccessS3BucketObjects
```

The role allowed:

```text
s3:ListBucket
s3:GetObject
```

Against:

```text
arn:aws:s3:::img.cybrlabs.io
arn:aws:s3:::img.cybrlabs.io/*
```

---

# Step 4: Install s3-account-search

Ubuntu's protected Python environment prevented a direct installation with pip.

Install pipx:

```bash
sudo apt install pipx
```

Create a virtual environment:

```bash
python3 -m venv myenv
source myenv/bin/activate
```

Install the tool:

```bash
pip install s3-account-search
```

---

# Step 5: Discover the Bucket Owner Account

Run:

```bash
s3-account-search arn:aws:iam::014498641646:role/S3AccessImages s3://img.cybrlabs.io
```

Output:

```text
found: 7
found: 70
found: 703
found: 7036
found: 70367
found: 703671
found: 7036719
found: 70367192
found: 703671927
found: 7036719278
found: 70367192780
found: 703671927808
```

The complete AWS account ID was recovered.

---

# Flag

```text
703671927808
```

---

# Key Takeaways

- Enumerated IAM roles and trust policies.
- Identified permissions granted to an assumable role.
- Used `s3-account-search` to determine the owner account of an S3 bucket.
- Demonstrated how AWS IAM misconfigurations can expose useful reconnaissance information.
