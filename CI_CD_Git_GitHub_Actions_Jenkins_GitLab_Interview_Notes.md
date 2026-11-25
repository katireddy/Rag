# CI/CD, Git Strategies, GitHub Actions, Jenkins, GitLab CI/CD — Interview Notes

> Practical DevOps / DevSecOps revision guide with examples, comparison tables, workflow patterns, and interview questions.

## 1. CI/CD fundamentals

### What is CI?

**Continuous Integration (CI)** is the practice of frequently integrating code changes into a shared repository. Every change is automatically validated through activities such as:

- Checkout source code
- Install dependencies
- Compile or build
- Run unit tests
- Run linting and static analysis
- Run security and dependency scans
- Publish test and scan reports

**Goal:** Find defects early and give developers fast feedback.

### What is CD?

CD has two commonly used meanings:

- **Continuous Delivery:** Every successful change is kept in a deployable state. Production deployment is usually a controlled/manual approval.
- **Continuous Deployment:** Every change that passes the required pipeline checks is automatically deployed to production.

### Typical CI/CD flow

```text
Developer changes code
        |
        v
Create branch -> Commit -> Push
        |
        v
Open Pull Request / Merge Request
        |
        v
CI: build + unit tests + lint + security scans
        |
        v
Review + required checks pass
        |
        v
Merge into main
        |
        v
Build/package + publish artifact
        |
        v
Deploy to dev -> test/stage -> approval (if required)
        |
        v
Production deployment
        |
        v
Smoke tests + monitoring + rollback if needed
```

**Important:** A pull request is a request to merge changes; it is not itself a merge. CI can run when a PR is opened, updated, or reopened. A separate pipeline can run after the PR is merged.

## 2. GitHub Actions

GitHub Actions is GitHub's built-in automation and CI/CD platform. Workflows are YAML files stored under `.github/workflows/`.

### Main concepts

| Concept | Meaning |
|---|---|
| Workflow | Complete automated process defined in YAML |
| Event / trigger | Event that starts a workflow, such as `push` or `pull_request` |
| Job | A group of steps executed on a runner |
| Step | A command or reusable action within a job |
| Runner | Machine that executes the job (GitHub-hosted or self-hosted) |
| Action | Reusable unit of automation |
| Artifact | File/output retained from a workflow run |
| Secret | Encrypted value used for credentials or tokens |
| Environment | Deployment target with optional protection rules and approvals |

### Example: run checks on push and pull request

Save as `.github/workflows/ci.yml`:

```yaml
name: Application CI

on:
  push:
    branches: [main, develop, "feature/**"]
  pull_request:
    branches: [main, develop]

permissions:
  contents: read

jobs:
  validate:
    name: Build and test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run lint
        run: |
          pip install flake8
          flake8 .

      - name: Run tests
        run: pytest -q
```

This workflow starts:
- When code is pushed to `main`, `develop`, or a branch matching `feature/**`.
- When a PR targeting `main` or `develop` is opened or updated.

### Example: run a deployment after merge to main

```yaml
name: Deploy application

on:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: ./scripts/deploy.sh
```

Configure the `production` environment in repository settings if you need required reviewers, deployment restrictions, or environment-specific secrets. Protect production credentials; do not hard-code them in YAML.

### Useful GitHub Actions triggers

| Trigger | Typical use |
|---|---|
| `push` | Validate every push to selected branches |
| `pull_request` | Validate PR creation, updates, and reopening |
| `pull_request_target` | Special PR automation that runs in the base repository context; use extreme care with untrusted code |
| `workflow_dispatch` | Manually start a workflow |
| `schedule` | Run on a cron schedule |
| `release` | Run when a release event occurs |
| `workflow_call` | Make a workflow reusable by another workflow |
| `workflow_run` | Start after another workflow completes |

### How to require green checkmarks before merge

In GitHub:

1. Open **Settings → Rules → Rulesets** (or branch protection settings).
2. Create a rule for `main`.
3. Require a pull request before merging.
4. Enable **Require status checks to pass**.
5. Select the checks/jobs that must pass, such as `Build and test`.
6. Optionally require approvals, conversation resolution, signed commits, or linear history.
7. Test by opening a PR and confirming that checks appear before merge is allowed.

