# Upgrade Procedures

This document describes how registries should add support for new APIs and features.

## Referrers API

The Referrers API here is described by [Listing Referrers](spec.md#listing-referrers) and [end-12a](spec.md#endpoints).
When registries add support for the Referrers API, this API needs to account for manifests that were pushed before the API was available using the [Referrers Tag Schema](spec.md#referrers-tag-schema).

1. Registries MUST include preexisting Image and Artifact manifests that are listed in an Index tagged with the tag schema and have a valid Refers field.
1. Registries MAY include all existing Image and Artifact manifests with a Refers field in the API response.
1. Registries MUST include all new Image and Artifact manifests with a valid Refers field in the API response.
