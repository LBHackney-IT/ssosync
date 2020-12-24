# SSO Sync

## awslabs, modified by Hackney

This is forked from the AWS Labs version with a few changes:
-  `--ignore-groups` is now inverse, and exclusively includes groups listed
- Previously fatal errors when syncing groups are now logged, instead of crashing the script 
- The Google Credentials environment variable (`SSOSYNC_GOOGLE_CREDENTIALS`) now takes the JSON directly in the parameter, as opposed to a path to the file on disk

This project provides a CLI tool to pull users and groups from Google and push them into AWS SSO.
`ssosync` deals with removing users as well. The heavily commented code provides you with the detail of
what it is going to do.

## To deploy at Hackney

This is deployed within the HackIT account (where SSO is controlled) in eu-west-1

### Running Locally 

From the project root:
- Build the image: `docker build -t ssosync .`
- Launch the container: `docker run -it --rm --name ssosync-running ssosync`

Note: you must supply the appropriate environment variables to the container in order to run this locally. You can do this with the `--env-file .envfile` option in the `docker run` command. Use `.envfile.example` as a template. 

### Deploy to ECR
From the project root:
- Build the image: `docker build -t ssosync .`
`aws ecr get-login-password --profile hackney-hackit --region eu-west-1 | docker login --username AWS --password-stdin 338027813792.dkr.ecr.eu-west-1.amazonaws.com`

- Tag the image: `docker tag ssosync 338027813792.dkr.ecr.eu-west-1.amazonaws.com/hackney/ssosync`

- Push the image: `docker push 338027813792.dkr.ecr.eu-west-1.amazonaws.com/hackney/ssosync`

You then need to update the Task Definition to use the latest image.

Note: if you are deploying from scratch, you will need to pass the environment variables (as in `.envfile.example`) to the container from Parameter/Secret Store. 

### References

 * [SCIM Protocol RFC](https://tools.ietf.org/html/rfc7644)
 * [AWS SSO - Connect to Your External Identity Provider](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-idp.html)
 * [AWS SSO - Automatic Provisioning](https://docs.aws.amazon.com/singlesignon/latest/userguide/provision-automatically.html)

## Installation

You can `go get github.com/awslabs/ssosync` or grab a Release binary from the release page. The binary
can be used from your local computer, or you can deploy to AWS Lambda to run on a CloudWatch Event
for regular synchronization.

## Configuration

You need a few items of configuration. One side from AWS, and the other
from Google Cloud to allow for API access to each. You should have configured
Google as your Identity Provider for AWS SSO already.

You will need the files produced by these steps for AWS Lambda deployment as well
as locally running the ssosync tool.

### Google

First, you have to setup your API. In the project you want to use go to the [Console](https://console.developers.google.com/apis) and select *API & Services* > *Enable APIs and Services*. Search for *Admin SDK* and *Enable* the API.

You have to perform this [tutorial](https://developers.google.com/admin-sdk/directory/v1/guides/delegation) to create a service account that you use to sync your users. Save the JSON file you create during the process and rename it to `credentials.json`.

> you can also use the `--google-credentials` parameter to explicitly specify the file with the service credentials. Please, keep this file safe, or store it in the AWS Secrets Manager

In the domain-wide delegation for the Admin API, you have to specify the following scopes for the user.

`https://www.googleapis.com/auth/admin.directory.group.readonly,https://www.googleapis.com/auth/admin.directory.group.member.readonly,https://www.googleapis.com/auth/admin.directory.user.readonly`

Back in the Console go to the Dashboard for the API & Services and select "Enable API and Services".
In the Search box type `Admin` and select the `Admin SDK` option. Click the `Enable` button.

You will have to specify the email address of an admin via `--google-admin` to assume this users role in the Directory.

### AWS

Go to the AWS Single Sign-On console in the region you have set up AWS SSO and select
Settings. Click `Enable automatic provisioning`.

A pop up will appear with URL and the Access Token. The Access Token will only appear
at this stage. You want to copy both of these as a parameter to the `ssosync` command.

Or you specific these as environment variables.

```
SSOSYNC_SCIM_ACCESS_TOKEN=<YOUR_TOKEN>
SSOSYNC_SCIM_ENDPOINT=<YOUR_ENDPOINT>
```

## Local Usage

Usage:

The default for ssosync is to run through the sync.

```text
A command line tool to enable you to synchronise your Google
Apps (G-Suite) users to AWS Single Sign-on (AWS SSO)
Complete documentation is available at https://github.com/awslabs/ssosync

Usage:
  ssosync [flags]

Flags:
  -t, --access-token string         SCIM Access Token
  -d, --debug                       Enable verbose / debug logging
  -e, --endpoint string             SCIM Endpoint
  -u, --google-admin string         Google Admin Email
  -c, --google-credentials string   Provide the Google credentials JSON 
  -h, --help                        help for ssosync
      --ignore-groups strings       ignores these groups
      --ignore-users strings        ignores these users
      --log-format string           log format (default "text")
      --log-level string            log level (default "warn")
  -v, --version                     version for ssosync
```

The output of the command when run without 'debug' turned on looks like this:

```
2020-05-26T12:08:14.083+0100	INFO	cmd/root.go:43	Creating the Google and AWS Clients needed
2020-05-26T12:08:14.084+0100	INFO	internal/sync.go:38	Start user sync
2020-05-26T12:08:14.979+0100	INFO	internal/sync.go:73	Clean up AWS Users
2020-05-26T12:08:14.979+0100	INFO	internal/sync.go:89	Start group sync
2020-05-26T12:08:15.578+0100	INFO	internal/sync.go:135	Start group user sync	{"group": "AWS Administrators"}
2020-05-26T12:08:15.703+0100	INFO	internal/sync.go:172	Clean up AWS groups
2020-05-26T12:08:15.703+0100	INFO	internal/sync.go:183	Done sync groups
```

You can ignore users to be synced by setting `--ignore-users user1@example.com,user2@example.com` or `SSOSYNC_IGNORE_USERS=user1@example.com,user2@example.com`. Groups are **included** by setting `--ignore-groups group1@example.com,group1@example.com` or `SSOSYNC_IGNORE_GROUPS=group1@example.com,group1@example.com`.

## License

[Apache-2.0](/LICENSE)
