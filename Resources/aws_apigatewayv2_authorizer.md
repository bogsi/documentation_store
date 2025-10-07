# aws_apigatewayv2_authorizer — Terraform reference (detailed)

> Purpose: Manage an API Gateway v2 (HTTP/WebSocket) Authorizer in AWS. This document is an original, detailed reference modeled after common Terraform provider docs. It explains the resource’s configuration, arguments, attributes, nested blocks, typical examples, import syntax, and troubleshooting tips.

## Quick summary

- Resource: aws_apigatewayv2_authorizer
- Use: Create or manage authorizers for API Gateway v2 (HTTP API / WebSocket API).
- Common authorizer types: REQUEST (custom Lambda or external) and JWT (JSON Web Token validation performed by API Gateway).

## Contract (short)

- Inputs: HCL arguments described below (api_id, name, authorizer_type, jwt_configuration, authorizer_uri, identity_sources, etc.).
- Outputs: The created authorizer is identified by an authorizer ID; the resource exposes attributes like `authorizer_id`, `authorizer_uri`, `jwt_configuration`, and `identity_sources`.
- Error modes: AWS API validation errors (missing required fields), IAM permission errors, invalid JWT configuration or malformed identity source strings.
- Success criteria: Authorizer exists in API Gateway and returns a stable `id` that can be referenced by integrations or routes.

## Usage notes and high-level flow

- Choose authorizer type early: `JWT` authorizers are validated by API Gateway against an issuer and audience; `REQUEST` authorizers are usually Lambda-based (or other integration) that receive the raw request and return allow/deny decisions.
- `api_id` ties the authorizer to a specific API Gateway v2 instance.
- For `REQUEST` authorizers you typically supply: `authorizer_uri` (invocation URI for the Lambda), `identity_sources` (where to extract auth info), and optionally an IAM role (`authorizer_credentials_arn`) if API Gateway needs permission to invoke the authorizer.
- For `JWT` authorizers you supply a `jwt_configuration` block with `issuer` and one-or-more `audience` values; `identity_sources` indicates which request values contain the token.

---

## Resource syntax (conceptual)

This is the conceptual HCL form of the resource; examples below provide real usage.

```hcl
resource "aws_apigatewayv2_authorizer" "example" {
  api_id           = "..."
  name             = "my-authorizer"
  authorizer_type  = "JWT" # or "REQUEST"

  jwt_configuration {
    issuer   = "https://issuer.example.com/"
    audience = ["audience-1", "audience-2"]
  }

  # REQUEST (Lambda authorizer) fields
  authorizer_uri                 = aws_lambda_function.auth.invoke_arn
  authorizer_credentials_arn     = aws_iam_role.invoke_role.arn
  identity_sources               = ["$request.header.Authorization"]

  enable_simple_responses = true
}
```

---

## Arguments (fields)

Below are the commonly used fields and their descriptions. Marked as (required) or (optional) where appropriate.

- api_id (required)
  - Type: string
  - Description: The id of the API (API Gateway v2) to attach this authorizer to. This is the same `id` you get from the `aws_apigatewayv2_api` resource.

- name (required)
  - Type: string
  - Description: A friendly name for the authorizer. Used to identify the authorizer in the AWS Console and API.

- authorizer_type (required)
  - Type: string
  - Valid values: `REQUEST`, `JWT`
  - Description: The type of the authorizer. `REQUEST` means API Gateway forwards parts of the request to a custom authorizer (Lambda) for a decision. `JWT` means API Gateway itself validates a JWT token using an issuer and audience.

- authorizer_uri (optional; required for many REQUEST authorizers)
  - Type: string
  - Description: For `REQUEST` authorizers this is the invocation URI (usually a Lambda function URI) that API Gateway calls to run the authorizer logic. When using Lambda, you can pass the function's `arn` or the `arn:aws:apigateway:...:lambda:path/2015-03-31/functions/{function-arn}/invocations` form if needed by your setup.

- authorizer_credentials_arn (optional)
  - Type: string
  - Description: ARN of an IAM role that API Gateway assumes when invoking the authorizer. Use this when your authorizer needs cross-account permissions or specific execution permissions.

- identity_sources (required for REQUEST; recommended for JWT)
  - Type: list(string)
  - Description: A list of request parameter sources that API Gateway should pass to or use for authentication. Common examples: `"$request.header.Authorization"`, `"$request.querystring.auth"`, `"$request.path.param"`. For JWT authorizers, identity_sources typically point to the header or query parameter containing the JWT.

- enable_simple_responses (optional)
  - Type: bool
  - Default: false
  - Description: When enabled for `REQUEST` authorizers, allows the authorizer Lambda to return a simplified response format that API Gateway understands and converts to an HTTP response (useful for simple allow/deny handling).

### jwt_configuration (block)

- This nested block is used only when `authorizer_type = "JWT"`.

  - issuer (required inside jwt_configuration)
    - Type: string
    - Description: The token issuer (iss) value. Typically the base URL of your identity provider (for example, Auth0, Cognito user pool issuer, or another OpenID Connect issuer).

  - audience (required inside jwt_configuration)
    - Type: list(string)
    - Description: One or more audience (`aud`) values expected in the JWT. API Gateway will validate the token's `aud` claim against this list.


---

## Attributes (exported / read-only)

After creation, Terraform exposes several attributes for use elsewhere in your configuration:

- id / authorizer_id — The resource ID assigned by AWS (commonly used when importing or referencing the authorizer).
- api_id — The API this authorizer belongs to (echo of the input value).
- name — The authorizer name.
- authorizer_type — The configured type (`REQUEST` or `JWT`).
- authorizer_uri — The configured invocation URI (if set).
- authorizer_credentials_arn — ARN of the role used by API Gateway to invoke the authorizer.
- identity_sources — The list of identity source strings.
- jwt_configuration.* — A nested object exposing the `issuer` and `audience` (if configured).
- enable_simple_responses — Boolean if configured.

(Note: exact attribute names and behavior can vary slightly across provider versions; use `terraform show` or `terraform state show` to inspect live state.)

---

## Examples

Below are two compact, practical examples you can adapt: a JWT authorizer and a REQUEST (Lambda) authorizer.

### Example A — JWT authorizer (typical OIDC provider)

This example shows how to configure a JWT authorizer that validates tokens from an OpenID Connect issuer.

```hcl
resource "aws_apigatewayv2_api" "example_api" {
  name          = "example-http-api"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_authorizer" "jwt_auth" {
  api_id          = aws_apigatewayv2_api.example_api.id
  name            = "jwt-authorizer"
  authorizer_type = "JWT"

  identity_sources = [
    "$request.header.Authorization"
  ]

  jwt_configuration {
    issuer   = "https://auth.example.com/"
    audience = ["api://default"]
  }
}
```

Usage on a route or integration typically references the authorizer id or name in the route's `authorization_type/authorizer_id` settings.

### Example B — REQUEST authorizer using Lambda

This example shows a Lambda-based authorizer that receives the incoming request and returns allow/deny.

```hcl
resource "aws_iam_role" "lambda_invoke_role" {
  name = "api-gateway-authorizer-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "apigateway.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_lambda_function" "auth" {
  filename         = "lambda_auth.zip"
  function_name    = "myAuthorizer"
  handler          = "handler.main"
  runtime          = "python3.9"
  role             = aws_iam_role.lambda_exec.arn

  # ...other configuration such as source code
}

resource "aws_apigatewayv2_authorizer" "request_auth" {
  api_id                      = aws_apigatewayv2_api.example_api.id
  name                        = "lambda-request-authorizer"
  authorizer_type             = "REQUEST"
  authorizer_uri              = aws_lambda_function.auth.invoke_arn
  authorizer_credentials_arn  = aws_iam_role.lambda_invoke_role.arn

  identity_sources = [
    "$request.header.Authorization"
  ]

  enable_simple_responses = true
}
```

Notes on the Lambda URI: some setups require a special URI format for API Gateway to invoke Lambda (see provider or AWS Console). If you encounter invocation permission errors, ensure the Lambda has an appropriate resource-based policy and that `authorizer_credentials_arn` has the right trust policy for API Gateway.

---

## Import

Authorizers can be imported into Terraform state. The import id format is typically a combination of the API id and the authorizer id. Examples:

- Import by API and Authorizer id:

```bash
terraform import aws_apigatewayv2_authorizer.example <api-id>/<authorizer-id>
```

After import, run `terraform plan` to reconcile the imported resource with your configuration and add any missing required attributes to your HCL.

---

## Edge cases, tips and gotchas

- Mismatched authorizer type and fields:
  - If you set `authorizer_type = "JWT"` but also supply `authorizer_uri`, AWS will reject the configuration. Keep JWT and REQUEST fields separate.

- Identity source strings:
  - Use the exact request reference format API Gateway expects (for HTTP APIs this is typically `$request.header.<name>` or `$request.querystring.<name>`). Typos here will cause the authorizer not to find the token.

- Lambda permissions:
  - When using Lambda as a REQUEST authorizer, ensure the Lambda has a resource-based permission allowing API Gateway to invoke it. Terraform has `aws_lambda_permission` to add this.

- Caching & propagation:
  - API Gateway changes sometimes take a few seconds to propagate. When creating many dependent resources in automation, add appropriate retries or small delays if you see transient not-found or permission errors.

- Provider version differences:
  - Field names, defaults and behavior can change across provider versions. If you rely on a specific behavior, pin the `hashicorp/aws` provider to a known version in your configuration.

---

## Troubleshooting checklist

1. If `terraform apply` fails with an AWS validation error, inspect the error message — it often indicates which field is invalid or missing.
2. If API Gateway returns 401/403 when testing integrations using the authorizer:
   - Verify tokens are issued by the configured `issuer` and contain the expected `aud` claim (for JWT).
   - Verify `identity_sources` correctly points to the header/query param that contains the token.
3. If using a Lambda authorizer and API Gateway fails to invoke it:
   - Ensure `aws_lambda_permission` grants `apigateway.amazonaws.com` permission to invoke the function.
   - Confirm `authorizer_credentials_arn` (if used) is a role that API Gateway can assume.

---

## Quick reference checklist

- Required fields: `api_id`, `name`, `authorizer_type`.
- JWT-specific: use `jwt_configuration { issuer, audience }` and set appropriate `identity_sources`.
- REQUEST-specific: set `authorizer_uri` and `identity_sources`; consider `authorizer_credentials_arn` and `enable_simple_responses`.
- Import syntax: `<api-id>/<authorizer-id>`.

---

## Where to look next

- AWS docs on HTTP API authorizers (Jwt vs Request) for the most current runtime behavior.
- Terraform provider `hashicorp/aws` changelog for any provider-specific changes.

---

If you'd like, I can also:

- Add fully working example projects with the minimal Lambda code, IAM policies, and `aws_lambda_permission` wiring.
- Convert these examples into a Terraform module that manages both authorizer types and exposes common variables.


