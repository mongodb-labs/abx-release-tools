Onboarding a repository
=======================

One-time setup before a repository can use these actions for a release.
The steps follow the DRIVERS `secure release guide
<https://wiki.corp.mongodb.com/spaces/DRIVERS/pages/427464463/How+To+Set+up+Secure+Release+Process+using+GitHub+Actions>`_,
adapted for ABX.

Policy
------

- All commits are signed.
- Branches and tags are protected.
- Release artifacts are signed.

Team setup
----------

Do these steps once for the team.

Garasign credentials
~~~~~~~~~~~~~~~~~~~~

The ABX key exists. The username is ``ai-builder-experience``. The
credentials are in the ABX-Frameworks 1Password vault.

A new team requests its own key with a DevProd ticket, like
`DEVPROD-7556 <https://jira.mongodb.org/browse/DEVPROD-7556>`_. Request
one set of credentials per project, so a leak affects one project.

To find the key ID from the public key URL, run:

.. code-block:: shell

   curl -O https://pgp.mongodb.com/ai-builder-experience.asc
   gpg --import ai-builder-experience.asc

The import lists the key ID. Store it in the secret as ``gpg-key-id``.

Release bucket and secret
~~~~~~~~~~~~~~~~~~~~~~~~~

Ask DevProd Release Infrastructure for these, as part of the release
infrastructure onboarding request (DEVPROD-44346):

- A private S3 bucket for the release assets, for example
  ``abx-release-assets``.
- A Secrets Manager secret named ``abx-release/<team>``.

Do not reuse a name that starts with ``drivers/``. Non-admins can read
secrets under that prefix.

The secret holds these keys:

.. list-table::
   :header-rows: 1

   * - Key
     - Value
   * - ``garasign-username``
     - Garasign username, for example ``ai-builder-experience``.
   * - ``garasign-password``
     - Garasign password.
   * - ``gpg-key-id``
     - Key ID of the signing key.
   * - ``gpg-public-url``
     - Public key URL, for example
       ``https://pgp.mongodb.com/ai-builder-experience.asc``.
   * - ``release-assets-bucket``
     - Name of the bucket above.
   * - ``security-report-url``
     - The central security testing summary:
       https://docs.google.com/document/d/1y2K_RY4GZVXpQvv4JH_35mSzFRTawNJ3mibpvSBU8H0
   * - ``kondukto-token``
     - Team Kondukto token. See :doc:`scanning`.

IAM policy
~~~~~~~~~~

The role that reads the secret and writes the bucket needs a policy
like this. Replace the placeholders. Scope the secret to the team's
own secret.

.. code-block:: json

   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "ListReleaseBucket",
         "Effect": "Allow",
         "Action": [
           "s3:ListBucket",
           "s3:GetBucketLocation"
         ],
         "Resource": "arn:aws:s3:::<bucket>"
       },
       {
         "Sid": "WriteReleaseAssets",
         "Effect": "Allow",
         "Action": [
           "s3:PutObject",
           "s3:PutObjectAcl",
           "s3:GetObjectAcl"
         ],
         "Resource": "arn:aws:s3:::<bucket>/*"
       },
       {
         "Sid": "ReadReleaseSecret",
         "Effect": "Allow",
         "Resource": "<secret-arn>",
         "Action": [
           "secretsmanager:ListSecrets",
           "secretsmanager:DescribeSecret",
           "secretsmanager:GetSecretValue"
         ]
       }
     ]
   }

Repo setup
----------

Do these steps once per repository.

Kondukto project
~~~~~~~~~~~~~~~~

File an RNDSEC task, one per repository. Project: RNDSEC. Issue type:
Task. Summary: ``Onboard GitHub Repository to Kondukto - <org>/<repo>``.
Assigned team: R&D Security. This creates the Kondukto project that
SilkBomb uploads to.

Repositories in the drivers org use a DevProd ticket instead, like
`DEVPROD-15304 <https://jira.mongodb.org/browse/DEVPROD-15304>`_.
Kondukto questions go to ``#ask-product-security``.

Release identity
~~~~~~~~~~~~~~~~

The ``setup`` action sets the git identity to
``mongodb-dbx-release-bot[bot]``. If the release job pushes commits or
tags as that app, give the app access to the repository. Raise a ZenDesk
IT ticket. The request must come from an owner of the repository.

If the job pushes with ``GITHUB_TOKEN`` instead, add the GitHub Actions
app to the bypass list in the rulesets below.

IAM role
~~~~~~~~

An AWS admin creates a role that trusts the GitHub OIDC provider. Add
the team policy above. Use a role name like ``abx-release-role``.

.. code-block:: json

   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::<aws-account-id>:oidc-provider/token.actions.githubusercontent.com"
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

The subject condition limits the role to one repository and one
environment. Change both values for each repository.

Release environment
~~~~~~~~~~~~~~~~~~~

In the repository settings, open Environments and create an environment
named ``release``. The name must match the subject condition above.

- Add the team as required reviewers.
- Add the secrets below. See :doc:`setup` for the action inputs.
- Add the variable ``AWS_REGION_NAME``, for example ``us-east-1``.

.. list-table::
   :header-rows: 1

   * - Name
     - Kind
     - Value
   * - ``AWS_ROLE_ARN``
     - Secret
     - ARN of the role above.
   * - ``AWS_SECRET_ID``
     - Secret
     - Name or ARN of the team secret.
   * - ``AWS_REGION_NAME``
     - Variable
     - Region of the secret, for example ``us-east-1``.

The DRIVERS guide also lists ``APP_ID`` and ``APP_PRIVATE_KEY``. They
belong to the drivers-github-tools GitHub App. These actions do not use
that app, so leave them out.

Rulesets
~~~~~~~~

Under Settings, Rules, Rulesets, create or edit the branch ruleset and
the tag ruleset. Add the release identity to the bypass list in each
ruleset.

Workflows
~~~~~~~~~

Add workflows that call the actions. The release workflow runs
``setup`` first, then the step actions. See
:doc:`index` for the actions and the `example project
<https://github.com/mongodb-labs/abx-release-example>`_ for a complete
set of workflows.

Verification
------------

Run the workflow with ``workflow_dispatch`` and a dry run before the
first release. A real release is still the first test of the Garasign
container, the secret load, and the S3 upload.
