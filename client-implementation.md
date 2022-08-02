# Client Implementations

The distribution-spec is mainly concerned with registry behaviour. However,
some registries may not always support all aspects of the spec.

This document is a guide for clients that wish to add backwards compaitibility
for registries which only use older endpoints.

For the remainder of this document "referrers API" will refer to [end-12](TODO),
and the use of reference types.

## Digest Tags

For registries that do not support the referrers API, a tag MUST be pushed containing an index of all manifests that refer to the specified manifest, matching the API response above.
The tag uses the following schema:

```text
<alg>-<ref>
```

- E.g. `registry.example.org/project:sha256-0000000000000000000000000000000000000000000000000000000000000000`
- `<alg>`: the digest algorithm
- `<ref>`: the digest from the `refers` field (limit of 64 characters)
- Periodic garbage collection may be performed by clients pushing new referrers, deleting stale referrers that have been replaced with newer versions, and tags that no longer point to an accessible manifest.
- Clients can verify the registry does not support the referrers API by querying the API and checking for a 404.
- Clients should pull the existing tag and extend it with additional entries, minimizing the time between the pull and push to reduce the chance of overwriting changes from other clients.
- Clients should use a conditional push for registries that support ETag conditions to avoid overwriting a tag that has been modified by another client since the previous tag manifest was pulled.

## Client Expectations

### Creating Artifacts

- For portability, clients should generate artifacts using image-spec until all registries where the artifact is pushed to have been upgraded.
- The descriptor of the manifest this artifact refers to should be set in the `refers` field of the manifest.
- Other metadata should be included in the manifest annotations.
- When pushing, the registry should be checked for the referrers API support.
- If the referrers API is not available, the artifact should be pushed with a tag using the digest schema. Tags with the digest schema should not be pushed if the referrers API is available.

### Pulling Artifacts

To pull artifacts that reference an existing manifest:

- Clients requests artifacts associated with a manifest first query the referrers API using the digest of the requested manifest.
- If the request fails, clients must fall back to listing tags in the repository and pull manifests for each tag with the matching `<alg>` and `<ref>` prefix.
- Clients then pull any artifact matching their criteria from the API response or the list generated from the tag query.

### Copying Images

- Tooling that copies images between registries may recursively query for referrers and copy them.
- Copying an artifact that refers to an image uses the same pull and push steps described above and should support both existing registries and registries with the referrers API on both the pull and push.

##### Deleting Manifests

For managing existing registries without the referrers API:

- Client tooling that deletes manifests should also delete digest tags that reference that manifest.
- Client tooling may periodically check for dangling digest tags that refer to missing manifest, and prune those tags.
