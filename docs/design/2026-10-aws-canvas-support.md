# AWS and EKS support in the Radius canvas

- **Author**: Karishma Chawla (@kachawla)
- **Date**: 2026-10
- **Status**: Draft

## Current state

The Radius canvas deploys only to Azure. A developer whose team runs on AWS and Amazon EKS can't connect an account, create an environment, or deploy from the canvas. Some AWS code exists, but users can't reach it. The provider list shows "AWS (coming soon)", an AWS credential profile asks for an IAM role the developer must create by hand, and the deploy workflow runs MySQL on Amazon RDS and everything else on the cluster.

The functional spec in `docs/design/2026-09-aws-support-canvas.md` defines the AWS journey with the same screens as Azure. A prototype built that journey and deployed a real app to EKS with MySQL on Amazon RDS. It also showed where the work is hard: sharing one IAM role across environments, IAM size limits, short-lived tokens during long deploys, and apps that can't get credentials for Amazon S3 or Amazon Bedrock. This doc settles those so the prototype can land as reviewable changes.

## Proposal

The canvas gets AWS as a second provider, with the same journey as Azure. Setup creates one deploy identity per repository per AWS account, gives each environment its own region-limited permissions on it, and grants it access to the EKS cluster. Deploys use an AWS recipe pack in Bicep, so backing services run as AWS managed services, and AWS stays hidden from users until all of it ships.

## Objectives

### Goals

1. A developer can connect an AWS account and region, checked against their AWS CLI session.
2. A developer can create an AWS environment on an EKS cluster without opening the AWS console.
3. A developer can use an IAM role their platform team owns, and Radius never deletes it.
4. A second environment in the same account reuses the repository's deploy identity and leaves the first one working.
5. A deploy runs the app's backing services as AWS managed services.
6. A deploy that runs longer than the GitHub and EKS token lifetimes still succeeds.
7. The deployed graph shows each AWS resource's status and links it to the AWS console.
8. Deleting an environment removes only what that environment added in AWS.
9. Setup that stops part way can be rolled back or retried.
10. Users don't see AWS until all of the above ships.

### Non-goals

- App credentials for S3 and Bedrock, and the recipes for those two types: this doc records the choice of EKS Pod Identity, and the work is designed and built separately.
- Kafka on AWS in the first release: Radius can't yet read Amazon MSK's broker addresses.
- AWS resource coverage in Radius Bicep: covered by the full AWS coverage design.
- Creating or changing EKS clusters, VPCs, or subnets: the canvas only discovers them, the same as for AKS.
- Compute outside EKS, such as Amazon ECS or AWS Fargate: Radius runs containers on Kubernetes.
- Testing EKS Auto Mode: these clusters can be selected, but they're untested.
- Choosing an AWS CLI profile in the canvas: the canvas uses the CLI's default credentials and checks the account matches.
- Terraform recipes: Terraform cannot be used in Radius.
- Deleting the deploy identity: Azure never deletes the app registration, because other environments or services may use it.

## Design

### Terms

- **Deploy identity**: the IAM role that GitHub Actions assumes to deploy. AWS's equivalent of the Entra app registration on Azure.
- **Subject**: the text GitHub puts in its OIDC token to say which repository and environment the token is for, such as `repo:contoso/storefront:environment:dev`. The deploy identity's trust policy lists the subjects it accepts.
- **GitHub identity provider**: the account-level IAM object that lets AWS accept GitHub's OIDC tokens. One serves every repository in the account.
- **Access entry**: the EKS object that gives an IAM role access to a cluster's Kubernetes API.
- **Recipe pack**: the list that maps each Radius resource type to the recipe that creates it.

### Connect an AWS account

A credential profile holds an AWS account ID and a region, and no role. **Verify Credentials** runs `aws sts get-caller-identity` on the developer's machine. If the CLI session is signed in to a different account, verification fails and Save stays blocked, naming both accounts. With no session, the canvas offers **Sign in to AWS CLI**, which runs `aws sso login`.

The profile stores no secrets. Deleting a profile never changes anything in AWS, and environments keep working, because each environment keeps its own copy of the values.

### Deploy identity

**One deploy identity per repository per AWS account.** A second environment in the same account adds itself to the existing deploy identity instead of creating another. An IAM role can't be used from another account, so an account per environment means a deploy identity per environment.

