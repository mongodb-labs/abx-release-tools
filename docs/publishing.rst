PyPI publishing
===============

GitHub Actions: ``python/publish``
----------------------------------

Publishes dists to PyPI with trusted publishing. No stored token: the
job presents a GitHub OIDC token and PyPI checks the claims against
the registered publisher.

The publish job needs ``id-token: write``. Without the permission the
OIDC token is not issued and the publish step fails:

.. code-block:: yaml

   publish:
     runs-on: ubuntu-latest
     permissions:
       id-token: write
     steps:
       - uses: actions/download-artifact@v4
         with:
           name: dist
           path: dist/
       - uses: mongodb-labs/abx-release-tools/actions/python/publish@<sha>

Inputs
------

.. list-table::
   :header-rows: 1

   * - Input
     - Required
     - Description
   * - ``dist_path``
     - no
     - Directory holding the dists. Default ``dist/``.
   * - ``repository-url``
     - no
     - PyPI upload URL. Default ``https://upload.pypi.org/legacy/``.
   * - ``skip-existing``
     - no
     - Skip files already on the index. Default ``false``.

The action disables PyPI attestations. The Garasign signature from
:doc:`signing` is the only signature on an ABX release. pymongo
additionally attests build provenance in its release pipeline; ABX
deliberately ships one signature.

Pending publisher
-----------------

A publisher can only attach to a PyPI project that already exists.
Register a pending publisher by hand on pypi.org before the first
release; without it, the release fails at the publish step. The form
fields and their gotchas are in the ABX release how-to guides:
https://docs.google.com/document/d/1rhYkWgbasperhG_9m5enMLamVkM0Nm5E7XuZ9lixxLI/edit

Sign the release tag
--------------------

The DBX policy requires signed release tags. Sign them with the
``git-sign`` action; see :doc:`signing`.

Create a tag ruleset that blocks tag deletion and force-push. Add the
release identity to the bypass list. Create the branch ruleset in the
same way. The release identity is in :doc:`signing`.
