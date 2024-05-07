---
title: "Bulk User Imports"
date: "2019"
summary: "Created to address customer feedback about a lack of examples for importing huge amounts of user data."
weight: 10
hiddenInHomelist: true
params:
  sample:
    kind: "Tutorial"
    src: "Auth0"
---

This tutorial shows you how to bulk import user data into Auth0 using the [create import users job](##) endpoint. Bulk imports are useful for migrating users from an existing database or service to Auth0.

## Before you start

Before you launch the import users job:

* [Configure a database connection](##) to import the users into and enable it for at least one application.
* If you are importing passwords, make sure the passwords are hashed using one of the [supported algorithms](##). Users with passwords hashed by unsupported algorithms will need to reset their password when they log in for the first time after the bulk import.
* Make sure multi-factor authentication (MFA) enrollments you import are a [supported type](##): `email`, `phone`, or `totp`. 
* [Get a Management API Token](##) for job endpoint requests.

## Create users JSON file

First you'll need to create a JSON file with the user data you want to import into Auth0. How you export user data to JSON will vary depending on your existing user database. To see the JSON file schema and examples, visit [Bulk Import Database Schema and Examples](##).

The file size limit for a bulk import is 500KB. Start multiple imports if your data exceeds this size.

## Request bulk user import

To start a bulk user import job, make a `POST` request encoded as type `multipart/form-data` to the [create import users job](##) endpoint. Be sure to replace the `MGMT_API_ACCESS_TOKEN`, `USERS_IMPORT_FILE.json`, `CONNECTION_ID`, and `EXTERNAL_ID` placeholder values with your Management API Access Token, users JSON file, database connection ID, and external ID, respectively.

```http
curl --request POST \
  --url 'https://your_domain.com/api/v2/jobs/users-imports' \
  --header 'authorization: Bearer MGMT_API_ACCESS_TOKEN' \
  --form users=@USERS_IMPORT_FILE.json \
  --form connection_id=CONNECTION_ID \
  --form external_id=EXTERNAL_ID
```

| Parameter | Description |
|-----------|-------------|
| `users` | [File in JSON format](##) that contains the users to import. |
| `connection_id` | ID of the connection to import the users into. You can retrieve the ID using the [GET /api/v2/connections](##) endpoint. |
| `upsert` | Boolean value; `false` by default. When set to `false`, pre-existing users that match on email address, user ID, or username will fail. When set to `true`, pre-existing users that match on any of these fields will be updated, but only with [upsertable attributes](##). |
| `external_id` | Optional user-defined string that can be used to correlate multiple jobs. Returned as part of the job status response. |
| `send_completion_email` | Boolean value; `true` by default. When set to `true`, sends a completion email to all tenant owners when the import job is finished. If you do *not* want emails sent, set this parameter to `false`. |

If the request is successful, you'll receive a response similar to the following:

```json
{
  "status": "pending",
  "type": "users_import",
  "created_at": "",
  "id": "job_abc123",
  "connection_id": "CONNECTION_ID",
  "upsert": false,
  "external_id": "EXTERNAL_ID",
  "send_completion_email": true
}
```

The returned entity represents the import job.

When the user import job finishes and if `send_completion_email` was set to `true`, the tenant administrators will get an email notifying them that the job either failed or succeeded. For example, an email for a failed job might say it failed to parse the users JSON file when importing users.

### Concurrent import jobs

The [create import users job](##) endpoint has a limit of two concurrent import jobs. Requesting additional jobs while there are two pending returns a `429 Too Many Requests` response:

```json
{
  "statusCode": 429,
  "error": "Too Many Requests",
  "message": "There are 2 active import users jobs, please wait until some of them are finished and try again"
}
```

## Check job status

You can check a job's status by making a `GET` request to the [get a job](##) endpoint. Be sure to replace the `MGMT_API_ACCESS_TOKEN` and `JOB_ID` placeholder values with your Management API Access Token and user import job ID.

```http
curl --request GET \
  --url 'https://your_domain.com/api/v2/jobs/JOB_ID' \
  --header 'authorization: Bearer MGMT_API_ACCESS_TOKEN' \
  --header 'content-type: application/json'
```

Depending on the status of the user import job, you'll receive a response similar to one of the following:

### Pending:

```json
{
  "status": "pending",
  "type": "users_import",
  "created_at": "",
  "id": "job_abc123",
  "connection_id": "CONNECTION_ID",
  "external_id": "EXTERNAL_ID"
}
```

### Processing:

```json
{
  "status": "processing",
  "type": "users_import",
  "created_at": "",
  "id": "job_abc123",
  "connection_id": "CONNECTION_ID",
  "external_id": "EXTERNAL_ID",
  "percentage_done": 0,
  "time_left_seconds": 0
}
```

### Completed

If a job is completed, the job status response includes totals of successful, failed, inserted, and updated records.

```json
{
  "status": "completed",
  "type": "users_import",
  "created_at": "",
  "id": "job_abc123",
  "connection_id": "CONNECTION_ID",
  "external_id": "EXTERNAL_ID",
  "location": "https://your_domain.com/EXAMPLE",
  "summary": {
    "failed": 0,
    "updated": 0,
    "inserted": 1,
    "total": 1
  }
}
```

### Failed

If there is an error in the job it will return as failed. Invalid user information, such as an invalid email, will not make the entire job fail.

```json
{
  "status": "failed",
  "type": "users_import",
  "created_at": "",
  "id": "job_abc123",
  "connection_id": "CONNECTION_ID",
  "external_id": "EXTERNAL_ID",
  "summary": {
    "failed": 1,
    "updated": 0,
    "inserted": 0,
    "total": 1
  }
}
```

To learn how to get details for failed entries, see [Retrieve failed entries](#retrieve-failed-entries). 

### Expired

Expired jobs are completed jobs that were created more than 2 hours ago.

```json
{
  "status": "expired",
  "type": "users_import",
  "created_at": "",
  "id": "job_abc123",
  "connection_id": "CONNECTION_ID",
  "external_id": "EXTERNAL_ID"
}
```

Job status is also added to [Tenant Logs](##), letting you setup a custom WebHook to be triggered using the [WebHook Logs Extension](##).

## Job timeouts

All user import jobs timeout after two (2) hours. If your job does not complete within this time frame, it is marked as failed.

All of your job-related data is automatically deleted after 24 hours and cannot be accessed afterward. As such, we strongly recommend storing job results using the storage mechanism of your choice.

## Retrieve failed entries

If there were errors in the user import job, you can get the error details by making a `GET` request to the [get job error details](##) endpoint. Be sure to replace the `MGMT_API_ACCESS_TOKEN` and `JOB_ID` placeholder values with your Management API Access Token and user import job ID.

```http
curl --request GET \
  --url 'https://your_domain.com/api/v2/jobs/JOB_ID/errors' \
  --header 'authorization: Bearer MGMT_API_ACCESS_TOKEN' \
  --header 'content-type: application/json'
```

If the request is successful, you'll receive a response similar to the following. Sensitive fields such as `hash.value` will be redacted in the response.

```json
[
    {
        "user": {
            "email": "test@test.io",
            "user_id": "7af4c65cb0ac6e162f081822422a9dde",
            "custom_password_hash": {
                "algorithm": "ldap",
                "hash": {
                    "value": "*****"
                }
            }
        },
        "errors": [
            {
                "code": "...",
                "message": "...",
                "path": "..."
            }
        ]
    }
]
```

Each error object will include an error code and a message explaining the error in more detail. The possible error codes are:

- `ANY_OF_MISSING`
- `ARRAY_LENGTH_LONG`
- `ARRAY_LENGTH_SHORT`
- `CONFLICT`
- `CONFLICT_EMAIL`
- `CONFLICT_USERNAME`
- `CONNECTION_NOT_FOUND`
- `DUPLICATED_USER`
- `ENUM_MISMATCH`
- `FORMAT`
- `INVALID_TYPE`
- `MAX_LENGTH`
- `MAXIMUM`
- `MFA_FACTORS_FAILED`
- `MIN_LENGTH`
- `MINIMUM`
- `NOT_PASSED`
- `OBJECT_REQUIRED`
- `PATTERN`
