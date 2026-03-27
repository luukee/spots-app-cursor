---
name: Bypass referenceAuth Cognito Migration
overview: Step-by-step migration plan to bypass Amplify referenceAuth and connect directly to a new Cognito User Pool with username or email sign-in. Uses Parameter Store for auth config and includes Identity Pool creation, user migration, API Gateway updates, and a local dev fetch script.
todos: []
isProject: false
---

# Bypass referenceAuth and Connect Directly to Cognito

## Context

The current setup uses Amplify Gen2 `referenceAuth`, which rejects Cognito pools with alias attributes. To support username + email sign-in, we must bypass referenceAuth and wire the app directly to a new pool. Auth config will live in AWS Systems Manager Parameter Store (per your spec); the Google OAuth client secret goes in Secrets Manager.

## Prerequisites

- AWS CLI configured with appropriate permissions
- `jq` installed (`brew install jq`)
- Access to Google Cloud Console (for OAuth)
- Old pool ID for export: `us-east-2_Xx7zXeTLI`

---

## Phase 1: Create Cognito User Pool and Identity Pool

### Step 1: Run the create-user-pool script

Run [create-new-pool.sh](spots-app/docs/cognito/create-user-pool/create-new-pool.sh):

```bash
cd spots-app/docs/cognito/create-user-pool
bash create-new-pool.sh
```

**Output:** `SPOTS-App-User-Pool-v4.json` contains the new pool ID. The script also creates the app client and domain (`spots-app-v4`).

**Capture:**

- `UserPool.Id` (e.g. `us-east-2_XXXXXXXXX`)
- App Client ID from AWS Console (Cognito → User Pools → your pool → App integration → App clients) or via:

```bash
  aws cognito-idp list-user-pool-clients --user-pool-id <POOL_ID> --region us-east-2
  

```

### Step 1.1: Create Cognito Identity Pool

The Identity Pool exchanges Cognito User Pool tokens for temporary AWS credentials. Create it and attach it to the new user pool.

**AWS Console:**

1. Cognito → Identity pools → Create identity pool
2. Identity pool name: `spots-auth-identity-pool-v4`
3. Under "Authentication providers" → Cognito: add your User pool ID and App client id
4. Create pool
5. Create roles for authenticated and unauthenticated (or use existing policy templates)
6. Copy: Identity pool ID, Authenticated role name, Unauthenticated role name

**Or via CLI** (after creating IAM roles):

```bash
aws cognito-identity create-identity-pool \
  --identity-pool-name "spots-auth-identity-pool-v4" \
  --allow-unauthenticated-identities \
  --cognito-identity-providers ProviderName="cognito-idp.us-east-2.amazonaws.com/us-east-2_8Bq85Vvio",ClientId="5jr1a7hhbfldassoosium225p4" \
  --region us-east-2
```

Then create and attach IAM roles for authenticated/unauthenticated users.

**Capture:** Identity pool ID, auth role name, unauth role name, AWS account ID.

---

## Phase 2: Parameter Store and Identity Providers

### Step 2: Create SSM Parameters

Store the eight auth parameters in Parameter Store (Standard tier; these are not secrets):

```bash
aws ssm put-parameter --name "/spots/auth/user-pool-id" --value "us-east-2_8Bq85Vvio" --type String --overwrite --region us-east-2
aws ssm put-parameter --name "/spots/auth/user-pool-client-id" --value "5jr1a7hhbfldassoosium225p4" --type String --overwrite --region us-east-2
aws ssm put-parameter --name "/spots/auth/identity-pool-id" --value "us-east-2:fbc1652d-7eb5-4edf-9278-bd7025e2792c" --type String --overwrite --region us-east-2
aws ssm put-parameter --name "/spots/auth/account-id" --value "834038620060" --type String --overwrite --region us-east-2
aws ssm put-parameter --name "/spots/auth/auth-role-name" --value "amplify-awsamplifygen2-ti-amplifyAuthauthenticatedU-aOSoYLcMvr20" --type String --overwrite --region us-east-2
aws ssm put-parameter --name "/spots/auth/unauth-role-name" --value "amplify-awsamplifygen2-ti-amplifyAuthunauthenticate-Iv1AZND3wBjp" --type String --overwrite --region us-east-2
aws ssm put-parameter --name "/spots/auth/aws-region" --value "us-east-2" --type String --overwrite --region us-east-2
aws ssm put-parameter --name "/spots/auth/oauth-domain" --value "spots-app-v4.auth.us-east-2.amazoncognito.com" --type String --overwrite --region us-east-2
```

