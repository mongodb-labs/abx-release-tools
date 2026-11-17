Setup
=====

The ``setup`` action prepares the release environment. Run it first:
every other action except ``python/publish`` reads the environment it
writes.

Usage
-----

.. code-block:: yaml

   - uses: mongodb-labs/abx-release-tools/actions/setup@<sha>
     with:
       aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
       aws_region_name: ${{ vars.AWS_REGION_NAME }}
       aws_secret_id: ${{ secrets.AWS_SECRET_ID }}

Inputs
------

.. list-table::
   :header-rows: 1

   * - Input
     - Required
     - Description
   * - ``aws_role_arn``
     - yes
     - AWS role that can read the release secret.
   * - ``aws_region_name``
     - yes
     - AWS region of the secret.
   * - ``aws_secret_id``
     - yes
     - Name or ARN of the AWS Secrets Manager secret holding the
       release credentials. A comma is rejected: the secrets-manager
       action reads it as an alias separator.
   * - ``ecr_registry_id``
     - no
     - ECR registry id. Default ``901841024863``.
   * - ``ecr_role_arn``
     - no
     - ECR pull role. Default ``ecr-role-gha-ro``.
   * - ``ecr_region``
     - no
     - ECR region. Default ``us-east-1``.

What it writes
--------------

The action logs in to ECR, reads the AWS secret, and exports:

- ``RELEASE_ASSETS``: directory for artifacts that ship with the
  release.
- ``S3_ASSETS``: directory for files uploaded to the release S3
  bucket.
- ``GARASIGN_ENVFILE``: environment file for the Garasign container.
  Holds ``GARASIGN_USERNAME`` and ``GARASIGN_PASSWORD`` from the secret.
- ``KONDUKTO_ENVFILE``: environment file with ``KONDUKTO_TOKEN``.
  Written only if the secret contains the token.
- ``AWS_BUCKET``: release S3 bucket (``RELEASE_ASSETS_BUCKET`` key in
  the secret).
- ``GPG_KEY_ID``, ``GPG_PUBLIC_URL``, ``SECURITY_REPORT_URL``: passed
  through from the secret when present.

The credential values are checked before the env files are written: a
missing ``GARASIGN_USERNAME``, ``GARASIGN_PASSWORD``, or ``GPG_KEY_ID``
fails the step, as does a newline in any of them (docker env files are
line-oriented). The files are written under ``RUNNER_TEMP`` with
``umask 077``.

It also sets the git identity to ``mongodb-dbx-release-bot[bot]`` for
release commits and tags.

AWS setup
---------

Create the private S3 bucket, the secret, and the role before the first
release.

S3 bucket
~~~~~~~~~

Use a private bucket for release assets, for example
``abx-release-assets``. The bucket name goes in the secret as
``release-assets-bucket``. The actions write to it under
``arn:aws:s3:::abx-release-assets``.

IAM policy
~~~~~~~~~~

The role needs a policy that reads the secret and writes the bucket.
Replace the bucket name, region, account ID, and secret ARN with your
values.

.. code-block:: json

   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "s3:PutObject",
           "s3:PutObjectAcl",
           "s3:GetObjectAcl"
         ],
         "Resource": "arn:aws:s3:::abx-release-assets/*"
       },
       {
         "Effect": "Allow",
         "Action": [
           "s3:ListBucket",
           "s3:GetBucketLocation"
         ],
         "Resource": "arn:aws:s3:::abx-release-assets"
       },
       {
         "Effect": "Allow",
         "Action": "secretsmanager:ListSecrets",
         "Resource": "*"
       },
       {
         "Effect": "Allow",
         "Action": [
           "secretsmanager:DescribeSecret",
           "secretsmanager:GetSecretValue"
         ],
         "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:abx-release/team-abcdef"
       }
     ]
   }

Name the policy, for example ``abx-release-assets``.

IAM role trust policy
~~~~~~~~~~~~~~~~~~~~~

The role trusts the GitHub OIDC provider for one environment. Replace
the account ID and repository with your values. Keep the environment
name ``release``.

.. code-block:: json

   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
             "token.actions.githubusercontent.com:sub": "repo:<org>/<repo>:environment:release"
           }
         }
       }
     ]
   }

The environment name in the subject must be ``release``.

.. _evergreen-fetch:

Evergreen: fetch the tools
--------------------------

Evergreen has no cross-repo ``uses:``. It can include YAML from a repo
attached as a module, but no MongoDB driver does that, so this repo
follows the pattern the drivers do run: clone the tools at run time and
call their scripts. That is how
`drivers-evergreen-tools
<https://github.com/mongodb-labs/drivers-evergreen-tools>`_ is
consumed; pymongo clones it and calls
``${DRIVERS_TOOLS}/.evergreen/teardown.sh`` and the rest.

Copy this function from ``evergreen/fetch.yml`` into the project
config, and call it before any other function in ``evergreen/``:

.. code-block:: yaml

   functions:
     "fetch abx-release-tools":
       - command: shell.exec
         params:
           script: |
             set -euo pipefail
             rm -rf abx-release-tools
             git clone https://github.com/mongodb-labs/abx-release-tools.git
             git -C abx-release-tools checkout "${abx_release_tools_ref}"

Set ``abx_release_tools_ref`` to a commit SHA, the same pin the actions
take. A branch name is not a pin.

The shell is shared this way, so a fix here reaches every project on
its next run. The YAML is still copied, because Evergreen built-ins
such as ``ec2.assume_role``, ``expansions.update`` and
``papertrail.trace`` have no shell equivalent.

Most of that shell is under ``evergreen/``. One script,
``scripts/generate-sbom.sh``, sits outside it, because the
``python/generate-sbom`` action runs the same file. The clone still
reaches it: the function calls
``abx-release-tools/scripts/generate-sbom.sh``.
