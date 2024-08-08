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

JSON object representing the Dolus module, following the Dolus module schema.

</details>

<details>
<summary><strong>Responses</strong></summary>

| Status | Description           | Body                                              |
|--------|-----------------------|---------------------------------------------------|
| 200    | Successful Signing    | `{ "error": false, "message": "Module signed successfully", "id": "<module_uuid>", "signedModule": "<signed_module_json>" }` |
| 422    | Validation Error      | See [Validation Error Response](#validation-error-response) |
| 400    | Other Errors          | `{ "error": true, "message": "Error description" }` |

</details>

<details>
<summary><strong>Usage Example (cURL)</strong></summary>

```sh
curl -X POST https://api.dolus.app/v1/module/sign \
  -H "Authorization: Bearer dol.your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "example-module",
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

### Module Validation

`POST /v1/module/validate`

Validates a Dolus module without signing it.

<details>
<summary><strong>Request Body</strong></summary>

JSON object representing the Dolus module, following the Dolus module schema.

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
    "name": "example-module",
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
      "instancePath": "/actions/0/value",
      "schemaPath": "#/definitions/modifyRegistryEntry/properties/value/type",
      "keyword": "type",
      "params": {
        "type": "string"
      },
      "message": "must be string"
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
          
          MODULE_ID=$(echo "$BODY" | jq -r .id)
          echo "$BODY" | jq -r .signedModule > "$MODULE_ID.json"
          echo "module_id=$MODULE_ID" >> $GITHUB_ENV

      - name: Release
        uses: ncipollo/release-action@v1
        with:
          artifacts: "${{ env.module_id }}.json"
          artifactContentType: application/json
          bodyFile: "CHANGELOG.md"
          token: ${{ secrets.GITHUB_TOKEN }}
          allowUpdates: true
          artifactErrorsFailBuild: true
          makeLatest: true
          name: "Release ${{ github.ref_name }}"
          tag: ${{ github.ref }}
```

This action:
1. Triggers on pushes of tags starting with 'v'
2. Signs the module using the Dolus API, with error checking
3. Creates a new GitHub release using `ncipollo/release-action@v1`
4. Uploads the signed module as a release asset
5. Renames the artifact using the module ID
6. Uploads the renamed artifact to the workflow artifacts

**Notes:** 
- Set the `DOLUS_API_KEY` secret in your GitHub repository settings.
- Ensure you have a `CHANGELOG.md` file in your repository root for the release notes.
- The `permissions: contents: write` is required to create releases.
- The action checks for both HTTP status codes and the `error` field in the API response.

</details>

## Notes

- Keep your API key secure and never commit it to your repository.
- Use GitHub secrets or similar secure methods to manage sensitive information in your CI/CD pipelines.