### Step 2.1: Add Google as Identity Provider in Cognito

**Order matters:** Add Google to the pool before enabling it on the app client.

1. Cognito → User Pools → your pool (SPOTS App User Pool v4)
2. Sign-in experience → Federated identity provider sign-in → Add identity provider
3. Choose Google
4. Enter Google app Client ID and Client secret (from Google Cloud Console)
5. Map attributes: email → email, name → name, etc. (do not require phone_number)
6. Save

Then update the app client:

1. App integration → App clients → SPOTS Web Client v4 → Edit
2. Hosted UI → Identity providers: enable Google
3. OAuth 2.0 grant types: Authorization code
4. OpenID Connect scopes: openid, email, profile
5. Callback URLs and Sign-out URLs: already set by script
6. Save

---

## Phase 3: Secrets and User Migration

### Step 3: Add Google OAuth Client Secret to Secrets Manager

1. Secrets Manager → Store a new secret
2. Type: Other type of secret
3. Key/value: `GOOGLE_OAUTH_CLIENT_SECRET` = your Google OAuth client secret
4. Secret name: `spots/google-oauth-client-secret` (or similar)
5. Store

Cognito will use the secret you entered in Step 2.1. This step is for any backend or scripting that needs the secret outside Cognito.

### Step 4: Export and Import Users

**4a. Update export script**

In [export-users.py](spots-app/docs/cognito/export-import-users/export-users.py), `user_pool_id = 'us-east-2_Xx7zXeTLI'` is correct (source pool).

**4b. Run export**

```bash
cd spots-app/docs/cognito/export-import-users
python export-users.py
```

**4c. Update import script**

In [import-users.py](spots-app/docs/cognito/export-import-users/import-users.py):

- Set `new_user_pool_id` to your new pool ID from Step 1
- Remove or adjust `phone_number` if the new pool does not require it (v4 schema has `phone_number` optional)

**4d. Run import**

```bash
python import-users.py
```

Native (password) users will be imported. Google users are recreated on first sign-in.

**4e. Notify users:** Password users must reset their password on first login.

### Step 4.1: Google Cloud Console – Add Cognito Callback URL

1. Google Cloud Console → APIs & Services → Credentials
2. Edit your OAuth 2.0 Client ID
3. Authorized redirect URIs: add `https://spots-app-v4.auth.us-east-2.amazoncognito.com/oauth2/idpresponse`
4. Save

---

## Phase 4: API Gateway and Amplify Auth

### Step 5: API Gateway Authorizer and Method Auth

The API `vox46zfh9j` (spotsToRedCapAPI) must validate tokens from the new pool.

**AWS Console:**

1. API Gateway → spotsToRedCapAPI → Authorizers
2. Create or edit the Cognito authorizer:
  - Type: Cognito
  - Cognito User Pool: select the new pool (or enter its ID)
  - Token Source: `Authorization`
3. Resources: for each protected route, edit the method (GET, POST, PATCH):
  - Method Request → Authorization: select the Cognito authorizer
4. Deploy API to the `Dev` stage

Protected endpoints typically include: analytics, family, auth, user, symptoms, history, etc. OPTIONS methods usually stay `NONE`.

### Step 5.1: Modify Amplify Auth Resource (Bypass referenceAuth)

[amplify/auth/resource.ts](spots-app/amplify/auth/resource.ts) currently uses `referenceAuth`, which rejects alias pools and causes sandbox failures.