**Name.** The default name is `radius-deploy-<owner>-<repo>`. IAM role names allow at most 64 characters. A longer name keeps its first 57 characters, then adds `-` and the first 6 characters of a SHA-256 hash of `owner/repo`. The hash keeps two long repository names from being shortened to the same role name. The developer can type another name.

**How Radius knows a role is its own.** Radius tags a role it creates with `radius-managed=true` and `radius-repository=<owner>/<repo>`. If a role with the proposed name exists without those tags, setup stops and changes nothing, because that role belongs to someone else. The developer can pick it on purpose instead.

**Picking an existing role.** "use an existing role…" opens a picker, like Azure's application picker. It lists the IAM roles the signed-in identity can read, except AWS service-linked roles, which GitHub can't assume. Each row shows the role name and the repositories its trust policy already accepts. Roles that already trust GitHub are listed first. Radius never tags or deletes a picked role, and only adds and removes what each environment needs.

**Trust.** The trust policy has one statement for the account's GitHub identity provider, listing every accepted subject. Setup adds the environment's subject to that list, and leaves any statement that doesn't name the GitHub identity provider unchanged. Setup builds the subject with the same code Azure uses, so it matches what GitHub sends.

**Permissions.** Each environment adds an inline policy named `radius-deploy-<environment>` to the deploy identity. The policy has the statements of AWS's `PowerUserAccess`, with every action limited to the environment's region by the `aws:RequestedRegion` condition. A few global read actions, such as listing regions, stay unlimited. The deploy identity can therefore act in exactly the regions its environments use, and deleting an environment removes that region again.

Two kinds of actions are kept from the deploy identity:

| Actions                                                                                                                                                           | How                                                                                                                                                                                                  | Why                                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| `iam:*`, `organizations:*`, `account:*`                                                                                                                           | Not granted. The only exceptions are the few global actions `PowerUserAccess` grants, such as `iam:CreateServiceLinkedRole`, which RDS and OpenSearch need the first time they're used in an account | The deploy identity can't give itself or anyone else more access                                                     |
| EKS access entry actions (`eks:CreateAccessEntry`, `eks:UpdateAccessEntry`, `eks:DeleteAccessEntry`, `eks:AssociateAccessPolicy`, `eks:DisassociateAccessPolicy`) | Denied                                                                                                                                                                                               | Without this, `PowerUserAccess` would let the deploy identity make itself cluster-admin on any cluster in the region |

Setup grants cluster access while running as the developer, so the deploy workflow never needs those EKS actions. The workflow no longer creates the access entry on every run.

**Limits.** The size limits match Azure's limit of 20 federated credentials per app registration.

| Limit                                                                                        | What it allows                            | What the user sees when it's reached                     |
|----------------------------------------------------------------------------------------------|-------------------------------------------|----------------------------------------------------------|
| A role's inline policies share 10,240 characters, and each environment's policy is about 500 | About 20 environments per deploy identity | Setup stops at the permissions step, quoting AWS's error |
| The trust policy allows 2,048 characters by default (8,192 with a quota increase)            | About 30 subjects                         | Setup stops at the trust step, quoting AWS's error       |

### Cluster access

Setup adds an access entry for the deploy identity on the selected cluster, with AWS's `AmazonEKSClusterAdminPolicy`. That matches Azure, which grants `Azure Kubernetes Service RBAC Cluster Admin`. Access entries need AWS CLI 2.15.3 or later, and a cluster whose authentication mode is `API` or `API_AND_CONFIG_MAP`.

### Infrastructure discovery

Discovery runs the developer's `aws` CLI with `--region` set to the profile's region.

| Field       | Where the choices come from                                           | Default and rules                                                                                                                                                                              |
|-------------|-----------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| EKS cluster | `eks list-clusters`, then `describe-cluster` for each                 | Clusters in `CONFIG_MAP` mode are listed but can't be selected, and show the command their owner runs to switch modes                                                                          |
| Namespace   | The cluster's namespaces, or a typed name                             | A typed name is recorded and created at deploy time, the same as Azure                                                                                                                         |
| VPC         | `ec2 describe-vpcs`                                                   | The cluster's VPC. Choosing another VPC shows a warning that reaching it needs VPC peering or a transit gateway, which Radius doesn't create                                                   |
| Subnets     | `ec2 describe-subnets` and `describe-route-tables` for the chosen VPC | Each subnet shows its availability zone, and is labeled public if its route table reaches an internet gateway. Private subnets are checked by default. Picking a public subnet shows a warning |