The check name comes from the workflow/job context. Run the workflow at least once so GitHub can identify the status check. A green check means the configured check succeeded; it does not prove the application is free of all defects.

### GitHub Actions security practices

- Give workflows the minimum permissions using `permissions`.
- Pin third-party actions to a verified full commit SHA in high-security environments.
- Store credentials in GitHub Secrets or use OIDC federation with the cloud provider.
- Avoid exposing secrets to untrusted fork PR code.
- Review `pull_request_target` carefully; never check out and execute untrusted PR code with privileged tokens.
- Use dependency, secret, container, and code scanning where appropriate.
- Separate build and deployment permissions; protect production environments.
- Use concurrency controls to avoid overlapping deployments when necessary.

## 3. Jenkins

Jenkins is an open-source automation server used for CI/CD and general automation. It can run on a controller with build agents, and it has a large plugin ecosystem.

### Jenkins architecture

```text
Developer -> Git repository / webhook
                    |
                    v
             Jenkins Controller
             - schedules jobs
             - loads pipeline
             - manages credentials
                    |
          -----------------------
          |          |          |
          v          v          v
       Agent 1    Agent 2    Agent 3
       Build      Test       Deploy
          |          |          |
          v          v          v
       Artifacts / reports / deployment
```

- **Controller:** Coordinates jobs, stores configuration, and assigns work.
- **Agent (node):** Executes pipeline steps.
- **Executor:** A slot on a node that can run a task.
- **Plugin:** Adds integrations or capabilities.
- **Credential store:** Stores secrets for jobs and pipelines.

Do not overload the controller with builds in production setups; use appropriately isolated agents.

### Freestyle job vs Pipeline job

| Freestyle job | Pipeline job |
|---|---|
| Configured mainly through the Jenkins UI | Defined as code in a `Jenkinsfile` |
| Good for simple jobs and legacy tasks | Better for complex, repeatable CI/CD |
| Harder to review and version with application code | Can be stored and reviewed in Git |
| Complex flows can become difficult to maintain | Supports stages, parallel work, conditions, and approvals |

### Jenkins Pipeline types

- **Declarative Pipeline:** Structured syntax with `pipeline`, `agent`, `stages`, and `steps`. Usually the best default for maintainable pipelines.
- **Scripted Pipeline:** Groovy-based and more flexible, but easier to make complex and harder to standardize.

### Example: Declarative Jenkinsfile

Create a file named `Jenkinsfile` in the repository root:

```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'echo "Build application here"'
            }
        }

        stage('Test') {
            steps {
                sh 'echo "Run unit tests here"'
            }
        }

        stage('Security Scan') {
            steps {
                sh 'echo "Run SAST and dependency scans here"'
            }
        }

        stage('Deploy to Dev') {
            when {
                branch 'develop'
            }
            steps {
                sh './scripts/deploy-dev.sh'
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed. Publish test reports here.'
        }
        success {
            echo 'Pipeline succeeded'
        }
        failure {
            echo 'Pipeline failed'
        }
    }
}
```

The commands are illustrative. Replace them with the actual build, test, scan, and deployment commands for the project.

### Example: Scripted Pipeline

```groovy
node {
    stage('Checkout') {
        checkout scm
    }

    stage('Build and Test') {
        sh 'echo "Build and test"'
    }

    stage('Deploy') {
        if (env.BRANCH_NAME == 'main') {
            sh './scripts/deploy.sh'
        }
    }
}
```

### Jenkins + Terraform + Ansible example

A common infrastructure pipeline is:

```text
Git push / PR
   |
   v
Jenkins pipeline
   |
   +--> Terraform fmt / validate
   |
   +--> Terraform plan
   |
   +--> Approval before infrastructure changes
   |
   +--> Terraform apply
   |
   +--> Ansible configuration / application setup
   |
   +--> Smoke tests
```

