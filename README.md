# CIP build

Converts `Test-Block-Unblock.xml` into a binary `.cip` using `ConvertFrom-CIPolicy`,
which ships only on Windows Pro / Enterprise / Education / Server.

The GitHub Actions Windows runner has it; Windows Home does not.

**No secrets are used or needed.** The XML contains only policy rules and the
*public* fingerprint of the signing certificate. Signing happens locally,
never in CI. Do not add the .pfx to this repo.

Download the `unsigned-cip` artifact from the workflow run, then sign and
deploy it locally.