A selection must span at least two availability zones, because RDS and other VPC-based services require it. A single-zone selection is an error on the Subnets field, and keeps **Create Environment** disabled. If the cluster's private subnets don't span two zones, the form says so and leaves Subnets empty.

### Create Environment

Setup runs in the same three stages as Azure. Every check that can't change anything runs before the first write to AWS.

| Stage                     | Step                     | What happens                                                                                                                                                                   |
|---------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Authorize deploy identity | Check                    | AWS CLI version, signed-in account, cluster authentication mode, and that the GitHub environment isn't already set up for Azure                                                |
|                           | GitHub identity provider | Created if the account has none. Never deleted                                                                                                                                 |
|                           | Deploy identity          | Created with tags, reused if tagged for this repository, or the picked role is used                                                                                            |
|                           | Trust                    | The environment's subject is added                                                                                                                                             |
|                           | Permissions              | The environment's inline policy is written                                                                                                                                     |
|                           | Cluster access           | Access entry created, or reused if another environment on the same cluster already made it. A new one is retried for up to a minute while a new role spreads through IAM       |
| Configure environment     | GitHub                   | Writes `AWS_ROLE_ARN`, `AWS_REGION`, `AWS_ACCOUNT_ID`, `AWS_EKS_CLUSTER_NAME`, `RADIUS_VPC_ID`, `RADIUS_SUBNET_IDS`, and `KUBERNETES_NAMESPACE`, and commits the AWS workflows |
| Verify credentials        | Workflow                 | Runs `verify-aws.yml`, which assumes the deploy identity from GitHub and reaches the cluster                                                                                   |

The AWS workflows are committed only when the repository creates its first AWS environment. Until then, the deploy dispatcher keeps its AWS job removed, so a repository that only uses Azure never sees AWS files.

Setup records each AWS change in the existing setup operation record: role created, subject added, policy written, access entry created. Rollback uses that record. Setup moves out of `server.ts` into its own service behind the operation framework Azure uses.

### Deploy

`run-rad-commands.yml` sends an environment with `AWS_ROLE_ARN` set to `run-rad-commands-aws.yml`. That workflow:

1. Assumes the deploy identity over GitHub OIDC.
2. Starts the Radius control plane, gives it the GitHub token and a kubeconfig for the EKS cluster, and registers the AWS credential.
3. Creates the Radius environment, attaches the AWS recipe pack, and deploys.
4. Publishes the deployed graph and status, including after a failure.

**Tokens.** The GitHub token lasts 5 minutes, and an EKS token lasts 15. A deploy that waits 20 minutes for a database would fail at the next step after it. Every 2 minutes, a background script on the runner gets a new GitHub token and updates the copy the control plane reads. It then uses that token to get a new EKS token, and rewrites the control plane's kubeconfig. Azure already refreshes its GitHub token this way, on the same schedule. A single Kubernetes operation that runs longer than an EKS token's 15 minutes can still fail.

**Recipe pack.** The AWS recipe pack lives in `resource-types-contrib` next to the Azure and Kubernetes packs, and is versioned and published the same way. Its recipes are Bicep. Each recipe that creates a VPC-based service builds a subnet group from the environment's subnets, and adds the service to the cluster's security group, so pods can reach it without new rules. OpenSearch uses only the first subnet, because a single-node domain runs in one availability zone.

