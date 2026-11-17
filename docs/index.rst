abx-release-tools
=================

Shared release tooling for ABX (AI Builder Experience) integrations.

The tooling covers the four secure-release steps every ABX release
needs, plus the setup action that precedes them:

.. toctree::
   :maxdepth: 2

   Setup <setup>
   Dependency scanning with SilkBomb <scanning>
   Artifact signing with Garasign <signing>
   Artifact tracing: Papertrail or the authorized-publication report <tracing>
   Trusted publishing to PyPI <publishing>

Which CI system can run which step
----------------------------------

A cell holding ``✅`` means the step is the same work either way. A cell
naming a command or a mechanism means the two systems do different
things, not two implementations of the same thing.

.. list-table::
   :header-rows: 1
   :widths: 20 24 24 32

   * - Step
     - Evergreen
     - GitHub Actions
     - Notes
   * - Dependency scanning
     - ``silkbomb upload``
     - ``silkbomb update``, then ``augment``
     - Two halves of scanning when a repo runs both systems, not one
       job twice. Evergreen pushes the
       dependency list to Kondukto and produces no release artifact.
       GitHub Actions produces ``cyclonedx.sbom.json``, the SBOM that
       ships with the release. Only the second needs
       ``KONDUKTO_TOKEN``; ``upload`` is satisfied by the silkbomb IAM
       role. Each template does one half because that is how it was
       written, not because the other system cannot.
   * - Artifact signing
     - ✅
     - ✅
     - Same Garasign container, same GPG key, same output.
   * - Artifact tracing
     - ``papertrail.trace``
     - authorized-publication report
     - Both satisfy the requirement. Neither produces what the other
       does: Evergreen records one trace per downloadable file in
       Papertrail, GitHub Actions writes a report to the release S3
       bucket.
   * - Publishing to PyPI
     -
     - ✅
     - Trusted publishing works by PyPI checking a GitHub OIDC token,
       and Evergreen cannot present one. The only way to publish from
       Evergreen is a stored PyPI API token, which is what trusted
       publishing exists to remove.

GitHub Actions can run all four steps. Publishing must run there,
because PyPI trusted publishing needs a GitHub OIDC token. Evergreen
can run the three earlier steps, but it cannot publish.

A repo that already has an Evergreen project can run scanning,
signing, and tracing there. A repo without one uses GitHub Actions
for everything, which is the normal case for ABX.

The input SBOM is the one piece of the pipeline that is shared
outright: both systems build it with the same script. See
:doc:`scanning`.

Provenance and the verification trail live in the repo README. Pin
every action to a commit SHA in real workflows.
