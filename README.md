# Dolus Module API Documentation

## Table of Contents

- [Introduction](#introduction)
- [Authentication](#authentication)
- [Endpoints](#endpoints)
  - [Module Signing](#module-signing)
  - [Module Validation](#module-validation)
- [GitHub Action for CI](#github-action-for-ci)
- [Notes](#notes)

## Introduction

Dolus modules are typically authored and published via the [Dolus Playground](https://play.dolus.app). This API provides advanced users and automation tools with endpoints for module signing and validation.

Dolus modules now require a unique identifier (`id`) following a package-like format and a version number adhering to semantic versioning principles. These changes enhance module management and version control within the Dolus ecosystem.

<details>
<summary><strong>Why We Sign and Verify Modules</strong></summary>

Module signing ensures:
1. **Authenticity**: Confirm trusted sources
2. **Integrity**: Detect tampering
3. **Non-repudiation**: Authors can't deny creation

</details>

<details>
<summary><strong>Provenance and GitHub Requirement</strong></summary>

We require users to link their GitHub account and use a public GitHub email for module authorship. This establishes a clear chain of provenance, enhancing trust and accountability in the Dolus community.

</details>

## Authentication

All endpoints require API key authentication.

<details>
<summary><strong>Getting an API Key</strong></summary>

1. Visit [https://my.dolus.app](https://my.dolus.app)
2. Sign in and link your GitHub account
3. Navigate to the API Keys section
4. Generate a new API key

**Keep your API key secure and do not share it publicly.**

</details>

### Headers

| Header          | Value                       |
|-----------------|---------------------------- |
| Content-Type    | application/json            |
| Authorization   | Bearer dol.<your_api_key>   |

## Endpoints

Base URL: `https://api.dolus.app`

### Module Signing

`POST /v1/module/sign`

Signs a Dolus module after schema and AST validation.

<details>
<summary><strong>Request Body</strong></summary>

JSON object representing the Dolus module, following the Dolus module schema. Key requirements include:

- `id`: Must follow a package-like format (e.g., `@scope/module-name` or `module-name`)
- `version`: Must be included and follow semantic versioning (e.g., `1.0.0`)
- `name`: The name of the module (non-empty string)
- `description`: A description of the module's purpose (non-empty string)
- `actions`: An array of actions to be performed by the module (at least one action required)

Note: The `author` field will be automatically populated based on the authenticated user's GitHub information.

</details>

<details>
<summary><strong>Responses</strong></summary>

| Status | Description           | Body                                              |
|--------|-----------------------|---------------------------------------------------|
| 200    | Successful Signing    | `{ "error": false, "message": "Module signed successfully", "id": "<module_id>", "version": "<module_version>", "module": "<base64_encoded_module>" }` |
| 422    | Validation Error      | See [Validation Error Response](#validation-error-response) |
| 400    | Other Errors          | `{ "error": true, "message": "Error description" }` |
| 500    | Database Error        | `{ "error": true, "message": "An internal database error occurred. Please try again later." }` |

</details>

<details>
<summary><strong>Usage Example (cURL)</strong></summary>

```sh
curl -X POST https://api.dolus.app/v1/module/sign \
  -H "Authorization: Bearer dol.your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "@example-org/demo-module",
    "version": "1.0.0",
    "name": "Example Module",
    "description": "This is an example module",
    "actions": [
      {
        "name": "create-directory",
        "path": "%AppData%\\example\\"
      }
    ]
  }'
```

</details>

<details>
<summary><strong>Important Notes</strong></summary>

1. The `id` field must be unique or owned by the authenticated user.
2. The `version` must be greater than any previously published version for that module ID.
3. Certain scopes (e.g., `@example`, `@official`, `@andrew`, `@example-org`) are reserved and cannot be used.
4. The `author` field will be automatically populated based on the authenticated user's GitHub information.
5. The maximum payload size is limited. Ensure your module doesn't exceed this limit.
6. The response includes a `module` field containing the base64-encoded signed module data.

</details>


### Module Validation

`POST /v1/module/validate`

Validates a Dolus module without signing it.

<details>
<summary><strong>Request Body</strong></summary>

JSON object representing the Dolus module, following the Dolus module schema. Key requirements include:

- `id`: Must follow a package-like format (e.g., `@scope/module-name` or `module-name`)
- `version`: Must be included and follow semantic versioning (e.g., `1.0.0`)
- `name`: The name of the module (non-empty string)
- `description`: A description of the module's purpose (non-empty string)
- `author`: Information about the module's author (will be validated but not used in the actual signing process)
- `actions`: An array of actions to be performed by the module (at least one action required)

</details>

<details>
<summary><strong>Responses</strong></summary>

| Status | Description           | Body                                              |
|--------|-----------------------|---------------------------------------------------|
| 204    | Successful Validation | Empty                                             |
| 422    | Validation Error      | See [Validation Error Response](#validation-error-response) |
| 400    | Other Errors          | `{ "error": true, "message": "Error description" }` |

</details>

<details>
<summary><strong>Usage Example (cURL)</strong></summary>

```sh
curl -X POST https://api.dolus.app/v1/module/validate \
  -H "Authorization: Bearer dol.your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "@example-org/demo-module",
    "version": "1.0.0",
    "name": "Example Module",
    "description": "This is an example module",
    "author": {
      "name": "John Doe",
      "email": "john.doe@example.com",
      "github": "https://github.com/johndoe"
    },
    "actions": [
      {
        "name": "create-directory",
        "path": "%AppData%\\example\\"
      }
    ]
  }'
```

</details>

#### Validation Error Response

<details>
<summary><strong>Schema Validation Error</strong></summary>

```json
{
  "error": true,
  "message": "Invalid module payload: Schema validation failed",
  "schemaErrors": [
    {
      "instancePath": "/id",
      "schemaPath": "#/properties/id/pattern",
      "keyword": "pattern",
      "params": {
        "pattern": "^(?:@(?:[a-z0-9-*~][a-z0-9-*._~]*)?/)?[a-z0-9-~][a-z0-9-._~]*$"
      },
      "message": "must match pattern \"^(?:@(?:[a-z0-9-*~][a-z0-9-*._~]*)?/)?[a-z0-9-~][a-z0-9-._~]*$\""
    },
    {
      "instancePath": "/version",
      "schemaPath": "#/properties/version/pattern",
      "keyword": "pattern",
      "params": {
        "pattern": "^(0|[1-9]\\d*)\\.(0|[1-9]\\d*)\\.(0|[1-9]\\d*)(?:-((?:0|[1-9]\\d*|\\d*[a-zA-Z-][0-9a-zA-Z-]*)(?:\\.(?:0|[1-9]\\d*|\\d*[a-zA-Z-][0-9a-zA-Z-]*))*))?(?:\\+([0-9a-zA-Z-]+(?:\\.[0-9a-zA-Z-]+)*))?$"
      },
      "message": "must match pattern \"^(0|[1-9]\\d*)\\.(0|[1-9]\\d*)\\.(0|[1-9]\\d*)(?:-((?:0|[1-9]\\d*|\\d*[a-zA-Z-][0-9a-zA-Z-]*)(?:\\.(?:0|[1-9]\\d*|\\d*[a-zA-Z-][0-9a-zA-Z-]*))*))?(?:\\+([0-9a-zA-Z-]+(?:\\.[0-9a-zA-Z-]+)*))?$\""
    }
  ]
}
```

</details>

<details>
<summary><strong>AST Validation Error</strong></summary>

```json
{
  "error": true,
  "message": "Invalid module payload: AST validation failed",
  "astErrors": [
    {
      "severity": 8,
      "message": "Invalid action type",
      "startLineNumber": 10,
      "startColumn": 5,
      "endLineNumber": 10,
      "endColumn": 20
    }
  ]
}
```

| Severity | Meaning |
|----------|---------|
| 1        | Hint    |
| 2        | Info    |
| 4        | Warning |
| 8        | Error   |

</details>

## GitHub Action for CI

<details>
<summary><strong>Example GitHub Action</strong></summary>

```yaml
name: Sign and Release Dolus Module

on:
  push:
    tags:
      - 'v*'

jobs:
  sign-and-release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v3

      - name: Sign Dolus Module
        id: sign_module
        run: |
          RESPONSE=$(curl -s -w "%{http_code}" -X POST https://api.dolus.app/v1/module/sign \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer ${{ secrets.DOLUS_API_KEY }}" \
            -d @module.json)
          HTTP_STATUS=$(echo $RESPONSE | tail -n1)
          BODY=$(echo $RESPONSE | sed '$ d')
          
          if [ $HTTP_STATUS -ne 200 ]; then
            echo "Error: API request failed with status $HTTP_STATUS"
            echo "Response body: $BODY"
            exit 1
          fi
          
          if ! echo "$BODY" | jq -e '.error == false' > /dev/null; then
            echo "Error: API request failed"
            echo "Response body: $BODY"
            exit 1
          fi
          
          ID=$(echo "$BODY" | jq -r .id)
          VERSION=$(echo "$BODY" | jq -r .version)
          echo "$BODY" | jq -r .module | base64 --decode > "${ID}-${VERSION}.mod"

      - name: Release
        uses: ncipollo/release-action@v1
        with:
          artifacts: "${{ steps.sign_module.outputs.ID }}-${{ steps.sign_module.outputs.VERSION }}.mod"
          artifactContentType: application/octet-stream
          bodyFile: "CHANGELOG.md"
          token: ${{ secrets.GITHUB_TOKEN }}
          allowUpdates: true
          artifactErrorsFailBuild: true
          makeLatest: true
          name: "Release ${{ github.ref_name }}"
          tag: ${{ github.ref }}
```

**Notes:** 
- Ensure your `module.json` file includes the correct `id` and `version` fields before signing.
- The `version` in your module should match the Git tag (e.g., if the tag is `v1.0.0`, your module's version should be `1.0.0`).
- Set the `DOLUS_API_KEY` secret in your GitHub repository settings.
- Ensure you have a `CHANGELOG.md` file in your repository root for the release notes.
- The `permissions: contents: write` is required to create releases.

</details>
