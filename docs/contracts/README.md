# Shared contracts

Contracts that more than one repository implements. Each service's own API and
events are specified in that service's repository (`src/api/openapi.yaml`,
`docs/asyncapi.yaml`); only contracts between components live here.

| Contract | Producer | Consumers |
|---|---|---|
| [Tenant context](tenant-context.md) ([schema](tenant-context.schema.json)) | gateway | every service |

Repositories keep byte-identical copies in `test/contracts/` for their tests.
CI compares them with this repository's `main` (reusable CI input
`contract-repositories`), so a change is merged here first and then copied.
The gateway also checks the identity schemas it copies against
`service-identity`'s `main` the same way.
