# Vault → AWS RolesAnywhere Credentials

A GitHub Action that obtains temporary AWS credentials using short-lived 
certificates issued by Vault PKI for AWS IAM Roles Anywhere.

## Overview

This action automates the process of:
1. Requesting a short-lived certificate from HashiCorp Vault PKI
2. Using that certificate with AWS IAM Roles Anywhere to obtain temporary AWS credentials

## Prerequisites

- A configured Vault PKI engine with an appropriate issuer role
- An AWS IAM Roles Anywhere trust anchor configured to trust your Vault CA
- An AWS IAM Roles Anywhere profile
- An AWS IAM role that can be assumed via Roles Anywhere

## Usage

```yaml
- name: Get AWS Credentials
  uses: nacl-app/aws-roles-anywhere-action@v1
  with:
    vault-addr: https://vault.example.com
    vault-token: ${{ secrets.VAULT_TOKEN }}
    vault-engine: pki_int_aws_roles_anywhere
    vault-issuer-role: aws-roles-anywhere
    cert-cn: ci.iam-ra.example.com
    aws-trust-anchor-arn: arn:aws:rolesanywhere:us-east-1:123456789012:trust-anchor/abc123
    aws-profile-arn: arn:aws:rolesanywhere:us-east-1:123456789012:profile/def456
    aws-role-arn: arn:aws:iam::123456789012:role/CI
    aws-region: us-east-1

- name: Use AWS CLI
  run: aws sts get-caller-identity
```

## Inputs

| Input | Description | Required | Default | ENV Fallback |
|-------|-------------|----------|---------|--------------|
| `vault-addr` | Vault server address | Yes* | - | `VAULT_ADDR` |
| `vault-engine` | Vault PKI engine name | Yes* | - | `VAULT_ENGINE` |
| `vault-issuer-role` | Vault issuer role name | Yes* | - | `VAULT_ISSUER_ROLE` |
| `vault-token` | Vault token with permissions to issue certificates | Yes* | - | `VAULT_TOKEN` |
| `cert-cn` | Certificate common name | Yes* | - | `CERT_CN` |
| `aws-trust-anchor-arn` | AWS RolesAnywhere Trust Anchor ARN | Yes* | - | `AWS_TRUST_ANCHOR_ARN` |
| `aws-profile-arn` | AWS RolesAnywhere Profile ARN | Yes* | - | `AWS_PROFILE_ARN` |
| `aws-role-arn` | AWS IAM Role ARN to assume | Yes* | - | `AWS_ROLE_ARN` |
| `aws-region` | AWS region | No | `us-east-1` | `AWS_REGION` |

\* Required unless provided via environment variable

### Environment Variable Fallback

All inputs support fallback to environment variables from the GitHub Actions runner. If an input is not provided in the `with:` block, the action will automatically check for a corresponding environment variable. This is useful for:

- Setting runner-level defaults
- Simplifying workflows when using the same values across multiple jobs
- Managing configuration through GitHub environment variables or secrets

**Example using environment variables:**

```yaml
env:
  VAULT_ADDR: https://vault.example.com
  VAULT_ENGINE: pki_int_aws_roles_anywhere
  AWS_REGION: us-east-1

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Get AWS Credentials
        uses: nacl-app/aws-roles-anywhere-action@v1
        with:
          # Only specify non-default values
          vault-token: ${{ secrets.VAULT_TOKEN }}
          cert-cn: ci.iam-ra.example.com
          aws-trust-anchor-arn: ${{ secrets.AWS_TRUST_ANCHOR_ARN }}
          aws-profile-arn: ${{ secrets.AWS_PROFILE_ARN }}
          aws-role-arn: ${{ secrets.AWS_ROLE_ARN }}
          # vault-addr, vault-engine, and aws-region will use ENV variables
```

## How It Works

1. **Install Dependencies**: Installs required tools including `jq`, `curl`, and the AWS signing helper utility
2. **Issue Certificate**: Requests a short-lived certificate from Vault PKI (default TTL: 1 hour)
3. **Setup AWS Credentials**: Requests temporary AWS credentials via AWS Roles Anywhere using the certificate issued in the previous step

After this action runs, Any AWS tools/service will automatically use the temporary credentials obtained via Roles Anywhere.

## Security Considerations

- Certificates are short-lived (1 hour TTL by default)
- Private keys are generated during the workflow and never leave the runner
- All sensitive files (certificates, keys) are ignored by git (see `.gitignore`)
- Store your `vault-token` as a GitHub secret, never commit it

## Requirements

- GitHub Actions runner must be Linux-based (uses `apt-get`)
- Runner needs internet access to Vault and AWS services

## License

See LICENSE file for details.
