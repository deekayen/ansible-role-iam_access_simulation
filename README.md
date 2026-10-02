# deekayen.iam_access_simulation

[![CI](https://github.com/deekayen/ansible-role-iam_access_simulation/actions/workflows/ci.yml/badge.svg)](https://github.com/deekayen/ansible-role-iam_access_simulation/actions/workflows/ci.yml) [![Ansible Galaxy](https://img.shields.io/badge/galaxy-deekayen.iam__access__simulation-blue.svg)](https://galaxy.ansible.com/ui/standalone/roles/deekayen/iam_access_simulation/) ![BSD 3-Clause license](https://img.shields.io/badge/license-BSD%203--Clause-blue)

An Ansible role that lists every IAM user and role in an AWS account and asks the IAM policy simulator whether each one is allowed to perform a given action on a given resource. It prints the principals whose identity-based policies allow the action.

The role lists principals with `amazon.aws.iam_user_info` and `amazon.aws.iam_role_info`. For each entry in `resources_to_test`, it runs [`aws iam simulate-principal-policy`](https://docs.aws.amazon.com/cli/latest/reference/iam/simulate-principal-policy.html) once per user and once per role, passing the principal ARN, the resource ARN, and the action. Results with an `EvalDecision` of `allowed` are collected into one list, printed by the last task. Nothing in the account is changed.

## Requirements

- ansible-core 2.15 or newer on the controller.
- The `amazon.aws` and `community.general` collections.
- The `jmespath` Python library on the controller, for the `community.general.json_query` filter.
- On the host the play targets, normally `localhost`: the AWS CLI, plus `boto3` and `botocore`. amazon.aws 11.4.0 lists boto3 1.28.0 and botocore 1.31.0 as minimums for the IAM info modules.
- AWS credentials on that host, found through the usual AWS credential chain, with read access to IAM users and roles and permission for `iam:SimulatePrincipalPolicy`.

## Supported platforms

`meta/main.yml` declares GenericLinux, all versions. CI lints the role and runs `ansible-playbook --syntax-check`; it does not run a simulation against an AWS account.

## Installation

From Ansible Galaxy:

```bash
ansible-galaxy role install deekayen.iam_access_simulation
ansible-galaxy collection install amazon.aws community.general
```

Or pin it in `requirements.yml`:

```yaml
---
roles:
  - name: deekayen.iam_access_simulation
    src: https://github.com/deekayen/ansible-role-iam_access_simulation.git
    scm: git
    version: main

collections:
  - name: amazon.aws
  - name: community.general
```

```bash
ansible-galaxy install -r requirements.yml
```

## Role variables

| Variable | Default | Description |
| --- | --- | --- |
| `resources_to_test` | `[]` | List of dictionaries, each with a `resource` ARN and an IAM `action` such as `s3:GetObject`. The role asserts that every `resource` starts with `arn:` and every `action` has the form `service:Action`. With an empty list the role lists principals and prints an empty result. |

`vars/main.yml` initializes `iam_access_simulation_output`, the internal list the results are collected into.

## Behavior

- The role passes no `--resource-policy`, so the simulation evaluates identity-based policies only. As of October 2026, the [simulate-principal-policy reference](https://docs.aws.amazon.com/cli/latest/reference/iam/simulate-principal-policy.html) says a resource policy has to be passed as a string to be included, and that simulating resource-based policies is not supported for IAM roles. Grants made only in an S3 bucket policy or a KMS key policy do not show up in the results.
- For users, the simulation also includes policies attached to the user's groups, per the same reference.
- Only `allowed` results are printed. Principals that are implicitly or explicitly denied are left out.
- The role makes one `simulate-principal-policy` call per principal per entry in `resources_to_test`, so run time grows with the number of users and roles in the account.

## Dependencies

None. The collections are requirements, not role dependencies.

## Example playbook

```yaml
---
- name: Report which IAM principals' policies allow reading the audit bucket and key.
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    resources_to_test:
      - action: s3:GetObject
        resource: arn:aws:s3:::example-audit-bucket/*
      - action: kms:Decrypt
        resource: arn:aws:kms:us-west-2:123456789012:key/00000000-0000-0000-0000-000000000000

  roles:
    - deekayen.iam_access_simulation
```

The bucket name, account ID `123456789012`, and key ID are placeholders.

The last task prints the results as one string. With placeholder principals, the output looks like this:

```text
TASK [deekayen.iam_access_simulation : Print simulation results.] *************
ok: [localhost] => {
    "msg": "['User example-user allowed to s3:GetObject on arn:aws:s3:::example-audit-bucket/*', 'Role example-role allowed to kms:Decrypt on arn:aws:kms:us-west-2:123456789012:key/00000000-0000-0000-0000-000000000000']"
}
```

## Tags

| Tag | Tasks |
| --- | --- |
| `user` | Listing IAM users and simulating their access. |
| `role` | Listing IAM roles and simulating their access. |

Run with `--tags user` or `--tags role` to simulate only one kind of principal. Input validation, the per-resource loop, and the final results task are tagged `always`.

## Known issues

- `tasks/main.yml:43` passes the result list through `trim`, which turns it into a single string of the Python list representation instead of a list. Confirmed on ansible-core 2.21.4.
- The role-side "Print simulation outcome" task in `tasks/simulate.yml:53` has no `verbosity: 2`, unlike the user-side task at line 20, so it prints every raw role simulation result at default verbosity.

## Development

CI runs on every push to `main` and every pull request (see `.github/workflows/ci.yml`). It installs the collections from `tests/requirements.yml`, runs `ansible-lint --profile production`, and syntax-checks `tests/test.yml`. To run the same checks locally:

```bash
pip3 install ansible-lint
ansible-galaxy install -r tests/requirements.yml
ansible-lint --profile production
mkdir -p .ansible/roles && ln -sfn "$PWD" .ansible/roles/deekayen.iam_access_simulation
ANSIBLE_ROLES_PATH=.ansible/roles:~/.ansible/roles ansible-playbook --syntax-check tests/test.yml -i tests/inventory
```

The repository also has a `.pre-commit-config.yaml`; run `pre-commit run --all-files` before pushing.

### Repository layout

| Path | Purpose |
| --- | --- |
| `tasks/main.yml` | Lists users and roles, loops over `resources_to_test`, and prints the results. |
| `tasks/assert.yml` | Validates each `resources_to_test` entry, tagged `always`. |
| `tasks/simulate.yml` | Runs the simulator for every user and role against one resource. |
| `tasks/set_user_facts.yml`, `tasks/set_role_facts.yml` | Append `allowed` results to the output list. |
| `defaults/main.yml` | The one user-facing variable. |
| `vars/main.yml` | The internal output list. |
| `meta/argument_specs.yml` | Argument spec for `resources_to_test`. |
| `tests/` | Syntax-check playbook, inventory, and collection requirements used by CI. |
| `.github/workflows/` | `ci.yml` for lint and syntax check, `release.yml` for Galaxy import. |

## Releases

Pushing a git tag runs `.github/workflows/release.yml`, which imports the tagged commit into Ansible Galaxy as `deekayen.iam_access_simulation`. The import needs a `GALAXY_API_KEY` repository or organization secret.

## License

BSD 3-Clause. See [LICENSE](LICENSE).

## Author

[David Norman](https://github.com/deekayen). Sponsorship links are in [.github/FUNDING.yml](.github/FUNDING.yml).
