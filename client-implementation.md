# Client Implementations

This document describes client implementation recommendations for interoperability with registry servers and other clients using the OCI Distribution Spec.

## Backwards Compatibility

Client implementations SHOULD support registries that implement partial or older versions of the OCI Distribution Spec.
This section describes how client fallback procedures when an API is not available.

### Referrers API

The Referrers API here is described by [Listing Referrers](spec.md#listing-referrers) and [end-12](spec.md#endpoints).

A client that pushes an Image or Artifact manifest with a defined `Refers` field MUST verify the Referrers API is available.
A client querying the Referrers API and receiving a 404 MUST fallback to using an Index pushed to a tag described by the following schema.

#### Referrers Tag Schema

```text
<alg>-<ref>
```

- `<alg>`: the digest algorithm (e.g. `sha256` or `sha512`)
- `<ref>`: the digest from the `refers` field (limit of 64 characters)

For example, a manifest with the `Refers` field digest set to `sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa` in the `registry.example.org/project` repository would have a expect an Index at `registry.example.org/project:sha256-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa`

#### Pushing Manifests with Refers

When pushing an Image or Artifact manifest with the `Refers` field and the Referrers API returns a 404, the client MUST:

1. Pull the Index using the tag schema.
1. If the tag returns a manifest other than the expected Index, the client SHOULD report a failure and skip the remaining steps.
1. If the tag returns a 404, the client MUST begin with an empty Index.
1. Verify the descriptor for the manifest is not already in the Index (duplicate entries SHOULD NOT be created).
1. Append a descriptor for the pushed manifest to the Index manifests.
   The value of the `artifactType` MUST be set in the descriptor to value of the `artifactType` in the Artifact manifest, or the Config descriptor `mediaType` in the Image manifest.
   All annotations from the Image or Artifact manifest MUST be copied to this descriptor.
1. Push the updated Index using the same tag schema.
   The client MAY use conditional HTTP requests to prevent overwriting an Index that has changed since it was first pulled.

#### Listing Referrers

If the Referrers API returns a 404, the client MUST fallback to pulling the tag schema.
The response SHOULD be an Index with the same content that would be expected from the Referrers API.
If the response to the Referrers API is a 404, and the tag schema does not return a valid Index, the client SHOULD assume there are no Referrers to the manifest.

#### Deleting Referrers

When deleting an Image or Artifact manifest that contains a `Refers` field, and the Referrers API returns a 404, clients SHOULD:

1. Pull the Index using the tag schema.
1. Delete the descriptor from the manifest list to the deleted manifest.
1. Push the updated Index using the same tag schema.
   The client MAY use conditional HTTP requests to prevent overwriting an Index that has changed since it was first pulled.

#### Deleting Manifests with Referrers

Clients MAY delete a tag using the tag schema when it returns a valid Index manifest and the referred manifest has been deleted.

#### Referrers API Recommendations

- Clients MAY verify the registry does not support the referrers API by querying the API and checking for a 404.
- When the Referrers API is not available, clients MAY perform periodic garbage collection of stale tag schema tags and descriptors in the Index manifest list that no longer exist.
- Clients MAY use a conditional HTTP push for registries that support ETag conditions to avoid conflicts with other clients.
- For portability, clients MAY generate artifacts using image-spec until all registries where the artifact is pushed to have been upgraded.
- Tooling that copies images between registries MAY recursively query for referrers and copy them.
