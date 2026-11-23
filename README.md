# PrivateAI Persona attestation values

This repository publishes the attestation values that clients (for example the browser extension) use to check
that they are talking to a genuine PrivateAI Persona server running in an AWS confidential (Nitro) environment on
Amazon ECS.

It holds one file, [`pcrs.json`](pcrs.json):

```
https://raw.githubusercontent.com/tempest-tech-ltd/private-ai-persona-pcrs/main/pcrs.json
```

## What the values mean

Each server answers `GET /tee/attestation` with an attestation document signed by AWS. Clients check two values
from it against this file:

- **PCR17**: the hash of the server's attestation policy, which pins the container image digest and the task
  configuration. It changes with every release and differs per environment.
- **PCR8**: the hash of the certificate Amazon ECS signs its sandbox image with. It does not depend on our code, and
  ECS rotates it from time to time. AWS does not publish these values, so we publish the ones we observe.

PCR17 alone is not enough: code in any Nitro enclave can reproduce it. PCR8 shows that an Amazon ECS sandbox
recorded it.

## Format

```json
{
  "releases": [
    {
      "environment": "prod | qa",
      "version": "app version",
      "commit": "git commit SHA the image was built from",
      "image_digest": "sha256:<container image digest>",
      "pcr17": "<96 hex chars>",
      "released_at": "ISO 8601 UTC time the release started serving"
    }
  ],
  "ecs_pcr8": [
    { "pcr8": "<96 hex chars>", "first_seen": "YYYY-MM-DD" }
  ]
}
```

Both lists keep every entry, newest first. Entries are never edited or removed.

## Client rules

- Verify the AWS signature, the certificate chain up to the AWS Nitro Enclaves root, freshness and your nonce
  before looking at any PCR.
- Accept a server only if its **PCR17 matches one of the latest two `releases` entries for the environment the
  client talks to**. A production client must never accept a `qa` entry.
- Accept a server only if its **PCR8 matches one of the latest three `ecs_pcr8` entries**.
- This file is served with a short cache (about 5 minutes on raw.githubusercontent.com). Releases are published
  before they are deployed.

## How changes are made

- `main` is protected: every change is a pull request with an approving review, force-pushes and branch deletion
  are blocked, and nobody can bypass these rules. The git history is the public record of every value we have
  published.
- **New release (PCR17):** the deploy process opens a pull request with the new entry, and it is merged before or
  with the deployment.
- **New ECS certificate (PCR8):** a daily check, which also runs after every deployment, reads PCR8 from the running
  servers and opens a pull request when it sees a value that is not published. A person merges it only after
  confirming it comes from a genuine ECS rotation. It is never merged automatically: publishing every value a
  check sees would also publish a value from someone else's enclave.
