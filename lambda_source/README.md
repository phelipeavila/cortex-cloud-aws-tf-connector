# Lambda Source: Cortex XDR Organization Connector Registration

## Purpose

This Lambda function registers an AWS Organization with Cortex XDR by sending a JSON payload to a signed Google Cloud Storage URL. It is invoked once during `terraform apply` via `aws_lambda_invocation` and is **not** called again on subsequent runs (the invocation uses `lifecycle { ignore_changes = [input] }`).

Unlike the account connector Lambda, this handler also queries the AWS Organizations API to retrieve the Organization ID (`csp_org_id`), which is included in the registration payload.

## How It Works

1. Terraform creates the Lambda function and immediately invokes it with the connector registration data.
2. The handler extracts account details, IAM role ARNs, organization ID, and module permissions from the event.
3. It calls `boto3.client("organizations").describe_organization()` to get the AWS Organization ID (`csp_org_id`).
4. It constructs a JSON payload and sends an HTTPS PUT request to the Cortex registration endpoint.
5. On success (HTTP 200), it returns the result to Terraform.
6. On failure, it retries up to 5 times with a 3-second delay between attempts.

## Adaptation from CloudFormation

This code was adapted from the original CloudFormation Custom Resource Lambda embedded in the Cortex XDR YAML template (`Code.ZipFile`). The two handlers produce **identical HTTP requests** to the Cortex API, but differ in how they interface with their respective orchestrators.

### What changed

| Aspect | CloudFormation (original) | Terraform (this file) |
|--------|--------------------------|----------------------|
| Event structure | Data nested under `event["ResourceProperties"]` | Data at `event` root level |
| Data extraction | `pop()` keys from ResourceProperties dict inside an `if count == 1` guard (pop mutates the dict, so it must only run once) | `get()` with defaults before the loop (non-destructive, no guard needed) |
| `resources_data` | Residual of the ResourceProperties dict after popping known keys | Explicitly constructed from `audit_logs_byob` and `outpost_scanner` (the same two keys that remain after pops) |
| Response mechanism | `cfnresponse.send()` HTTP callback to a CloudFormation-provided URL | `return dict` (Terraform captures the return value) |
| Delete handling | Required — responds to `RequestType: Delete` or the CF stack hangs | Not applicable — Terraform handles resource destruction directly |
| `provisioning_method` | Hardcoded `"CF"` in the data dict | Hardcoded `"CF"` in the data dict (must stay `"CF"`) |

### What stayed the same

- The JSON payload sent to the Cortex API is identical.
- Retry logic: 5 attempts, 3-second sleep, 10-second HTTP timeout.
- SSL: uses `ssl.create_default_context()` with system CA certificates.
- HTTP method: PUT with `Content-Type: application/json`.
- `provisioning_method` must be `"CF"` — the Cortex backend expects this value regardless of the provisioning tool.
- `boto3.client("organizations").describe_organization()` call to get `csp_org_id` — preserved from the original CloudFormation Lambda.
- `organization_id` field populated from the event input.

## Event Input

The Lambda receives its input from `aws_lambda_invocation` in `lambda.tf`. The event contains:

| Key | Type | Source |
|-----|------|--------|
| `AccountId` | string | `data.aws_caller_identity.current.account_id` |
| `RoleArn` | string | `aws_iam_role.cortex_platform.arn` |
| `ExternalID` | string | `var.external_id` |
| `TemplateId` | string | `var.template_id` |
| `UploadOutputUrl` | string | `var.upload_output_url` (signed GCS URL) |
| `TemplateVersion` | object | `local.template_version` |
| `ConnectorId` | string | `var.connector_id` |
| `OrganizationId` | string | `var.organizational_unit_id` (OU root, e.g. `r-xxxx`) |
| `ModulePermissionScope` | object | Policy ARNs for each Cortex module |
| `audit_logs_byob` | object/null | SQS URL, reader role ARN, audience (null when CloudTrail disabled) |
| `outpost_scanner` | object | Scanner role ARN |
| `provisioning_method` | string | `"CF"` (must not be changed) |

## Return Value

On success:
```json
{"Success": true, "Upload Status Code": 200, "Resources Data": {...}}
```

On failure (after all retries):
```json
{"Success": false, "error": "...", "Status": "Failed to send"}
```

The return value is captured by Terraform and exposed via the `registration_result` output.

A `postcondition` on the `aws_lambda_invocation` resource checks the `Success` field. If `Success` is not `true`, **`terraform apply` fails** with a clear error message that includes the failure reason from the Lambda. This ensures a broken registration is never silently accepted.

## Dependencies

- Python 3.12 standard library: `json`, `urllib.request`, `ssl`, `time`
- `boto3` (included in AWS Lambda runtime): Used to call `organizations:DescribeOrganization` to retrieve the AWS Organization ID (`csp_org_id`).

The Lambda execution role must have `AWSOrganizationsReadOnlyAccess` attached for the `describe_organization()` call to succeed.