| Radius type                                  | AWS resource                     | Notes                                                                                                                                                                                                                                                                                                        |
|----------------------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| MySQL, PostgreSQL, SQL Server                | Amazon RDS DB instance           | SQL Server uses the Express edition                                                                                                                                                                                                                                                                          |
| Redis                                        | Amazon ElastiCache Serverless    | The only ElastiCache type Bicep can create                                                                                                                                                                                                                                                                   |
| MongoDB                                      | Amazon DocumentDB cluster        | Needs the native DocumentDB handler from the full AWS coverage design                                                                                                                                                                                                                                        |
| Object storage                               | Amazon S3 bucket                 | Added when pod identity ships, because apps need it to connect                                                                                                                                                                                                                                               |
| Models                                       | Amazon Bedrock                   | Creates no resource. Returns the regional endpoint and a Bedrock model ID, because the type's default model isn't on Bedrock. Added when pod identity ships, because apps need it to connect                                                                                                                 |
| Search                                       | Amazon OpenSearch Service domain |                                                                                                                                                                                                                                                                                                              |
| RabbitMQ                                     | Amazon MQ broker                 | Needs the full AWS coverage fix for resources whose name AWS generates. Until then, the Kubernetes recipe                                                                                                                                                                                                    |
| Kafka                                        | Amazon MSK, later                | Not in the first release. MSK gives its broker addresses only through a separate API call, so the recipe can't return them to the app. A native handler in the full AWS coverage work will read them. Running Kafka on the cluster isn't a shortcut, because no Kubernetes Kafka recipe exists for any cloud |
| Containers, images, volumes, routes, secrets | Kubernetes recipes               | Same as Azure                                                                                                                                                                                                                                                                                                |

Before a deploy starts, the canvas checks the app model against the AWS recipe pack. If the app uses a type the pack doesn't have yet, such as object storage, models, or Kafka, **Deploy** is blocked and names the type, so nothing is created. The planned graph shows that node as unresolved.

Deleting a deployment deletes these resources through the AWS Cloud Control API, which removes an RDS database without a final snapshot. The delete confirmation already names each resource it deletes.

### App credentials

On Azure, a recipe returns a key, such as a storage account key, and the app uses it. Amazon S3 and Amazon Bedrock accept only IAM credentials, which a recipe can't hand back as a key today. We'll use **EKS Pod Identity**.

| Option                                | What it is                                                                     | Result                                                                                                                                                                                                                                   |
|---------------------------------------|--------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Long-lived access keys                | The recipe creates an IAM user and returns its key                             | Rejected. AWS advises against long-lived keys. The key never expires, and copies end up in recipe state, a Kubernetes secret, and the app. The deploy identity would also need rights to create IAM users, which lets it create an admin |
| Generic credential fields             | Add AWS fields such as `accessKeyId` to the Radius types                       | Rejected. It still needs long-lived keys, and changing the types affects Azure apps                                                                                                                                                      |
| Node role                             | Grant access to the IAM role of the cluster's nodes                            | Rejected. Every pod on those nodes gets the same access, and it changes the user's cluster                                                                                                                                               |
| Separate types per cloud              | AWS-only Radius types with AWS-shaped outputs                                  | Rejected. The same app definition could no longer deploy to both clouds                                                                                                                                                                  |
| IAM roles for service accounts (IRSA) | A role per app, trusted through the cluster's own OIDC provider                | Works, but needs an OIDC provider per cluster, and each role's trust policy grows with every cluster                                                                                                                                     |
| **EKS Pod Identity**                  | A role per app, with access to only its own resources, linked to the app's pod | **Chosen**                                                                                                                                                                                                                               |

Pod Identity stores no secret. The app's AWS SDK gets short-lived credentials automatically, so most apps need no code change. It's AWS's recommended way for pods to reach AWS services, and it matches the passwordless model the canvas already uses for deploys. Azure workload identity works the same way, so both clouds can use it later.

It brings three new requirements:

- **A cluster prerequisite:** the EKS Pod Identity Agent add-on, installed once per cluster. Setup checks for it and tells the developer if it's missing.
- **A change to a shared type:** the container type and its recipe give each app its own Kubernetes service account.
- **More IAM rights for the deploy identity:** it must be able to create per-app roles, limited by a permissions boundary and an IAM path, so it can't create a role with more access than itself.

AWS ships before pod identity. Until pod identity ships, the AWS recipe pack leaves out object storage and models, so an app that uses them is blocked before deploy instead of deploying and then failing to connect.

### Application graph

The planned graph reads the AWS recipe pack, and shows each node with its AWS service name, such as "Amazon RDS for MySQL".

The deployed graph shows each resource's status from the deploy, the same as Azure, including the resource's own error when it fails. A deployed AWS resource links to its service's list in the AWS console, in the environment's region, for example the RDS databases list. Kubernetes resources, unknown types, and regions in China get no link, because a link to the wrong page is worse than none.

