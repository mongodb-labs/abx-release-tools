Artifact signing
================

Signing credentials
-------------------

The ABX key is ``ai-builder-experience``.

To find the key ID from the public key URL, run:

.. code-block:: shell

   curl -O https://pgp.mongodb.com/ai-builder-experience.asc
   gpg --import ai-builder-experience.asc

The import lists the key ID. Store the username, password, and key ID
in the AWS secret for GitHub Actions (:doc:`setup`), or as hidden
project variables for Evergreen.

GitHub Actions: ``gpg-sign``
----------------------------

Signs release artifacts with the Garasign GPG container and moves the
detached ``.sig`` files to ``RELEASE_ASSETS``. Run ``setup`` first.

The ``filenames`` input contract, shared by every action that takes a
file list: entries may be **space- or newline-separated**, and each may
be a glob. A pattern that matches nothing fails the step; a file
matched by two patterns is handled once.

Usage
-----

.. code-block:: yaml

   - uses: mongodb-labs/abx-release-tools/actions/gpg-sign@<sha>
     with:
       filenames: dist/*

Inputs
------

.. list-table::
   :header-rows: 1

   * - Input
     - Required
     - Description
   * - ``filenames``
     - yes
     - File names to sign. Can be a glob.
   * - ``ecr_registry``, ``ecr_repository``
     - no
     - Garasign container location. Defaults to the DevProd registry.

Pass a glob, never a package's file names. A rename or version bump
then never means editing the workflow.

GitHub Actions: ``git-sign``
----------------------------

Runs arbitrary git commands in a container with the Garasign GPG keys
loaded. Use it to sign release tags and commits. Run ``git commit -S``
to sign a commit.

Usage
-----

.. code-block:: yaml

   - uses: mongodb-labs/abx-release-tools/actions/git-sign@<sha>
     with:
       command: git tag -s -m "Release 1.0.0" v1.0.0

Inputs
------

.. list-table::
   :header-rows: 1

   * - Input
     - Required
     - Description
   * - ``command``
     - yes
     - Git command run inside the container.
   * - ``ecr_registry``, ``ecr_repository``
     - no
     - Garasign container location. Defaults to the DevProd registry.

Release identity
----------------

The ``setup`` action sets the git identity to
``mongodb-dbx-release-bot[bot]``. The bypass list in each ruleset must
include the identity that pushes commits and tags. See
:doc:`publishing`.

Evergreen: ``evergreen/sign_artifacts.yml``
-------------------------------------------

The signing shell lives in ``evergreen/ecr-login.sh`` and
``evergreen/sign-artifact.sh``. A project clones this repo at run time
and calls the scripts from the clone; see :ref:`evergreen-fetch`. The
function template in ``evergreen/sign_artifacts.yml`` is copied into
the project config, because ``ec2.assume_role`` is an Evergreen
built-in with no shell equivalent.

Hidden project variables needed:

- ``GARASIGN_USERNAME``: Garasign username (``ai-builder-experience``)
- ``GARASIGN_PASSWORD``: Garasign password
- ``GPG_KEY_ID``: key to sign with

``sign-artifact.sh`` passes ``GPG_KEY_ID`` to ``gpg --local-user``. A
Garasign account holding more than one key otherwise signs with
whichever key gpg picks, and the signature then verifies against a key
the release notes do not name.

Task usage:

.. code-block:: yaml

   tasks:
     - name: sign_artifacts
       commands:
         - func: "fetch abx-release-tools"
         - func: build  # produces dist/
         - func: sign_artifact
           vars:
             unsigned_artifact: dist/abx-1.0.0-py3-none-any.whl
             detached_signature: dist/abx-1.0.0-py3-none-any.whl.sig

Credentials ride the environment, never the script text: Evergreen
prints script text in task logs.

Source: `mongo-rust-driver
<https://github.com/mongodb/mongo-rust-driver/blob/main/.evergreen/release-sign.sh>`_.

Verification
------------

Users check a signature against MongoDB's public key:

.. code-block:: shell

   gpg --verify artifact.whl.sig artifact.whl

The public key lives at
https://pgp.mongodb.com/ai-builder-experience.asc. Publish the location
in the project README. Attach each ``.sig`` file to the GitHub Release
next to the artifact, as pymongo does.
