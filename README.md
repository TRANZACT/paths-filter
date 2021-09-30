# paths-filter

[GitHub Action](https://github.com/features/actions) that enables conditional execution of workflow steps and jobs, based on the files modified by pull request, on a feature
branch, or by the recently pushed commits.

Run slow tasks like integration tests or deployments only for changed components. It saves time and resources, especially in monorepo setups.
GitHub workflows built-in [path filters](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#onpushpull_requestpaths)
don't allow this because they don't work on a level of individual jobs or steps.

## Supported workflows

- **Pull requests:**
  - Workflow triggered by **[pull_request](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#pull_request)**
    or **[pull_request_target](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#pull_request_target)** event
  - Changes are detected against the pull request base branch
  - Uses GitHub REST API to fetch a list of modified files
- **Feature branches:**
  - Workflow triggered by **[push](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#push)**
  or any other **[event](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows)**
  - The `base` input parameter must not be the same as the branch that triggered the workflow
  - Changes are detected against the merge-base with the configured base branch or the default branch
  - Uses git commands to detect changes - repository must be already [checked out](https://github.com/actions/checkout)
- **Master, Release, or other long-lived branches:**
  - Workflow triggered by **[push](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#push)** event
  when `base` input parameter is the same as the branch that triggered the workflow:
    - Changes are detected against the most recent commit on the same branch before the push
  - Workflow triggered by any other **[event](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows)**
  when `base` input parameter is commit SHA:
    - Changes are detected against the provided `base` commit
  - Workflow triggered by any other **[event](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows)**
  when `base` input parameter is the same as the branch that triggered the workflow:
    - Changes are detected from the last commit
  - Uses git commands to detect changes - repository must be already [checked out](https://github.com/actions/checkout)
- **Local changes**
  - Workflow triggered by any event when `base` input parameter is set to `HEAD`
  - Changes are detected against the current HEAD
  - Untracked files are ignored

## Inputs
| Parameter               | Is Required | Default           | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ----------------------- | ----------- | ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `filters`             | true        |                   | Path to the configuration file or YAML string with filters definition.                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `list-files`          | true        | none              | Enables listing of files matching the filter: <br>'none'  - Disables listing of matching files (default). <br>'csv'   - Coma separated list of filenames. <br>'json'  - Serialized as JSON array. <br>'shell' - Space delimited list usable as command line argument list in linux shell. If needed it uses single or double quotes to wrap filename with unsafe characters. <br>'escape'- Space delimited list usable as command line argument list in linux shell. Backslash escapes every potentially unsafe character. |
| `token`               | false       | ${{github.token}} | GitHub Access Token                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `working-directory`   | false       |                   | Relative path under $GITHUB\_WORKSPACE where the repository was checked out.                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| `ref`                 | false       |                   | Git reference (e.g. branch name) from which the changes will be detected.<br>This option is ignored if action is triggered by pull\_request event.                                                                                                                                                                                                                                                                                                                                                                         |
| `base`                | false       |                   | Git reference (e.g. branch name) against which the changes will be detected. Defaults to repository default branch (e.g. master).<br>If it references same branch it was pushed to, changes are detected against the most recent commit before the push.<br>This option is ignored if action is triggered by pull\_request event.                                                                                                                                                                                          |
| `initial-fetch-depth` | false       | 100               | How many commits are initially fetched from base branch.<br>If needed, each subsequent fetch doubles the previously requested number of commits until the merge-base is found or there are no more commits in the history.<br>This option takes effect only when changes are detected using git against different base branch.                                                                                                                                                                                             |

## Outputs
| Output     | Description           |
| ---------- | --------------------- |
| `changes`  | JSON array with names of all filters matching any of changed files |

## Usage Examples

```yaml
- uses: TRANZACT/paths-filter@v2
  with:
    # Defines filters applied to detected changed files.
    # Each filter has a name and a list of rules.
    # Rule is a glob expression - paths of all changed
    # files are matched against it.
    # Rule can optionally specify if the file
    # should be added, modified, or deleted.
    # For each filter, there will be a corresponding output variable to
    # indicate if there's a changed file matching any of the rules.
    # Optionally, there can be a second output variable
    # set to list of all files matching the filter.
    # Filters can be provided inline as a string (containing valid YAML document),
    # or as a relative path to a file (e.g.: .github/filters.yaml).
    # Filters syntax is documented by example - see examples section.
    filters: ''

    # Branch, tag, or commit SHA against which the changes will be detected.
    # If it references the same branch it was pushed to,
    # changes are detected against the most recent commit before the push.
    # Otherwise, it uses git merge-base to find the best common ancestor between
    # current branch (HEAD) and base.
    # When merge-base is found, it's used for change detection - only changes
    # introduced by the current branch are considered.
    # All files are considered as added if there is no common ancestor with
    # base branch or no previous commit.
    # This option is ignored if action is triggered by pull_request event.
    # Default: repository default branch (e.g. master)
    base: ''

    # Git reference (e.g. branch name) from which the changes will be detected.
    # Useful when workflow can be triggered only on the default branch (e.g. repository_dispatch event)
    # but you want to get changes on a different branch.
    # This option is ignored if action is triggered by pull_request event.
    # default: ${{ github.ref }}
    ref:

    # How many commits are initially fetched from the base branch.
    # If needed, each subsequent fetch doubles the
    # previously requested number of commits until the merge-base
    # is found, or there are no more commits in the history.
    # This option takes effect only when changes are detected
    # using git against base branch (feature branch workflow).
    # Default: 100
    initial-fetch-depth: ''

    # Enables listing of files matching the filter:
    #   'none'  - Disables listing of matching files (default).
    #   'csv'   - Coma separated list of filenames.
    #             If needed, it uses double quotes to wrap filename with unsafe characters.
    #   'json'  - File paths are formatted as JSON array.
    #   'shell' - Space delimited list usable as command-line argument list in Linux shell.
    #             If needed, it uses single or double quotes to wrap filename with unsafe characters.
    #   'escape'- Space delimited list usable as command-line argument list in Linux shell.
    #             Backslash escapes every potentially unsafe character.
    # Default: none
    list-files: ''

    # Relative path under $GITHUB_WORKSPACE where the repository was checked out.
    working-directory: ''

    # Personal access token used to fetch a list of changed files
    # from GitHub REST API.
    # It's only used if action is triggered by a pull request event.
    # GitHub token from workflow context is used as default value.
    # If an empty string is provided, the action falls back to detect
    # changes using git commands.
    # Default: ${{ github.token }}
    token: ''
```

## Notes

- Paths expressions are evaluated using [picomatch](https://github.com/micromatch/picomatch) library.
  Documentation for path expression format can be found on the project GitHub page.
- Picomatch [dot](https://github.com/micromatch/picomatch#options) option is set to true.
  Globbing will also match paths where file or folder name starts with a dot.
- It's recommended to quote your path expressions with `'` or `"`. Otherwise, you will get an error if it starts with `*`.
- Local execution with [act](https://github.com/nektos/act) works only with alternative runner image. Default runner doesn't have `git` binary.
  - Use: `act -P ubuntu-latest=nektos/act-environments-ubuntu:18.04`

## TODOs
- Readme
  - [✔] Update the Inputs section with the correct action inputs
  - [✔] Update the Outputs section with the correct action outputs
  - [✔] Update the Usage Example section with the correct usage   
- package.json
  - [✔] Update the `name` with the new action value
- src/main.js
  - [✔] Implement your custom javascript action
- action.yml
  - [✔] Fill in the correct name, description, inputs and outputs
- .prettierrc.json
  - [✔] Update any preferences you might have
- CODEOWNERS
  - [✔] Update as appropriate
- Repository Settings
  - [✔] On the *Options* tab check the box to *Automatically delete head branches*
  - [ ] On the *Options* tab update the repository's visibility (must be done by an org owner)
  - [✔] On the *Branches* tab add a branch protection rule
    - [✔] Check *Require pull request reviews before merging*
    - [✔] Check *Dismiss stale pull request approvals when new commits are pushed*
    - [✔] Check *Require review from Code Owners*
    - [✔] Check *Include Administrators*
  - [✔] On the *Manage Access* tab add the appropriate groups
- About Section (accessed on the main page of the repo, click the gear icon to edit)
  - [✔] The repo should have a short description of what it is for
  - [✔] Add one of the following topic tags:
    | Topic Tag       | Usage                                    |
    | --------------- | ---------------------------------------- |
    | az              | For actions related to Azure             |
    | code            | For actions related to building code     |
    | certs           | For actions related to certificates      |
    | db              | For actions related to databases         |
    | git             | For actions related to Git               |
    | iis             | For actions related to IIS               |
    | microsoft-teams | For actions related to Microsoft Teams   |
    | svc             | For actions related to Windows Services  |
    | jira            | For actions related to Jira              |
    | meta            | For actions related to running workflows |
    | pagerduty       | For actions related to PagerDuty         |
    | test            | For actions related to testing           |
    | tf              | For actions related to Terraform         |
  - [✔] Add any additional topics for an action if they apply    
  - [✔] The Packages and Environments boxes can be unchecked
    

## Recompiling

If changes are made to the action's code in this repository, or its dependencies, you will need to re-compile the action.

```sh
# Installs dependencies, tests and packs the code
npm run all
```

## Code of Conduct

This project has adopted the [im-open's Code of Conduct](https://github.com/im-open/.github/blob/master/CODE_OF_CONDUCT.md).

## License

Copyright &copy; 2021, Extend Health, LLC. Code released under the [MIT license](LICENSE).