Illustrative Declarative Pipeline:

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Terraform Validate') {
            steps {
                dir('terraform') {
                    sh 'terraform fmt -check'
                    sh 'terraform init -input=false'
                    sh 'terraform validate'
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                dir('terraform') {
                    sh 'terraform plan -out=tfplan'
                }
            }
        }

        stage('Approval') {
            steps {
                input message: 'Approve infrastructure deployment?'
            }
        }

        stage('Terraform Apply') {
            steps {
                dir('terraform') {
                    sh 'terraform apply -input=false tfplan'
                }
            }
        }

        stage('Ansible Configure') {
            steps {
                sh 'ansible-playbook -i inventory/hosts.ini playbooks/configure.yml'
            }
        }
    }
}
```

**Production cautions:**
- Use a remote Terraform backend with locking and controlled state access.
- Do not commit state files, plan files containing sensitive values, or credentials.
- Use separate identities and least-privilege permissions.
- Add approval and policy checks before impactful `apply` operations.
- Ensure the Ansible inventory and SSH credentials are securely managed.
- Consider running `terraform plan` on PRs, but do not automatically apply unreviewed PR changes.

### Why Jenkins plugins matter

Plugins integrate Jenkins with source control, cloud platforms, test tools, notifications, artifact repositories, and security scanners. Examples include Git, Pipeline, Credentials Binding, JUnit, Kubernetes, and Slack integrations.

Plugin considerations:
- Plugins need maintenance and security updates.
- Too many or incompatible plugins can create operational overhead.
- Restrict who can install or configure plugins.
- Keep Jenkins core and plugins patched.
- GitHub Actions and GitLab CI also have reusable actions/components and integrations; they are not limited to built-in features.

## 4. GitLab CI/CD

GitLab CI/CD is integrated into GitLab. Pipeline configuration is usually stored in `.gitlab-ci.yml` at the repository root.

### Main concepts

| Concept | Meaning |
|---|---|
| Pipeline | Complete CI/CD execution |
| Stage | Logical phase such as build, test, deploy |
| Job | Unit of work within a stage |
| Runner | Agent that executes jobs |
| Artifact | Output retained and passed between jobs |
| Cache | Reusable downloaded dependencies or build data |
| Variables | Configuration values and secrets |
| Environment | Deployment destination such as staging or production |
| Include | Reuse YAML configuration from another file/project |
| Rules | Conditions controlling whether a job runs |

### Example: GitLab CI pipeline

```yaml
stages:
  - validate
  - test
  - deploy

workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH'

lint:
  stage: validate
  image: python:3.12
  script:
    - pip install flake8
    - flake8 .

unit_tests:
  stage: test
  image: python:3.12
  script:
    - pip install -r requirements.txt
    - pytest -q
  artifacts:
    when: always
    reports:
      junit: report.xml

deploy_production:
  stage: deploy
  image: alpine:latest
  script:
    - ./scripts/deploy.sh
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

This example is a template. The test command must generate `report.xml` for the JUnit report to be useful; add the appropriate test-runner option, for example `pytest --junitxml=report.xml`.

### GitLab runners and executors

A GitLab Runner picks up jobs and executes them. Depending on configuration, it can use executors such as Docker, Kubernetes, Shell, or other supported environments. Shared runners may be managed by GitLab; self-managed runners are operated by your organization.

## 5. GitHub Actions vs Jenkins vs GitLab CI/CD

| Area | GitHub Actions | Jenkins | GitLab CI/CD |
|---|---|---|---|
| Platform | Integrated with GitHub | Separate automation server | Integrated with GitLab |
| Pipeline definition | YAML in `.github/workflows/` | `Jenkinsfile` (Declarative or Scripted Groovy) | YAML in `.gitlab-ci.yml` |
| Execution | GitHub-hosted or self-hosted runners | Controller and agents | GitLab-managed or self-managed runners |
| Reuse | Actions and reusable workflows | Shared libraries and plugins | Includes, components, templates |
| Integrations | Marketplace actions and native GitHub features | Extensive plugin ecosystem | Integrated GitLab features and external integrations |
| PR/MR checks | Pull request events and status checks | Webhooks and branch-source integrations | Merge request pipelines |
| Configuration | Usually quick to start for GitHub repos | More server/admin setup | Usually quick to start for GitLab repos |
| Customization | Flexible, with hosted runner constraints | Very high flexibility and control | Strong YAML-based pipeline features |
| Operations | GitHub manages hosted runners | You manage Jenkins controller/plugins/agents unless using a managed service | Runner management depends on hosted/self-managed setup |
| Best fit | Teams already using GitHub | Complex, highly customized or legacy automation | Teams using GitLab for source, review, and delivery |

