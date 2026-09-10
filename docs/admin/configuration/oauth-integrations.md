---
id: oauth-integrations
title: Per-User OAuth Integration Setup
sidebar_label: OAuth Integrations
sidebar_position: 10
pagination_prev: admin/configuration/index
pagination_next: null
---

# Per-User OAuth Integration Setup

AI/Run CodeMie supports OAuth 2.0 authorization flows for GitLab, Jira, and Confluence
integrations. This guide explains how administrators configure the OAuth application
credentials and enable each provider's OAuth flow.

## Overview

With per-user OAuth integrations:

- An administrator registers a single OAuth application on GitLab or Atlassian and
  configures the shared `client_id` and `client_secret` in CodeMie once.
- Each platform member then authorizes under their own account. Tokens are stored
  per-user — no member's token is shared with or visible to other members.
- Calls to GitLab, Jira, or Confluence run as the individual member who connected.

This model contrasts with Personal Access Token (PAT) integrations, where a single
token shared at the project level is used for all members.

## Prerequisites

- Administrative access to the CodeMie platform.
- An existing OAuth application registered on the provider (GitLab instance,
  or Atlassian Developer Console for Jira and Confluence).
- Ability to set environment variables for the CodeMie deployment.

## Provider Enable Flags

Each provider's OAuth option is controlled by a dedicated platform flag. By default all
flags are **off** — the "Use OAuth 2.0 sign-in" toggle is hidden in the integration form
for that provider until the flag is enabled.

| Provider   | Environment Variable       | Effect when ON                                           |
| ---------- | -------------------------- | -------------------------------------------------------- |
| GitLab     | `GITLAB_OAUTH_ENABLED`     | Shows the OAuth toggle in Git integrations (GitLab only) |
| Jira       | `JIRA_OAUTH_ENABLED`       | Shows the OAuth toggle in Jira integrations              |
| Confluence | `CONFLUENCE_OAUTH_ENABLED` | Shows the OAuth toggle in Confluence integrations        |

The flags are delivered to the UI via `GET /v1/config` as runtime feature flags
(`features.gitlabOauth`, `features.jiraOauth`, `features.confluenceOauth`). Presence in
the response means enabled; absence means disabled.

:::warning Default state
All three flags default to `false`. Members see only the Personal Access Token option
until the corresponding flag is explicitly set to `true`.
:::

Set the flags in the deployment's environment configuration (Helm values or equivalent):

```yaml
env:
  GITLAB_OAUTH_ENABLED: 'true'
  JIRA_OAUTH_ENABLED: 'true'
  CONFLUENCE_OAUTH_ENABLED: 'true'
```

Each flag is independent — enabling Jira OAuth does not enable Confluence OAuth, and
vice versa.

## GitLab OAuth Application

GitLab OAuth is registered per-instance. The OAuth application must be registered on the
specific GitLab instance that members will authorize against.

### 1. Register the OAuth Application on GitLab

1. Log in to the GitLab instance as an admin.
2. Navigate to **Admin Area → Applications** (or use **User Settings → Applications** for a
   user-owned application).
3. Click **New application** and fill in:
   - **Name**: A display name (e.g., `CodeMie OAuth`).
   - **Redirect URI**: The CodeMie callback URL for GitLab:
     `https://<codemie-base-url>/v1/gitlab-oauth/callback`
   - **Scopes**: Select `api`, `read_user`.
4. Click **Save application**. Note the **Application ID** and **Secret**.

### 2. Configure CodeMie with the GitLab OAuth Credentials

Set the following environment variables on the CodeMie deployment:

```yaml
env:
  GITLAB_OAUTH_ENABLED: 'true'
  GITLAB_OAUTH_CLIENT_ID: '<Application ID from GitLab>'
  GITLAB_OAUTH_CLIENT_SECRET: '<Secret from GitLab>'
  GITLAB_OAUTH_CALLBACK_BASE_URL: 'https://<codemie-base-url>'
```

The callback URL registered on GitLab must match `<GITLAB_OAUTH_CALLBACK_BASE_URL>/v1/gitlab-oauth/callback`.

### 3. Allowed GitLab Instances

The platform maintains an allowlist of GitLab instances that members may authorize
against. Only the configured instance URL is accepted. Members who attempt to connect to
an unlisted instance receive an authorization error.

