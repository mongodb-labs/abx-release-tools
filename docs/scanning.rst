Dependency scanning
===================

Two ways to run SilkBomb on GitHub Actions. Choose by what the repo
needs:

- Scan-only CI, no AWS release infrastructure: run the container
  inline, with ``KONDUKTO_TOKEN`` from a GitHub Actions secret. Source:
  `mongodb-mcp-server
  <https://github.com/mongodb-js/mongodb-mcp-server/pull/1356>`_.
- A repo that also releases: use :doc:`setup` and the ``sbom`` action.
  The repo needs ``setup`` anyway for signing and S3 upload; the
  Kondukto token then lives in the AWS secret.

Onboard the repository to Kondukto
----------------------------------

File an RNDSEC task, one per repository. Project: RNDSEC. Issue type:
Task. Summary: ``Onboard GitHub Repository to Kondukto - <org>/<repo>``.
Assigned team: R&D Security. This creates the Kondukto project that
SilkBomb uploads to.

Repositories in the drivers org use a DevProd ticket instead, like
`DEVPROD-15304 <https://jira.mongodb.org/browse/DEVPROD-15304>`_.
Kondukto questions go to ``#ask-product-security``.

GitHub Actions: run SilkBomb inline (scan-only)
-----------------------------------------------

For a repo with no AWS release infrastructure, run the container
directly. Grant the workflow ``id-token: write`` and configure an AWS
OIDC role with read access to the DevProd Platforms ECR (secret
``DEVPROD_PLATFORMS_ECR_ROLE_ARN``). Store the Kondukto CI API token
as ``KONDUKTO_TOKEN`` in a GitHub Actions secret.

.. code-block:: shell

   ECR_HOST=901841024863.dkr.ecr.us-east-1.amazonaws.com
   aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_HOST"
   docker run --rm -e KONDUKTO_TOKEN -v "$PWD":/pwd \
     "$ECR_HOST/release-infrastructure/silkbomb:2.0" \
     upload --sbom-in /pwd/sbom.json \
     --repo "$GITHUB_REPOSITORY" --branch "$GITHUB_REF_NAME"

GitHub Actions: ``sbom``
------------------------

The ``sbom`` action generates an SBOM with SilkBomb, then augments it.
It writes ``cyclonedx.sbom.json`` to both ``RELEASE_ASSETS`` and
``S3_ASSETS``. Run ``setup`` first.

Usage
-----

.. code-block:: yaml

   - uses: mongodb-labs/abx-release-tools/actions/sbom@<sha>
     with:
       kondukto_sub_project: main

Inputs
------

.. list-table::
   :header-rows: 1

   * - Input
     - Required
     - Description
   * - ``sbom_in_path``
     - no
     - Input SBOM. Default ``sbom.json``.
   * - ``sbom_file_name``
     - no
     - Output SBOM name. Default ``cyclonedx.sbom.json``.
   * - ``kondukto_sub_project``
     - no
     - Kondukto sub-project, appended to the branch name.
   * - ``ecr_registry``, ``ecr_repository``
     - no
     - SilkBomb container location. Defaults to the DevProd registry.

The ``sbom`` action reads the input SBOM; it does not make one. Run
``python/generate-sbom`` first, or an ecosystem-native tool of your own.

The augment step needs the Kondukto token, so the AWS secret must
contain ``KONDUKTO_TOKEN``. See :doc:`setup`.

GitHub Actions: ``python/generate-sbom``
----------------------------------------

Builds the input SBOM for a Python project with ``cyclonedx-py``. It
needs no ``setup``: nothing here touches AWS.

.. code-block:: yaml

   - uses: mongodb-labs/abx-release-tools/actions/python/generate-sbom@<sha>
   - uses: mongodb-labs/abx-release-tools/actions/sbom@<sha>

Name the project type. ``cyclonedx-py`` reads one type per subcommand,
and the default, ``requirements``, reads ``requirements.txt`` from the
working directory. A project that declares its dependencies in
``pyproject.toml`` has no such file, and the run then fails.

Point ``source_path`` at the pinned release requirements. Dev
dependencies do not ship, so they do not belong in the SBOM.

Inputs
------

.. list-table::
   :header-rows: 1

   * - Input
     - Required
     - Description
   * - ``source``
     - no
     - ``cyclonedx-py`` subcommand naming the project type:
       ``requirements``, ``poetry``, ``pipenv`` or ``environment``.
       Default ``requirements``.
   * - ``source_path``
     - no
     - Positional argument for that subcommand: a requirements file, a
       project directory for ``poetry`` and ``pipenv``, an interpreter
       for ``environment``. Each subcommand has its own default, so
       this may be left empty.
   * - ``sbom_out``
     - no
     - File to write. Default ``sbom.json``, which is what ``sbom``
       reads.
   * - ``cyclonedx_version``
     - no
     - Version of ``cyclonedx-py`` to install. Empty installs whatever
       pip resolves. An installed ``cyclonedx-py`` is left alone.

Evergreen: ``evergreen/silkbomb.yml``
-------------------------------------

The shell lives in ``scripts/generate-sbom.sh``,
``evergreen/ecr-login.sh`` and ``evergreen/upload-sbom.sh``. A project
clones this repo at run time and calls the scripts from the clone; see
:ref:`evergreen-fetch`. The function and task templates in
``evergreen/silkbomb.yml`` are copied into the project config, and the
``<TODO>`` values adapted.

``generate-sbom.sh`` sits outside ``evergreen/`` because the
``python/generate-sbom`` action runs it too. One copy of the shell, so
a fix reaches both CI systems.

Project variables needed:

- ``silkbomb_role_arn``: ``arn:aws:iam::119629040606:role/silkbomb``
- ``devprod_platforms_ecr_readonly_role_arn``:
  ``arn:aws:iam::901841024863:role/ecr-role-evergreen-ro``
- ``branch_name``: the branch being scanned
- ``sbom_source``, ``sbom_source_arg``: optional, the project type and
  its path. Default ``requirements``, which reads
  ``requirements.txt``.

The template defines one task, ``UPLOAD_SBOM_TO_KONDUKTO``, in an
``ssdlc`` buildvariant. It needs no Kondukto token: the silkbomb IAM
role credentials are enough for ``upload``. The augmented SBOM that
ships with a release is produced on GitHub Actions by the ``sbom``
action.

Source: `mongo-csharp-driver
<https://github.com/mongodb/mongo-csharp-driver/blob/main/evergreen/upload-sbom.sh>`_.
