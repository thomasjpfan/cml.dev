# Self-hosted (On-premise or Cloud) Runners

GitHub Actions and GitLab CI workflows are run on GitHub- and GitLab- hosted
runners by default. However, there are many great reasons to use your own
runners: to take advantage of GPUs, orchestrate your team's shared computing
resources, or train in the cloud.

☝️ **Tip!** Check out the official documentation from
[GitHub](https://help.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners)
and [GitLab](https://docs.gitlab.com/runner) to get started setting up your
self-hosted runner.

## Allocating Cloud Compute Resources with CML

When a workflow requires computational resources (such as GPUs), CML can
automatically allocate cloud instances using `cml-runner`. You can spin up
instances on [AWS](#aws), [Azure](#azure), [GCP](#gcp), or
[Kubernetes](#kubernetes).

For example, the following workflow deploys a `p2.xlarge` instance on AWS EC2
and trains a model on the instance. After the job runs, the instance
automatically shuts down.

You might notice that this workflow is quite similar to the
[basic use case](/doc/usage). The only addition is `cml-runner` and a few
environment variables for passing your cloud compute credentials to the
workflow.

Note that `cml-runner` will also automatically restart your jobs (whether from a
[GitHub Actions 72-hour timeout](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners#usage-limits)
or a
[AWS EC2 spot instance interruption](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html)).

```yaml
name: Train-in-the-cloud
on: [push]
jobs:
  deploy-runner:
    runs-on: ubuntu-latest
    steps:
      - uses: iterative/setup-cml@v1
      - uses: actions/checkout@v2
      - name: Deploy runner on EC2
        env:
          REPO_TOKEN: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          cml-runner \
              --cloud=aws \
              --cloud-region=us-west \
              --cloud-type=p2.xlarge \
              --labels=cml-runner
  train-model:
    needs: deploy-runner
    runs-on: [self-hosted, cml-runner]
    timeout-minutes: 4320 # 72h
    container:
      image: docker://iterativeai/cml:0-dvc2-base1-gpu
      options: --gpus all
    steps:
      - uses: actions/checkout@v2
      - name: Train model
        env:
          REPO_TOKEN: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
        run: |
          pip install -r requirements.txt
          python train.py

          cat metrics.txt > report.md
          cml-send-comment report.md
```

In the workflow above, the `deploy-runner` step launches an EC2 `p2.xlarge`
instance in the `us-west` region. The `train-model` job then runs on the
newly-launched instance. See [Environment Variables](#environment-variables)
below for details on the `secrets` required.

🎉 **Note that jobs can use any Docker container!** To use functions such as
`cml-send-comment` from a job, the only requirement is to
[have CML installed](/doc/cml-with-npm).

## Docker Images

The CML Docker images (`docker://iterativeai/cml` or
`docker://ghcr.io/iterative/cml`) come loaded with Python, CUDA, `git`, `node`
and other essentials for full-stack data science. Different versions of these
essentials are available from different `iterativeai/cml` image tags. The tag
convention is `{CML_VER}-dvc{DVC_VER}-base{BASE_VER}{-gpu}`:

| `{BASE_VER}` | Software included (`-gpu`)                      |
| ------------ | ----------------------------------------------- |
| 0            | Ubuntu 18.04, Python 2.7 (CUDA 10.1, CuDNN 7)   |
| 1            | Ubuntu 20.04, Python 3.8 (CUDA 11.0.3, CuDNN 8) |

For example, `docker://iterativeai/cml:0-dvc2-base1-gpu`, or
`docker://ghcr.io/iterative/cml:0-dvc2-base1`.

## Options

The `cml-runner` function accepts the following arguments:

```
--help                      Show help                                [boolean]
--version                   Show version number                      [boolean]
--log                       Maximum log level
                 [choices: "error", "warn", "info", "debug"] [default: "info"]
--labels                    One or more user-defined labels for this runner
                            (delimited with commas)           [default: "cml"]
--idle-timeout              Seconds to wait for jobs before shutting down. Set
                            to -1 to disable timeout            [default: 300]
--name                      Name displayed in the repository once registered
                            cml-{ID}
--no-retry                  Do not restart workflow terminated due to instance
                            disposal or GitHub Actions timeout
                                                    [boolean] [default: false]
--single                    Exit after running a single job
                                                    [boolean] [default: false]
--reuse                     Don't launch a new runner if an existing one has
                            the same name or overlapping labels
                                                    [boolean] [default: false]
--driver                    Platform where the repository is hosted. If not
                            specified, it will be inferred from the
                            environment          [choices: "github", "gitlab"]
--repo                      Repository to be used for registering the runner.
                            If not specified, it will be inferred from the
                            environment
--token                     Personal access token to register a self-hosted
                            runner on the repository. If not specified, it
                            will be inferred from the environment
                                                            [default: "infer"]
--cloud                     Cloud to deploy the runner
                                [choices: "aws", "azure", "gcp", "kubernetes"]
--cloud-region              Region where the instance is deployed. Choices:
                            [us-east, us-west, eu-west, eu-north]. Also
                            accepts native cloud regions  [default: "us-west"]
--cloud-type                Instance type. Choices: [m, l, xl]. Also supports
                            native types like i.e. t2.micro
--cloud-gpu                 GPU type.
                                    [choices: "nogpu", "k80", "v100", "tesla"]
--cloud-hdd-size            HDD size in GB
--cloud-ssh-private         Custom private RSA SSH key. If not provided an
                            automatically generated throwaway key will be used
                                                                 [default: ""]
--cloud-spot                Request a spot instance                  [boolean]
--cloud-spot-price          Maximum spot instance bidding price in USD.
                            Defaults to the current spot bidding price
                                                               [default: "-1"]
--cloud-startup-script      Run the provided Base64-encoded Linux shell script
                            during the instance initialization   [default: ""]
--cloud-aws-security-group  Specifies the security group in AWS  [default: ""]
```

## Environment Variables

Sensitive values like cloud and repository credentials can be provided through
environment variables with the aid of GitHub
[secrets](https://docs.github.com/es/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-a-repository)
or GitLab
[masked variables](https://docs.gitlab.com/ee/ci/variables/#add-a-cicd-variable-to-a-project);
the latter also supports
[external secrets](https://docs.gitlab.com/ee/ci/secrets) for added security.

⚠️ You will need to create a
[personal access token (PAT)](#personal-access-token) with repository read/write
access and workflow privileges. In the example workflow above, this token is
stored as `PERSONAL_ACCESS_TOKEN`.

🛈 If using the `--cloud` option, you will also need to provide access
credentials for your cloud compute resources as secrets. In the above example,
`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` (with privileges to create &
destroy EC2 instances) are required.

### Personal Access Token

This token serves as a repository access credential.

<details>

#### GitHub

Use either:

- a
  [personal access token](https://docs.github.com/en/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token)
  with the **repo** scope, or
- a
  [GitHub App](https://docs.github.com/en/developers/apps/building-github-apps/authenticating-with-github-apps)
  with **Repository permissions / Administration** write permissions (for
  repository-level runners), or **Organization permissions / Self-hosted
  runners** write permissions (for organization-level runners).

</details>

<details>

#### GitLab

Use either:

- a
  [personal access token](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html)
  with the **api** scope, or
- a
  [project access token](https://docs.gitlab.com/ee/user/project/settings/project_access_tokens.html).
  This only works for project-level
  ([specific](https://docs.gitlab.com/ee/ci/runners/runners_scope.html#specific-runners)),
  but not for instance-level
  ([shared](https://docs.gitlab.com/ee/ci/runners/runners_scope.html#shared-runners))
  runners.

</details>

Ideally, you should not use personal access tokens from your own account, as
they grant access to all your repositories. Instead, it's higly recommended to
create a separate _bot account_ that only has access to the repositories where
you plan to deploy runners to. Bot accounts are
[the same](https://docs.github.com/en/get-started/learning-about-github/types-of-github-accounts#personal-user-accounts)
as normal user accounts, with the only difference being the intended use case.

### Cloud Compute Resource Credentials

Note that you will also need to provide access credentials of your compute
resources. In the above example, `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`
are required to deploy EC2 instances.

Click below to see credentials needed for supported compute providers.

<details>

#### AWS

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_SESSION_TOKEN` **(optional)**

Note that the same credentials can also be used for
[configuring cloud storage](/doc/cml-with-dvc#cloud-storage-provider-credentials).

</details>

<details>

#### Azure

- `AZURE_CLIENT_ID`
- `AZURE_CLIENT_SECRET`
- `AZURE_SUBSCRIPTION_ID`
- `AZURE_TENANT_ID`

</details>

<details>

#### GCP

Either one of:

- `GOOGLE_APPLICATION_CREDENTIALS_DATA`: the **contents** of a service account
  JSON file, or
- `GOOGLE_APPLICATION_CREDENTIALS`: the **path** to the JSON file.

The former is more convenient for CI/CD scenarios, where secrets are (usually)
provisioned through environment variables instead of files.

</details>

<details>

#### Kubernetes

- `KUBERNETES_CONFIGURATION`: the **contents** of a
  [`kubeconfig`](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig)
  file.

</details>

<details>

#### On-premise (Local) Runners

The `cml-runner` command can also be used to set up a local machine or
on-premise GPU cluster as a self-hosted runner. Simply
[install CML with NPM](/doc/cml-with-npm) and then run:

```
cml-runner \
  --repo="$REPOSITORY_URL" \
  --token="$PERSONAL_ACCESS_TOKEN" \
  --labels="local,runner" \
  --idle-timeout=180
```

The machine will listen for jobs from your repository and execute them locally.

⚠️ **Warning:** anyone with access to your repository (everybody for public
ones) may be able to execute arbitrary code on your machine. Refer to the
corresponding
[GitHub](https://docs.github.com/es/actions/learn-github-actions/security-hardening-for-github-actions#hardening-for-self-hosted-runners)
and [GitLab](https://docs.gitlab.com/runner/security/) documentation for
additional guidance.

</details>