### Which one should I choose?

- **GitHub Actions:** A natural choice when source code and reviews are already on GitHub and you want repository-native workflows.
- **Jenkins:** A strong choice when you need extensive customization, existing Jenkins investment, specialized agents, or a broad plugin ecosystem.
- **GitLab CI/CD:** A natural choice when GitLab is the central platform for repositories, merge requests, CI, security, and deployment workflows.

There is no universal winner. Compare hosting model, security controls, integrations, maintenance effort, cost, scale, and team skills.

## 6. Git branching strategies

A branching strategy defines how developers isolate work, collaborate, review changes, release software, and handle production fixes.

### 6.1 Trunk-based development

- Developers integrate small changes frequently into a shared main/trunk branch.
- Short-lived feature branches may be used, often merged within a day or a few days.
- Feature flags can hide incomplete functionality.
- Automated tests and fast CI are essential.

```text
main:   A---B---C---D---E
             \   /
feature:      X-Y
```

**Benefits:** Less long-lived divergence, faster integration, simpler release flow.

**Risks:** Requires strong automated tests, disciplined small changes, and feature-flag practices.

### 6.2 GitHub Flow

A simple model often used for continuously delivered applications:

1. Keep `main` deployable and protected.
2. Create a short-lived feature branch.
3. Commit and push changes.
4. Open a pull request.
5. Run automated checks and obtain review.
6. Merge to `main`.
7. Deploy from `main`.

```text
main:       A----------------M
             \              /
feature:      B---C---D-----/
```

### 6.3 Git Flow

A more structured model with longer-lived branches:

- `main`: production releases.
- `develop`: integration for the next release.
- `feature/*`: new functionality.
- `release/*`: release stabilization.
- `hotfix/*`: urgent production fixes.

```text
main:      A-------------------R------H
            \                 /       /
develop:     B---F1---F2------/---F3--/
                  \          /
feature:           X---Y----
```

**Benefits:** Clear release and hotfix lanes for products with scheduled releases.

**Risks:** More branches and merge complexity; can be heavy for teams deploying continuously.

### 6.4 Forking workflow

Common in open-source projects and cross-team contribution models:

1. The contributor creates a fork of the upstream repository.
2. They clone their fork and create a feature branch.
3. They push changes to their fork.
4. They open a PR from the fork to the upstream repository.
5. Maintainers review checks and merge into the upstream repository.

```text
Upstream repository (canonical)
        |
        +---- fork: contributor A
        |          |
        |          +--- feature branch -> PR to upstream
        |
        +---- fork: contributor B
                   |
                   +--- bugfix branch -> PR to upstream
```

The upstream repository remains the canonical source. A fork is a separate repository copy, not merely another branch. Fork-based workflows are especially useful when contributors should not have write access to the main repository.

### 6.5 Release branches

A team may create `release/2.4` from a stable point while development continues on `main`. Only release fixes go into that branch. Fixes may then be merged or cherry-picked back to `main`.

### 6.6 Hotfix branches

For a production issue:
1. Branch from the production baseline (often `main` or the release tag).
2. Make the smallest safe fix.
3. Run tests and security checks.
4. Review and deploy.
5. Merge/cherry-pick the fix back into active development branches.

### Branching strategy comparison

