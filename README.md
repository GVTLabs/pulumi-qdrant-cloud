# pulumi-qdrant-cloud

Pulumi SDKs for [Qdrant Cloud](https://cloud.qdrant.io), generated from
[`qdrant/terraform-provider-qdrant-cloud`](https://github.com/qdrant/terraform-provider-qdrant-cloud)
via Pulumi's Terraform provider bridge.

This repository does exactly one thing: it watches the upstream provider for new
releases, regenerates the SDKs, and tags the result with the same version. Tag
`v1.28.0` here contains the SDKs generated from upstream `v1.28.0`. Everything
under `sdks/` is generated — never edit it by hand.

## Using the SDKs

The supported way to consume the Qdrant Cloud provider is to let Pulumi generate
the SDK into your own project, which is the same command this repository
automates:

```bash
pulumi package add terraform-provider qdrant/qdrant-cloud
```

Pass a version as a trailing argument to pin to a specific upstream release:

```bash
pulumi package add terraform-provider qdrant/qdrant-cloud 1.28.0
```

The SDKs committed here are not published to PyPI, npm, NuGet, or Maven, and the
generated Go module keeps Pulumi's canonical module path
(`github.com/pulumi/pulumi-terraform-provider/sdks/go/qdrant-cloud`) rather than
this repository's, so it cannot be `go get`-ed from here without a `replace`
directive. Treat this repository as a versioned, reviewable record of what the
bridge produces: useful for diffing the generated API surface between provider
releases, auditing what lands in your stack before you upgrade, and vendoring.

## Configuration

The provider takes the same credentials as the Terraform provider — a Qdrant
Cloud management API key and an account ID:

```bash
pulumi config set --secret qdrant-cloud:apiKey <QDRANT_CLOUD_MANAGEMENT_KEY>
pulumi config set qdrant-cloud:accountId <QDRANT_CLOUD_ACCOUNT_ID>
```

See the [upstream provider documentation](https://registry.terraform.io/providers/qdrant/qdrant-cloud/latest/docs)
for the full resource and data source reference. Resource names are the
Terraform names in each language's idiomatic casing, so
`qdrant-cloud_accounts_cluster` becomes `AccountsCluster`.

## How the sync works

[`.github/workflows/sync.yml`](.github/workflows/sync.yml) runs
[`scripts/sync.sh`](scripts/sync.sh) hourly. The script reads the latest upstream
release, exits early if a matching tag already exists, and otherwise regenerates
all five SDKs, commits them, and creates the mirroring tag. Git tags are the only
state, so the job is idempotent and needs no external bookkeeping.

GitHub does not allow one repository to subscribe to another repository's release
events, so polling is the only trigger available to us without cooperation from
the upstream maintainers. The workflow also accepts a `repository_dispatch` event
of type `upstream-release` if you ever want to wire up a relay and skip the
polling delay:

```bash
gh api repos/gvtlabs/pulumi-qdrant-cloud/dispatches \
  -f event_type=upstream-release \
  -F 'client_payload[version]=1.28.0'
```

One wrinkle the script handles: the bridge resolves the provider from
`registry.opentofu.org`, which can lag a GitHub release by minutes or hours. When
the release exists upstream but is not yet resolvable, the script exits `75` and
the workflow reports a neutral notice instead of a failure, leaving the next
scheduled run to pick it up. The script also verifies that the SDK it generated
reports the version that was asked for, so a registry fallback to an older
provider can never be committed under the wrong tag.

## Running it locally

```bash
./scripts/sync.sh                      # sync the latest upstream release
./scripts/sync.sh --version 1.27.0     # backfill a specific version
./scripts/sync.sh --no-commit          # regenerate without committing
./scripts/sync.sh --force              # regenerate even if the tag exists
```

Requires the [Pulumi CLI](https://www.pulumi.com/docs/install/); no language
toolchains are needed, since the SDKs are generated but never built.

## License

Apache 2.0, matching the upstream provider. See [LICENSE](LICENSE).
