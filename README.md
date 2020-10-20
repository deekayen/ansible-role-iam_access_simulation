AWS IAM Access Simulation
=========================

![Super-Linter](https://github.com/deekayen/ansible-role-iam_access_simulation/workflows/Super-Linter/badge.svg) [![Build Status](https://travis-ci.org/deekayen/ansible-role-iam_access_simulation.svg?branch=main)](https://travis-ci.org/deekayen/ansible-role-iam_access_simulation)

Simulate the access of IAM users and roles performing various IAM actions against any ARN. This tells you which users and roles have access to your S3 buckets or KMS keys when the auditors ask.

This will gather all the users and roles in your AWS account, then test the provided actions against ARNs that you will define in the `resources_to_test` variable. If you'd like to test only users OR roles, then you can omit the one you don't need using the `user` or `role` tag assigned to the tasks for filtering.

Test access of IAM users and roles using the [AWS CLI simulate principal policy](https://docs.aws.amazon.com/cli/latest/reference/iam/simulate-principal-policy.html) feature.

Role Variables
--------------

```yaml
resources_to_test: []
```

Example Playbook
----------------

```yaml
---

- hosts: localhost
  connection: local
  gather_facts: no

  vars:
    resources_to_test:
      - action: s3:GetObject
        resource: arn:aws:s3:::deekayen-123456789000-secret-bucket
      - action: kms:Decrypt
        resource: arn:aws:kms:us-east-1:123456789000:key/1234abab-1e2c-3a4b-9ba8-1234567890ab

  roles:
    - deekayen.iam_access_simulation
```

The results of the simulation are printed to the console at the end of the playbook run.

```shell
TASK [iam_access_simulation : Print similation results.] **********************
ok: [localhost] => {
    "msg": [
        "User deekayen allowed to s3:GetObject on arn:aws:s3:::deekayen-123456789000-secret-bucket",
        "Role ec2-instances allowed to s3:GetObject on arn:aws:s3:::deekayen-123456789000-secret-bucket",
        "User deekayen allowed to kms:Decrypt on arn:aws:kms:us-east-1:123456789000:key/1234abab-1e2c-3a4b-9ba8-1234567890ab",
        "Role ec2-instances allowed to kms:Decrypt on arn:aws:kms:us-east-1:123456789000:key/1234abab-1e2c-3a4b-9ba8-1234567890ab"
    ]
}
[

PLAY RECAP *********************************************************************
localhost                  : ok=327  changed=0    unreachable=0    failed=0    skipped=324  rescued=0    ignored=0   

Playbook run took 0 days, 0 hours, 3 minutes, 14 seconds
```

Requirements
------------

The control machine needs boto and AWS CLI.

Dependencies
------------

```yaml
collections:
  - community.aws
  - community.general
```

License
-------

BSD-3-Clause