| Strategy | Branch model | Good for | Main trade-off |
|---|---|---|---|
| Trunk-based | One shared trunk; short-lived branches | Frequent releases, mature CI | Needs excellent test discipline |
| GitHub Flow | Main + short-lived feature branches | Web apps and continuous delivery | Less formal release separation |
| Git Flow | Main + develop + feature/release/hotfix | Scheduled/versioned releases | More process and merge overhead |
| Forking workflow | Separate repository forks + PRs | Open source and external contributors | Syncing forks adds work |
| Release branches | Main development plus supported release lines | Multiple maintained versions | Backports need careful coordination |

**Interview clarification:** Protected branches are not a separate branching strategy. Branch protection is a repository policy applied to branches such as `main` or `release/*`. It can require PR reviews, passing status checks, signed commits, and restricted pushes.

## 7. Essential Git commands and concepts

| Command | Purpose |
|---|---|
| `git clone URL` | Copy a remote repository locally |
| `git status` | Show working tree and staging status |
| `git add file` | Stage changes |
| `git commit -m "message"` | Create a local commit |
| `git push` | Send local commits to a remote |
| `git fetch` | Download remote refs/objects without merging |
| `git pull` | Fetch and integrate remote changes |
| `git switch -c feature/x` | Create and switch to a branch |
| `git merge feature/x` | Merge another branch into the current branch |
| `git rebase main` | Replay current branch commits on top of main |
| `git cherry-pick SHA` | Apply a selected commit |
| `git revert SHA` | Create a new commit that undoes a commit |
| `git reset` | Move HEAD/branch and optionally change index/working tree |
| `git log --oneline --graph --all` | View commit history graphically |
| `git diff` | Compare changes |

### Merge vs rebase vs cherry-pick vs revert vs reset

- **Merge:** Combines histories, usually preserving branch topology; may create a merge commit.
- **Rebase:** Replays commits on a new base; rewrites commit IDs. Avoid rebasing shared/public history unless the team explicitly agrees.
- **Cherry-pick:** Applies one or more selected commits onto the current branch.
- **Revert:** Safely undoes a change by adding a new commit; useful on shared branches.
- **Reset:** Moves the current branch pointer. `--soft`, `--mixed` (default), and `--hard` affect the index and working tree differently. `--hard` can discard local changes; use carefully.

## 8. Linux interview revision

### Linux fundamentals

**Linux** is an open-source, Unix-like operating system kernel. A Linux distribution combines the kernel with system utilities, package management, libraries, and other software. Examples include Ubuntu, Debian, and Red Hat Enterprise Linux (RHEL).

### Important directories

| Directory | Purpose |
|---|---|
| `/` | Root of the filesystem |
| `/bin` | Essential user commands (may be merged into `/usr/bin`) |
| `/sbin` | System administration commands (may be merged into `/usr/sbin`) |
| `/boot` | Bootloader and kernel-related files |
| `/dev` | Device files |
| `/etc` | System configuration |
| `/home` | Users' home directories |
| `/root` | Root user's home directory |
| `/var` | Variable data such as logs and spool files |
| `/tmp` | Temporary files |
| `/usr` | User applications, libraries, and shared data |
| `/opt` | Optional/add-on software |
| `/proc` | Virtual filesystem exposing process and kernel information |
| `/sys` | Kernel/device information interface |
| `/run` | Runtime state since boot |
| `/mnt` | Temporary mount point |
| `/media` | Removable media mount points |
| `/lib` | Essential shared libraries (often merged into `/usr/lib`) |

### Common Linux commands

```bash
# Files and directories
pwd
ls -lah
cd /var/log
mkdir demo
cp source.txt destination.txt
mv old.txt new.txt
rm file.txt
find /var/log -type f -name "*.log"
grep -R "ERROR" /var/log

# Permissions and ownership
chmod 640 file.txt
chown user:group file.txt
umask

# Processes and resources
ps aux
top
pgrep nginx
kill -15 <PID>
free -h
df -h
du -sh /var/log
uptime

# Logs and services (systemd systems)
journalctl -u nginx
journalctl -xe
systemctl status nginx
systemctl restart nginx
systemctl enable nginx

# Networking
ip addr
ip route
ss -lntup
ss -plant
dig example.com
nslookup example.com
curl -I https://example.com
curl -v https://example.com
ping -c 4 example.com
traceroute example.com
nc -vz example.com 443

# Archives and storage
tar -czf logs.tar.gz /var/log
tar -xzf logs.tar.gz
lsblk
mount
```

