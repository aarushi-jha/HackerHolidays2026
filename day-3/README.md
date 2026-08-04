# Day 3
- **Challenge name:** [Complimentary](https://tryhackme.com/room/hh-complimentary-05e0b604)
- **Category:** Cloud Security / AWS Misconfiguration
- **Difficulty:** Easy/Medium
- **Date completed:** 29th July 2026

## Summary

This challenge centers on an insecurely configured AWS S3 wellness dashboard that distributes temporary IAM credentials to anonymous web visitors, allowing unauthenticated browsers to query a DynamoDB table directly. The core vulnerability exploits this broken trust model: by intercepting the credential-issuing mechanism, an attacker can reuse those permissions to bypass authorization boundaries, scrape records meant for other users, and extract unauthorized flags.

## Exploitation / Walkthrough

### Step 1 - Finding the client-side AWS logic
I opened the site and went straight into Developer Tools (F12) → Sources. Inside `app.js` was this main section:

```js
AWS.config.credentials.get(function (err) {
  ...
  const dynamodb = new AWS.DynamoDB({ region: AWS_REGION });
  dynamodb.getItem(
    { TableName: TABLE_NAME, Key: { guest_id: { S: guestId() } } },
    function (err, data) { renderDashboard(data.Item); }
  );
});
```

Inspecting the application flow revealed that the site leveraged AWS Cognito Identity Pools to hand out unauthenticated, temporary credentials straight to the browser. Under the hood, the frontend used those keys to execute client-side DynamoDB getItem queries, narrowly scoped to fetch only the active guest's personal record by their guest_id.

### Step 2 - Extracting the Identity Pool configuration
Next, I found the actual configuration values:

```js
const IDENTITY_POOL_ID = "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688";
const AWS_REGION = "us-east-1";
const TABLE_NAME = "complimentary-GuestWellnessProfiles";
AWS.config.credentials = new AWS.CognitoIdentityCredentials({
  IdentityPoolId: IDENTITY_POOL_ID,
});
```
The three main credentials were now found. 

### Step 3 - Watching Cognito issue credentials (Network tab)
Switching to the **Network** tab and filtering for `cognito`, I found and inspected the request to `cognito-identity.us-east-1.amazonaws.com`. Its `X-Amz-Target` header showed `AWSCognitoIdentityService.GetCredentialsForIdentity`. The **Response** body contained a full set of temporary AWS credentials:

```json
{
  "Credentials": {
    "AccessKeyId": "ASIA...",
    "SecretKey": "...",
    "SessionToken": "...",
    "Expiration": ...
  },
  "IdentityId": "us-east-1:..."
}
```

This confirmed the AWS mechanism: any anon visitor gets a temporarily validated IAM identity just by loading the page.

### Step 4 - Scanning instead of getting
The critical flaw lay in the authorization boundary: while the frontend application explicitly restricted itself to calling getItem for a single guest, the underlying IAM role lacked proper enforcement. The access limits existed purely in client-side JavaScript, not in the actual AWS permissions.

Because the AWS SDK was already initialized and AWS.config.credentials was populated in the session, bypassing the UI was trivial. I opened the browser console and executed:

```js
new AWS.DynamoDB({ region: 'us-east-1' }).scan(
  { TableName: 'complimentary-GuestWellnessProfiles' },
  (err, data) => console.log(err ? err.message : JSON.stringify(data, null, 2))
);
```

Swapping getItem for a DynamoDB scan operation dumps the entire table at once. Because the anonymous guest role was severely over-permissioned and allowed dynamodb:Scan, the cloud layer happily obliged, exposing every record in the database.
### Step 5 - Retrieving the flag
The `scan()` call returned every guest record in the table, including profiles that didn't belong to me. Reading through the JSON output in the console, one of the other guest records contained the flag.

Mission Acomplished!

## Flag
![redacted](https://img.shields.io/badge/-REDACTED-000000) <!-- TODO: paste the actual flag once ready to publish -->

## Lessons Learned
Key takeaway: Treating the browser as a trusted execution environment is a massive operational failure. Handing out temporary AWS credentials to anonymous visitors via Cognito Identity Pools—and relying entirely on frontend JavaScript logic rather than IAM policies to enforce access control—creates a wide-open door. Because the underlying role was over-permissioned to allow global actions like dynamodb:Scan, extracting the keys meant effortlessly dumping every single user's record. Cloud security requires strict least-privilege scoping at the API and database layer; assuming a user will only execute the frontend queries you designed for them is an absolute security illusion.