**Options:**

- **A) Remove referenceAuth and export minimal auth:** Replace `referenceAuth` with a custom export that returns auth config from env/Parameter Store without using the referenceAuth custom resource. The backend would need a different way to supply auth outputs for the frontend.
- **B) Conditional referenceAuth:** Add an env var (e.g. `USE_REFERENCE_AUTH=false`). When false, skip referenceAuth and export a stub or config built from env vars. This avoids the alias-pool check when the var is unset.

Implementation depends on how Amplify Gen2 expects the auth export to look. The goal is that sandbox synthesis does not invoke the referenceAuth custom resource when using the alias pool.

---

## Phase 5: Frontend Config and Local Dev

### Step 6: Fetch Script for Local Dev

Create a script that fetches the eight parameters from Parameter Store and writes `.env.local` (or generates a partial `amplify_outputs.json` auth section) so local dev and sandbox use the new pool.

**Script location:** `spots-app/scripts/fetch-auth-config.js` (or `.ts`)

**Logic:**

1. Use `@aws-sdk/client-ssm` to get parameters under `/spots/auth/`
2. Map them to `AMPLIFY_AUTH_`* env var names
3. Write `.env.local` (append or overwrite auth vars only)
4. Optionally log success

**npm script:** Add to [package.json](spots-app/package.json):

```json
"fetch-auth": "node scripts/fetch-auth-config.js",
"sandbox": "npm run fetch-auth && dotenvx run -f .env.local -- npx ampx sandbox"
```

**IAM:** The script runs locally; use AWS CLI credentials or a profile with `ssm:GetParameters` on `/spots/auth/`*.

---

## Phase 6: Update Frontend Auth Config

### Step 6.1: Amplify outputs and client-amplify

With referenceAuth bypassed, `amplify_outputs.json` may no longer be generated for auth. Two approaches:

1. **Generate auth section from Parameter Store:** The fetch script (or a build step) constructs the auth section and merges it with the Data section, writing `amplify_outputs.json`.
2. **Use env vars in client-amplify:** Modify [client-amplify.tsx](spots-app/app/_lib/auth/client-amplify.tsx) to read auth config from `process.env` when `amplify_outputs.json` auth is missing, and ensure the fetch script populates those env vars.

For alias pools (username + email), ensure `loginWith` supports both. Amplify v6 `signIn` accepts a username that can be email or preferred_username; `loginWith` may need to allow `username: true` for alias pools. Verify against [Amplify Auth documentation](https://docs.amplify.aws/react/build-a-backend/auth/).

---

## Summary Checklist


| Step | Action                                                                    |
| ---- | ------------------------------------------------------------------------- |
| 1    | Run create-new-pool.sh; capture pool ID and app client ID                 |
| 1.1  | Create Identity Pool; create/attach IAM roles; capture IDs and role names |
| 2    | Create 8 SSM parameters with `aws ssm put-parameter`                      |
| 2.1  | Add Google identity provider in Cognito; update app client for OAuth      |
| 3    | Store Google OAuth client secret in Secrets Manager                       |
| 4    | Export users from old pool; update import script; import into new pool    |
| 4.1  | Add Cognito callback URL in Google Cloud Console                          |
| 5    | Update API Gateway authorizer and method auth to new pool; deploy         |
| 5.1  | Modify auth resource to bypass referenceAuth                              |
| 6    | Create fetch script; wire into sandbox and dev workflow                   |
| 6.1  | Ensure frontend auth config uses new pool (env or generated outputs)      |


---

## Notes

- **Order dependency:** Add Google to the pool (2.1) before enabling Google on the app client.
- **Import script:** New pool has `preferred_username` optional; adjust import attributes if needed.
- **Production:** Amplify Console env vars should be populated from Parameter Store (manual or via a pre-build script). Amplify supports “Link to secret” for env vars; Parameter Store can be used similarly.
- **Risks:** Removing referenceAuth may affect how Amplify wires Data/AppSync to auth. Test Data operations after the switch.