Use `ip addr` rather than Windows `ipconfig` on standard Linux systems. `ss -lntup` displays listening TCP/UDP sockets with numeric ports and process information (process details may require elevated privileges).

### Linux permissions

A permission string such as `-rwxr-xr--` represents:
- Owner: `rwx` (read, write, execute)
- Group: `r-x`
- Others: `r--`

Numeric values: read = 4, write = 2, execute = 1. For example, `chmod 754 script.sh` means owner `rwx`, group `r-x`, others `r--`.

### Linux networking quick reference

| Command | What it helps diagnose |
|---|---|
| `ip addr` | Interface IP addresses |
| `ip route` | Routing table/default gateway |
| `ss -lntup` | Listening ports and sockets |
| `dig` | DNS lookup details |
| `curl -I URL` | HTTP response headers |
| `curl -v URL` | Connection/TLS/HTTP troubleshooting |
| `nc -vz host port` | TCP port connectivity |
| `ping` | Basic reachability/latency (may be blocked) |
| `traceroute` | Network path troubleshooting |

## 9. Common interview questions and short answers

### Linux

1. **What is Linux?**  
   An open-source, Unix-like operating system kernel; distributions package it with utilities and software.

2. **What is the difference between a process and a thread?**  
   A process has its own address space and resources; threads are execution units within a process that share process resources.

3. **How do you check CPU and memory usage?**  
   Use `top`/`htop`, `ps`, `free -h`, and tools such as `vmstat`.

4. **How do you check disk space?**  
   `df -h` shows filesystem usage; `du -sh path` shows directory/file usage.

5. **How do you check listening ports?**  
   `ss -lntup` (use `sudo` if process details are restricted).

6. **How do you identify a process using a port?**  
   `sudo ss -lntup` or `sudo lsof -i :PORT`.

7. **What is the difference between `kill` and `kill -9`?**  
   `kill` normally sends SIGTERM, allowing graceful shutdown. `kill -9` sends SIGKILL and cannot be caught or handled; use it only when necessary.

8. **What is DNS?**  
   DNS resolves names such as `example.com` to IP addresses and can provide other records.

9. **What is the difference between TCP and UDP?**  
   TCP provides a reliable, ordered byte stream; UDP is connectionless and does not guarantee delivery or ordering.

10. **How do you troubleshoot a website that is unreachable?**  
    Check DNS (`dig`), route/interface (`ip addr`, `ip route`), connectivity (`curl`, `nc`), listening service (`ss`), firewall, service logs, and application health.

11. **What is a symbolic link vs hard link?**  
    A symbolic link points to a path and can cross filesystems; a hard link is another directory entry for the same inode and generally cannot cross filesystems.

12. **What is cron?**  
    A scheduler for recurring commands; inspect with `crontab -l` and edit with `crontab -e`.

13. **What is a zombie process?**  
    A process that has exited but whose parent has not yet collected its exit status.

14. **What is load average?**  
    A measure of runnable and uninterruptible tasks averaged over 1, 5, and 15 minutes; interpret it relative to CPU count.

### Git and GitHub

1. **What is Git?**  
   A distributed version control system that records repository history.

2. **Git vs GitHub?**  
   Git is the version-control tool; GitHub is a hosted collaboration platform built around Git repositories.

3. **What is a branch?**  
   A movable reference to a commit, used to isolate work.

4. **What is a pull request?**  
   A proposal to merge changes from one branch/repository into another, with review and automated checks.

5. **What is the difference between `fetch` and `pull`?**  
   Fetch downloads remote changes without integrating them; pull fetches and then integrates, typically by merge or rebase depending on configuration.

6. **Merge vs rebase?**  
   Merge joins histories; rebase replays commits on a new base and rewrites commit IDs.

7. **Revert vs reset?**  
   Revert creates a new undo commit; reset moves a branch pointer and can alter staged/working files.

