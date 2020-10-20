AWS IAM Access Simulation
=========================

![Super-Linter](https://github.com/deekayen/ansible-role-iam_access_simulation/workflows/Super-Linter/badge.svg) [![Build Status](https://travis-ci.org/deekayen/ansible-role-iam_access_simulation.svg?branch=main)](https://travis-ci.org/deekayen/ansible-role-iam_access_simulation)

Test the access of IAM users and roles performing various IAM actions against any ARN.

Test access of IAM users and roles using the [AWS CLI simulate principal policy](https://docs.aws.amazon.com/cli/latest/reference/iam/simulate-principal-policy.html) feature.

## Role Variables

```
resources_to_test: []
```

## Example Playbook

```
---

- hosts: localhost
  connection: local
  gather_facts: no

  roles:
    - deekayen.iam_access_simulation
```

## Requirements

The control machine needs boto and AWS CLI.

## Dependencies

```
collections:
  - community.aws
  - community.general
```

## License

BSD-3-Clause
