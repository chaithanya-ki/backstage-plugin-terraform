# [Backstage](https://backstage.io)

This is an example Backstage app that includes two Terraform plugins:

1. Scaffolder Action - creates Terraform Cloud/Enterprise resources using scaffolder
1. Terraform frontend plugin - retrieves information for an entity based on an organization and workspace

## Prerequisitess

Install [Backstage prerequisites](https://backstage.io/docs/getting-started/#prerequisites).

## Install

In your terminal, set the Terraform Cloud token and GitHub token

```sh
export GITHUB_TOKEN=""
export TF_TOKEN=""
```

To start the app, run:

```sh
yarn install

yarn dev
```

## Using the Scaffolder Action

This repository includes an example template for the Scaffolder to use
under `examples/terraform`.

Review `template.yaml` for the series of custom actions specific
to Terraform. You can...

- Create projects
- Create workspaces
- Create runs

However, you will encounter a few caveats:

- Scaffolder is *not* intended to be idempotent. If you have an
  existing project, you must remove the "Create Project" step
  from the template.

- Variables must be passed through scaffolder. Secrets should
  not be passed directly through scaffolder, consider setting them
  separately using variable sets or using dynamic credentials
  from Vault.

- Workspaces use VCS connections. This ensures that you can
  manage your infrastructure on Day 2.
  - If you do not specific a `vcsAuthUser`, the VCS connection will
    default to the first OAuth client returned by the Terraform API.
  - If you specify a `vcsAuthUser`, the action will return
    the first VCS OAuth token associated with that user. **Note that
    `vcsAuthUser` must have sufficient permissions to access
    the `vcsRepo` you are connecting.**


## Using Scaffolder with HashiCorp Vault & GitHub

Ideally, you'll want to scope your Terraform token to
the workspace and projects specific to a group. One approach
is to use HashiCorp Vault to generate the Terraform tokens.

In order to allow Backstage to access Vault, you need to configure
an authentication provider for Backstage using an SCM tool (GitHub, GitLab, etc.).

This is because Scaffolder has a
[built-in action](https://backstage.io/docs/features/software-templates/writing-templates/#using-the-users-oauth-token)
that allows you to retrieve a user OAuth token from the SCM tool for use in subsequent actions.

```text
             ┌─────────────► SCM Provider ◄─────────────────────┐
             │                 (GitHub)                         │
             │                                                  │
             │                                                  │
             │                                             ┌────┴────┬──────────────────┐
             │                                             │         │ Vault            │
┌────────────┴──────────────┐Auth with OAuth user token    │         │                  │
│                           ├──────────────────────────────►  GitHub │                  │
│        Backstage          │  Return Vault token          │   Auth  │ Terraform Cloud  │
│   (GitHub Auth Provider)  ◄──────────────────────────────┤  Method │  Secrets Engine  │
│                           │                              │         │                  │
└────────────▲───┬──────────┘                              └─────────┴────▲──┬──────────┘
             │   │             Use Vault token to get TFC token           │  │
             │   └────────────────────────────────────────────────────────┘  │
             │                    Return TFC Token                           │
             └───────────────────────────────────────────────────────────────┘
```

### Set up GitHub

1. Create a [personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
   with read-only access to your organization.

1. Set the personal access token as an environment variable for Backstage.
   This allows Backstage to read repositories and register entities into catalog.
   ```shell
   export GITHUB_TOKEN=""
   ```

1. Create an OAuth App for Backstage in GitHub under **your organization**.

1. Set the client ID and secret as environment variables for Backstage.
   ```shell
   export AUTH_GITHUB_CLIENT_ID=""
   export AUTH_GITHUB_CLIENT_SECRET=""
   ```

1. Sign into Backstage using your GitHub user and make sure
   you grant a user access to the organization.

### Set up Terraform Cloud

1. Set a read-only Terraform Cloud token that allows
   Backstage frontend components to retrieve information
   about workspaces, runs, and outputs.
   ```shell
   export TF_TOKEN=""
   ```

1. In your Terraform Cloud organization, add a VCS provider.

1. Create an OAuth App for Terraform Cloud in GitHub under **your organization**.

### Set up Vault

1. Using Docker, create a development Vault server.
   ```shell
   cd vault && docker-compose up -d && cd ..
   ```

1. Set environment variables to configure Vault Github auth method,
   using organization, organization ID, and a sample user.
   ```shell
   export VAULT_GITHUB_ORG=""
   export VAULT_GITHUB_ORG_ID=""
   export VAULT_GITHUB_USER=""
   ```

1. Set environment variables for Terraform Cloud secrets engine,
   including organization token and team ID specific to backstage.
   ```shell
   export TERRAFORM_CLOUD_ORGANIZATION_TOKEN=""
   export TERRAFORM_CLOUD_TEAM_ID=""
   ```

1. Run `bash vault/setup.sh`. This sets up the auth method, policies,
   and secrets engines for Backstage to authenticate and retrieve secrets
   from Vault.

### Run the Scaffolder Template

Start Backstage. Choose the Terraform template and enter the form values.
The Vault defaults are already set. When you create the repository,
Backstage authenticates to Vault with its GitHub OAuth user access token,
gets a Vault token, and uses that to retrieve the Terraform Cloud token.

The Terraform Cloud token in this example is a team token, which has sufficient
permission to create workspaces and projects in Terraform Cloud.