:::info
The allowed instance is determined by the `GITLAB_OAUTH_CALLBACK_BASE_URL` and the
`instance_url` set in the integration form. Ensure the instance URL in the integration
matches the GitLab instance where the application was registered.
:::

## Atlassian OAuth Application (Jira and Confluence)

Jira and Confluence share a **single** Atlassian OAuth application and a single callback
URL. One Atlassian app registration covers both providers — the platform routes Jira and
Confluence flows through a shared Atlassian callback endpoint
(`/v1/atlassian-oauth/callback`). The `cloud_id` for the member's Atlassian site is
resolved automatically after authorization.

### 1. Register the OAuth Application on Atlassian

1. Go to the [Atlassian Developer Console](https://developer.atlassian.com/console/myapps/).
2. Click **Create** and select **OAuth 2.0 integration**.
3. Give the app a name (e.g., `CodeMie OAuth`).
4. Under **Permissions**, add the required API scopes:
   - **Jira**: `read:jira-work`, `write:jira-work`, `read:jira-user`
   - **Confluence**: `read:confluence-content.all`, `write:confluence-content`,
     `read:confluence-space.summary`
5. Under **Authorization**, add the callback URL:
   `https://<codemie-base-url>/v1/atlassian-oauth/callback`
6. Click **Save** and note the **Client ID** and **Secret**.

### 2. Configure CodeMie with the Atlassian OAuth Credentials

Set the following environment variables on the CodeMie deployment:

```yaml
env:
  JIRA_OAUTH_ENABLED: 'true'
  CONFLUENCE_OAUTH_ENABLED: 'true'
  JIRA_OAUTH_CLIENT_ID: '<Client ID from Atlassian>'
  JIRA_OAUTH_CLIENT_SECRET: '<Secret from Atlassian>'
  JIRA_OAUTH_CALLBACK_BASE_URL: 'https://<codemie-base-url>'
  CONFLUENCE_OAUTH_CLIENT_ID: '<Client ID from Atlassian>'
  CONFLUENCE_OAUTH_CLIENT_SECRET: '<Secret from Atlassian>'
  CONFLUENCE_OAUTH_CALLBACK_BASE_URL: 'https://<codemie-base-url>'
```

:::note
The Jira and Confluence `CLIENT_ID` and `CLIENT_SECRET` values are the same (same
Atlassian app), but they are configured as separate environment variables to allow
independent rotation in future. The `CALLBACK_BASE_URL` must resolve to the Atlassian
callback: `<CALLBACK_BASE_URL>/v1/atlassian-oauth/callback`.
:::

## Shared vs. Personal OAuth Integrations

The OAuth credentials (app `client_id` / `client_secret`) configured by an administrator
are placed in the integration form when creating a **Project integration**. For **User
integrations**, a member may also create a personal OAuth integration by entering their
own OAuth application credentials.

In both cases the per-user token model applies: whichever `client_id` / `client_secret`
is used for the application, the resulting access token belongs to the individual member
who authorized.

## Verification

After configuring the flags and credentials:

1. Restart the CodeMie deployment to apply the environment changes.
2. Log in as a regular member (not an admin).
3. Navigate to **Integrations → User → + Create**.
4. Select **Credential Type: Jira** (or Confluence or Git).
5. Confirm the **Use OAuth 2.0 sign-in** toggle is visible.
6. Complete the sign-in flow and verify the integration saves successfully.

If the toggle is not visible, confirm the flag is set to `true` and the deployment was
restarted. Check the `GET /v1/config` response for the corresponding feature flag key.

## Troubleshooting

| Symptom                                             | Likely cause                                                                |
| --------------------------------------------------- | --------------------------------------------------------------------------- |
| OAuth toggle not visible                            | Provider flag (`*_OAUTH_ENABLED`) is `false` or not set                     |
| Sign-in popup fails or returns an error             | Callback URL mismatch between the OAuth app registration and CodeMie config |
| "Instance not allowed" error on GitLab OAuth        | The GitLab instance URL is not on the platform allowlist                    |
| "cloud_id not found" error on Atlassian OAuth       | The Atlassian account has no accessible sites; check site permissions       |
| Token encrypted but inaccessible after key rotation | Encryption key changed; members need to reconnect                           |