8. **What is a merge conflict?**  
   Git cannot automatically reconcile overlapping changes; a developer must resolve and test the result.

9. **What is a protected branch?**  
   A branch with policies such as PR-only changes, required reviews, status checks, or restricted direct pushes.

10. **What is a fork?**  
    A separate repository copy under another account or namespace, often used for external contributions.

11. **What is a tag?**  
    A named reference, commonly used to mark releases.

12. **What is `.gitignore`?**  
    A file describing untracked files Git should ignore; it does not automatically untrack files already committed.

### GitHub Actions

1. **What is GitHub Actions?**  
   GitHub's automation and CI/CD platform using YAML workflows.

2. **Where are workflow files stored?**  
   In `.github/workflows/`.

3. **What is the difference between a workflow, job, and step?**  
   A workflow is the full automation; a job is a unit of work; a step is a command or action inside a job.

4. **What is a runner?**  
   The machine that executes workflow jobs.

5. **How do you trigger a workflow on push and PR?**  
   Configure `on: push` and `on: pull_request` with the required branch filters.

6. **How do you prevent merging when tests fail?**  
   Configure branch protection/rulesets to require the relevant status checks.

7. **How do you store secrets?**  
   Use GitHub Secrets or a supported OIDC-based identity flow; never hard-code credentials.

8. **What is a reusable workflow?**  
   A workflow callable by another workflow using `workflow_call`.

9. **What is an artifact?**  
   A file or output uploaded from a workflow run for later download or use.

10. **What is the difference between `pull_request` and `pull_request_target`?**  
    `pull_request` runs in the PR context with restrictions; `pull_request_target` runs with base-repository context and can have elevated access, so untrusted PR code must not be executed with its privileges.

### Jenkins

1. **What is Jenkins?**  
   An automation server commonly used to implement CI/CD pipelines.

2. **What is the Jenkins controller-agent architecture?**  
   The controller schedules and coordinates jobs; agents execute the work.

3. **What is a Jenkinsfile?**  
   A pipeline-as-code file, commonly stored in the source repository.

4. **Declarative vs Scripted Pipeline?**  
   Declarative is structured and opinionated; Scripted is Groovy-based and more flexible.

5. **What is a stage?**  
   A logical phase of a pipeline, such as Build, Test, or Deploy.

6. **What are Jenkins plugins?**  
   Extensions that add integrations and capabilities; they require compatibility and security maintenance.

7. **How do you store credentials?**  
   Use Jenkins Credentials and bind them securely at runtime; do not print secrets in logs.

8. **How can Jenkins trigger on a Git push?**  
   Configure a repository webhook or supported SCM integration and a suitable multibranch/pipeline job.

9. **How do you run Terraform through Jenkins?**  
   Checkout code, run `fmt`/`validate`/`plan`, apply policy and approval gates, then run `apply` with secure backend credentials.

10. **How do you run Ansible through Jenkins?**  
    Use an agent with Ansible installed, securely provide inventory and SSH credentials, then run `ansible-playbook`.

11. **What is a Jenkins agent label?**  
    A selector used to route a job to agents with the required operating system, tools, or capabilities.

12. **What is a post block?**  
    Declarative Pipeline behavior that runs after pipeline/stage completion, for example always, success, or failure actions.

### GitLab CI/CD

1. **Where is GitLab pipeline configuration stored?**  
   In `.gitlab-ci.yml`.

2. **What is a GitLab Runner?**  
   An agent that picks up and executes GitLab CI jobs.

3. **What is a stage vs job?**  
   A stage groups pipeline phases; jobs are the actual tasks executed in those phases.

4. **How do jobs run in stages?**  
   By default, stages run in order and jobs in the same stage can run in parallel, subject to runner capacity and dependencies.

5. **What are artifacts?**  
   Outputs retained from jobs and optionally passed to later jobs.

6. **What is a cache?**  
   Reusable data, often dependencies, intended to speed up jobs; it is not a substitute for build artifacts.

7. **What are `rules`?**  
   Conditions that decide whether a job is included and how it is run.