### Delete

Deleting a deployment runs `delete-aws.yml`, the same flow as Azure.

Deleting an environment runs `delete-environment-aws.yml`. It refuses while apps are still deployed, and then deletes the Radius environment. After that, the canvas removes this environment's subject from the trust policy, and deletes its inline policy. Trust statements Radius didn't add stay as they are. If removing the subject leaves the trust policy with no statements, the canvas writes a single deny statement, because IAM rejects an empty trust policy. The canvas keeps the deploy identity and the GitHub identity provider. Whether it keeps the access entry depends on who owns the deploy identity:

| Deploy identity         | Access entry on delete                                                                                                                                | Why                                                                                                                         |
|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Created by Radius       | Removed when no remaining environment in the repository uses that cluster. Setup tags the access entry it creates, and only a tagged entry is removed | Only this repository uses the deploy identity, and the repository's GitHub environments say which clusters are still in use |
| Picked by the developer | Always kept                                                                                                                                           | Another repository may use the same role on the same cluster, and Radius can't see that repository's environments           |

The completion message names what was removed and what was kept. Once the last environment is gone, the trust policy accepts no GitHub subject, so no GitHub workflow can assume the deploy identity or use a kept access entry.

### Hidden until it ships

The changes land in small pull requests, with AWS hidden from users until a final change turns it on.

- The provider list keeps "AWS (coming soon)", and **Create Env** stays disabled on AWS profiles.
- The server refuses to create or delete an AWS environment.
- The AWS workflows aren't committed to any repository.
- A developer-only switch, read when the extension starts, lifts all three, so the team can test against real AWS.
- Hidden changes carry no release notes. The final change announces AWS.

## Error handling

Each failure says what failed, what was changed, and what to do next. A read that fails never leads to a write.

| Situation                                                | What the user sees                                                                                                                                                                    |
|----------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| AWS CLI older than 2.15.3, or missing                    | Setup stops before any write, naming the installed and required versions and linking to the installer                                                                                 |
| No CLI session                                           | The **Sign in to AWS CLI** button, which runs `aws sso login`                                                                                                                         |
| CLI signed in to another account                         | Verification fails and Save stays blocked, naming both accounts                                                                                                                       |
| Discovery is denied, or finds nothing                    | Each list says whether the account has none or the signed-in identity can't read them. The namespace still accepts a typed name                                                       |
| Cluster in `CONFIG_MAP` mode                             | The cluster can't be selected. The list shows the `aws eks update-cluster-config` command its owner runs                                                                              |
| GitHub environment already set up for Azure              | Setup stops before any write, and asks for another environment name                                                                                                                   |
| A role with the proposed name isn't Radius's             | Setup stops and changes nothing. The developer can pick that role or choose another name                                                                                              |
| Picked role is in another account                        | Rejected in the picker. An IAM role can't be used from another account                                                                                                                |
| GitHub identity provider can't be created                | Setup stops, and shows the `aws iam create-open-id-connect-provider` command for an IAM administrator                                                                                 |
| Role creation denied                                     | Setup stops, and suggests picking an existing role or asking for `iam:CreateRole`                                                                                                     |
| Picked role can't be changed by the signed-in identity   | Setup stops, names the change that was denied, and leaves the role as it was                                                                                                          |
| Two environment names become the same policy name        | Setup stops and names both, so one environment never overwrites another's permissions. IAM policy names allow fewer characters than GitHub environment names                          |
| Trust policy can't be read                               | Setup stops without writing, so it never replaces a policy it couldn't see                                                                                                            |
| Another setup changed the trust policy at the same time  | After each write, setup reads the policy back. If its subject is missing, it merges again, up to 3 times, then stops naming the conflict                                              |
| Access entry fails while a new role spreads              | Retried for up to a minute, then setup stops quoting AWS's error                                                                                                                      |
| A size limit is reached                                  | Setup stops, naming the limit and quoting AWS's error                                                                                                                                 |
| A step fails after earlier steps wrote to AWS            | The **Setup didn't finish** card lists what was created. **Roll back** undoes those steps in reverse order. It never deletes the GitHub identity provider, or a role it didn't create |
| The app restarts during setup                            | The setup is marked interrupted. **Retry** runs every step again, and each step skips what already exists                                                                             |
| GitHub write fails after AWS steps succeeded             | Reported as a failure of the `Configure environment` stage, separate from AWS. Retry continues from there                                                                             |
| Verify credentials fails                                 | Says whether GitHub couldn't assume the deploy identity, couldn't reach the cluster, or couldn't push to GHCR, and links to the run                                                   |
| A token refresh fails during deploy                      | The refresher retries on its next cycle, 2 minutes later. If a token expires anyway, the failed resource shows the authorization error                                                |
| The app uses a type the AWS recipe pack doesn't have yet | **Deploy** is blocked before anything is created, naming the type                                                                                                                     |
| A recipe fails                                           | The deployed graph marks that resource failed and shows its error. The resources that succeeded stay marked as succeeded                                                              |
| Environment delete can't read the trust policy           | The canvas stops without writing, and tells the developer to remove the subject by hand                                                                                               |
| Removing the subject or the policy fails at delete       | The canvas says which one is left, and **Retry** removes what's still there                                                                                                           |
| The deploy identity is already gone at delete            | Reported as already gone, and the delete continues                                                                                                                                    |

