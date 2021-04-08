# Business Central git hooks in bash
Business Central is able to push changes into remote git repositories utilizing post-commit git hooks.
This project offers a bash-based implementation for such git hooks.


## Using the Git Hooks in OpenShift
The Githooks themselves are designed to be environment agnostic, but the roll out of these is targeted to the OpenShift environment. To install and use these, a remote transfer to the Persistent Volume is required. The mount point within the Decision Center Pod is found at /opt/kie/data/configuration and would need both files copied/replaced for any updates. An ```scp``` session to the currently running pod would be enough to access the /opt/kie/data/configuration mount point and then on each configuration restart, the configuration would be present.


> **IMPORTANT** To utilize the Githooks and configuration, the files will need to be uploaded to the Persistent Volume attached to the rhdm-centr deployment. 

## Features
* Lightweight, relies only on standard git client and bash
* Works with any git provider, e.g. [GitLab](https://gitlab.com/), [GitHub](https://github.com/), [Bitbucket](https://bitbucket.org/), [Azure DevOps Repos](https://azure.microsoft.com/en-gb/services/devops/repos/), [Gitea](https://gitea.io/en-us/), [Gogs](https://gogs.io/), etc
* Can push to different git repository per project
* Supports run-of-the-mill git operations such as create a new project, create branch, commit/push changes to branches
* Works on Linux, Windows (on a Cygwin environment), probably on Mac (not tested)
* Scripted or manual installation mode
* Configurable logging of operations
## Configuration
**bcgithook** will look for its configuration in file `gitlab.conf` placed in `$HOME` directory. This file must be present event if per-project configuration files are used. The following variables need to be configured:

|Variable|Type|Content|
|--|--|--|
|`GIT_USER_NAME` | **required** | The ID for the git repo you are using. Surround the value in single quotation marks. |
|`GIT_PASSWD` | **required** | The password for the git repo. Surround the value in single quotation marks. |
|`GIT_URL` | **required** | The URL to connect to the git repo. See below for examples for various Git repos. Surround the value in single quotation marks.|
|`GIT_TYPE` | optional | Leave blank or undefined for all Git repos. Use **"azure"** (in quotation marks) for Azure DevOps if Florida Blue were to migrate away from this |
|`LOG_LOCATION` | optional | The directory where logs should be written. Defaults to `$HOME/logs` |
|`LOG_SYSTEM_REPOS` | optional | If set to "yes" will log access to system repositories, can result in some verbosity |
|`BRANCH_ALLOW`| optional | A comma-separated list, or a regular expession, of branches to allow commits to be pushed to.  When using a comma-separated list of branches, do not leave space between the comma and the branch name or enclose in quotes. Right now this is not set for the project - but can be updated. The service account has maintainer access to the repos, so it can publish to any branch. If this is not the desired behavior this will need to be adjsuted |
|`BRANCH_ALLOW_REGEX`| optional | A regular expression specifying branches to allow commits for. If defined overrides `BRANCH_ALLOW`. Use quotes to specify a regular expression, for example `"^(feature/)\|^(hotfix)"` |
|`BRANCH_DENY`| optional | a comma-separated list of branches to deny commits to be pushed to. Do not leave space between the comma and the branch name or enclose in quotes. |

See below for example configurations for various Git repos.

### Commits per branch
Post-commit git hooks by default will push all commits in a branch to the configured remote git repository. This behaviour can be modified by declaring ALLOW and DENY lists.

ALLOW and DENY lists refer to branches that the post-commit git hooks will selectively push commits to according to the following rules:

* Both lists are optional. You can define either ALLOW or DENY, both or none.
* If present, ALLOW_REGEX will override ALLOW.
* If no list is defined post-commit git hooks will by default allow all commits to be pushed to the remote git repo.
* If only the ALLOW (or ALLOW_REGEX) list is defined, commits will be pushed to the remote git repo only for branches that can be found in this list.
* If only the DENY list is defined, commits to branches that can be found in this list will NOT be pushed to the remote git repo.
* If both ALLOW (or ALLOW_REGEX) and DENY lists are defined, then the DENY list takes precedence. If a branch can be found at both the ALLOW (or ALLOW_REGEX) and DENY lists, then commits to that branch will not be pushed to the remote git repo.

**Example 1: Separate branches in ALLOW and DENY lists**

| Definition | Expected Action
|-|-|
|`BRANCH_ALLOW=branch2,feature/fa`|Commits in branches "branch2" and "feature/fa" will be pushed to the remote git repo.|
|`BRANCH_DENY=master,release`|Commits in branches "master" and "release" will not be pushed to the remote git repo.|

**Example 2: Some branches in both ALLOW and DENY lists**

| Definition | Expected Action
|-|-|
|`BRANCH_ALLOW=branch2,feature/fa`|Commits in branch "feature/fa" will be pushed to the remote git repo. Branch "branch2" is also declared at `BRANCH_DENY` which takes precedence.|
|`BRANCH_DENY=master,release,branch2`|Commits in branches "master", "release" and "branch2" will not be pushed to the remote git repo.|

**Example 3: Separate branches in ALLOW and DENY lists, with regular expressions**

| Definition | Expected Action
|-|-|
|`BRANCH_ALLOW_REGEX="^(hotfix/)\|^(feature/)"`| Specifies branches to allow commits for in the form of regular expressions. In this example commits in branches starting with `hotfix/` or `feature/` will be allowed to be pushed to the remote git repo |
|`BRANCH_ALLOW="branch2,feature/fa"`| This variable is overriden by `BRANCH_ALLOW_REGEX` and will be ignored |
|`BRANCH_DENY=master,release`| Commits in branches "master" and "release" will not be pushed to the remote git repo.|

### per-project configuration
**bcgithook** allows for different configuration per-project. For this to happen a file with the same name as the project having the `.conf` suffix should be placed in `$HOME` directory. `gitlab.conf` can be used as a template. 

> Please follow case sensitivity rules for your operating system when naming configuration files.

For new projects you can create the configuration beforehand so when BusinessCentral creates them the project specific configuration will be used automatically. That way different projects created in BusinessCentral can be associated to different repositories.

Please note that projects imported in Business Central will always be associated with the git repository they were imported from.

## Installation


* Create default configuration in file `$HOME/gitlab.conf` on OpenShift Persistent Volume (mount point on the pod by default is `/opt/kie/data/configuration`). 
* Modify the deployment configuration in `digital-rhdmcentr` by adding the following system property
```yaml
            - name: GIT_HOOKS_DIR
              value: /opt/kie/data/configuration
```
* Copy the `post-commit` script into the `GIT_HOOKS_DIR` directory
* Modify the contents of **bcgithook** configuration in `$HOME/gitlab.conf` to match your needs before triggering a pod restart.

> **bcgithook** can be installed at anytime after Business Central is used, but post-commit git hooks will only be applied to projects created (or imported) after *bcgithook* the post-commit is added to the configuration. For existing projects, it is recommended to delete and reimport projects frequently from the spaces because of other environments needing to be synced appropriately. If the project does not have the corresponding history or a merge conflict occurs, the hook will fail.

## Notes on Git Repos
### GitLab
Pushing to GitLab works without any additional configuration.
When a project is created in Business Central it will be pushed to a same-named repository to Gitlab. The way that the project is configured right now, the project is expected to be found in the *git-digital-mlp* group. 

Example configuration:
```bash
GIT_TYPE=""
GIT_USER_NAME=gitlab_id
GIT_PASSWD=passwd
GIT_URL='https://gitlab.consulting.redhat.com/ba-nacomm'
```
replace `gitlab_id` with your GitLab service account Id. Do not put a trailing `/` in the `GIT_URL`. By not specifying a specific project in `GIT_URL` you can reuse the configuration for multiple projects. The Githook is currently setup to just utilize the *git-digital-mlp* group for Gitlab. Meaning whatever project is created in Business Central would need an associated project created in Gitlab. 

> **IMPORTANT** The project needs to allow *GIT_USER_NAME* maintainer access to the corresponding project or else the hooks will fail. 


## Compatibility
**bcgithook** should be compatible with all versions of [RHPAM](https://developers.redhat.com/products/rhpam/overview), [jBPM](https://www.jbpm.org/) and [Drools](https://www.drools.org/) but it had only been tested with the following on 
* OpenShift 3.11 - RHPAM  7.7
* OpenShift 4.5 - RHPAM 7.7, 7.8


## Other Implementations

Other implementations providing git hook support for Business Central are:
* [bc-git-integration-push](https://github.com/porcelli/bc-git-integration-push) Java based