8. **How do you run a pipeline for a merge request?**  
   Configure merge request pipeline rules and ensure the project is set up for the desired MR pipeline behavior.

9. **What is an environment?**  
   A named deployment target such as development, staging, or production.

10. **How do you protect production deployment?**  
    Use protected environments/branches, restricted variables, approvals where supported, and least-privilege runner credentials.

## 10. Scenario-based interview questions

### Scenario 1: A developer pushes code, but no GitHub Actions workflow runs. What do you check?

- Is the YAML file under `.github/workflows/`?
- Is the YAML valid?
- Does the `on` trigger match the event and branch?
- Is Actions enabled for the repository/organization?
- Are path filters excluding the changed files?
- Is the workflow disabled or blocked by policy?
- Is the event from a fork subject to permission restrictions?

### Scenario 2: A PR shows a red check. How do you troubleshoot?

- Open the failing workflow run and identify the failed job/step.
- Read the logs and reproduce locally where possible.
- Check dependencies, environment variables, permissions, runner image, and flaky tests.
- Fix the cause, push an update, and verify the new run.
- Do not bypass required checks without an approved exception.

### Scenario 3: A production deployment failed.

- Stop or prevent additional deployments if necessary.
- Identify the failed stage and impact.
- Check logs, health checks, recent changes, and infrastructure status.
- Roll back to a known-good release or use a tested recovery plan.
- Verify service health and communicate status.
- Complete root-cause analysis and add preventive tests/alerts.

### Scenario 4: Terraform plan wants to replace a production database.

- Do not apply automatically.
- Inspect the plan and identify which configuration change forces replacement.
- Verify state, provider version, lifecycle settings, and resource dependencies.
- Review backups and recovery strategy.
- Obtain the required change approval.
- Apply only after confirming the impact and maintenance plan.

### Scenario 5: A secret appears in a pipeline log.

- Treat it as compromised.
- Revoke/rotate it immediately.
- Review access and audit logs for misuse.
- Remove unsafe logging and fix masking/secret handling.
- Review repository history and artifacts; deleting a log line alone does not make an exposed credential safe.

## 11. A concise interview answer: explain an end-to-end CI/CD design

> "When a developer pushes code or opens a pull request, the CI pipeline checks out the code, installs dependencies, runs linting, unit tests, static analysis, dependency and secret scans, and builds an artifact. The pull request is protected by required status checks and code reviews, so it cannot be merged until the required checks pass. After merge to the protected main branch, a release pipeline publishes a versioned artifact and deploys it to the development or staging environment. We run smoke/integration tests and require approval before production if the release process needs a gate. Infrastructure can be provisioned using Terraform, and configuration or application setup can be handled with Ansible. Credentials are stored securely, permissions are least-privilege, and deployment health checks and rollback procedures are included."

## 12. Quick revision checklist

- [ ] Explain CI, Continuous Delivery, and Continuous Deployment.
- [ ] Explain workflow, job, step, action, runner, and artifact.
- [ ] Write a GitHub Actions workflow for `push` and `pull_request`.
- [ ] Explain required status checks and protected branches.
- [ ] Explain Jenkins controller, agent, plugin, and Jenkinsfile.
- [ ] Compare Declarative and Scripted Jenkins pipelines.
- [ ] Explain GitLab stages, jobs, runners, artifacts, cache, and rules.
- [ ] Compare GitHub Actions, Jenkins, and GitLab CI/CD.
- [ ] Explain trunk-based, GitHub Flow, Git Flow, and fork-based workflows.
- [ ] Explain merge, rebase, cherry-pick, revert, and reset.
- [ ] Practise Linux processes, permissions, logs, storage, and networking commands.
- [ ] Explain Terraform plan/apply safety and Ansible playbook execution.
- [ ] Explain secrets, least privilege, branch protection, approvals, and rollback.

---
**Interview tip:** Do not only define tools. Explain one real flow end to end: developer push → PR checks → review → merge → artifact → deployment → verification. Be clear about which steps run on a PR and which run only after merge.