## Test plan

- **Unit tests**:
  - Role names, including the hash for long names.
  - Adding and removing subjects, including the deny-only trust policy.
  - The permission policy's region condition and its denied actions.
  - Public and private subnet labels, and the two-zone rule.
  - Console links for each type, including GovCloud and China regions.
  - Parsing the CLI version.
- **Server tests**, with a fake `aws` runner:
  - Each row of the error table.
  - That no write happens before the checks pass.
  - That rollback undoes exactly what the setup record lists.
  - The read-back loop when another setup writes at the same time.
  - The access entry rules at environment delete, for a Radius-created and a picked deploy identity.
  - Blocking a deploy that uses a type the AWS recipe pack doesn't have.
- **Browser tests**: the deploy identity section, the role picker, subnet checks and warnings, and that AWS stays hidden with the switch off.
- **Workflow tests**: the token refresher in AWS mode, removing the dispatcher's AWS job, and the delete guard in `delete-environment-aws.yml`.
- **Cloud end-to-end tests**, in a dedicated AWS account:
  - Create two environments in two regions on one deploy identity.
  - Deploy an RDS database and a container in a deploy longer than 15 minutes.
  - Deploy a resource that fails, and check the graph shows it.
  - Delete both environments, and check that only the deploy identity and the GitHub identity provider are left.
  - A cleanup job removes anything a failed run leaves behind.
- **Azure regression**: the Azure end-to-end tests pass with the AWS switch both off and on.

## Alternatives considered

- **One managed policy per environment:** managed policy names are unique across the account, so two repositories with a `dev` environment would collide. A role also holds at most 20 managed policies, and `PowerUserAccess` would use one.
- **Attaching `PowerUserAccess` and denying other regions per environment:** deny statements from different environments add up, so a second region would block the first.
- **One deploy identity per environment:** every new environment would need rights to create IAM roles and a new name to type, and the spec chose one per repository to match Azure.
- **Deleting the deploy identity with its last environment:** Azure never deletes the app registration, and a role picked from a platform team must never be deleted.
- **Keeping the AWS recipes in this repository and publishing them to each user's registry at deploy:** this needs a registry secret copied into the control plane, and a hard-coded recipe list for the planned graph. One versioned pack in `resource-types-contrib` works like the Azure pack.
- **Shipping S3 and Bedrock recipes before pod identity:** apps would deploy and then fail to connect, with nothing explaining why.
- **Waiting for pod identity before shipping AWS:** it would hold back the other eight catalog types by the weeks pod identity takes.
- **Kafka on the cluster for the first release:** no Kubernetes Kafka recipe exists for any cloud, so it's new work, and MSK is the AWS answer once Radius can read its broker addresses.
- **Always keeping the access entry:** a deploy identity still used by an environment on another cluster would keep cluster-admin on a cluster it no longer needs.
- **A per-deploy Kubernetes service account token instead of refreshing EKS tokens:** it covers single long operations, but creates objects on the user's cluster. It's the fallback if long operations prove common.
