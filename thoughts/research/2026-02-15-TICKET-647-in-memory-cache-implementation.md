---
date: "2026-02-15T20:51:58Z"
researcher: claude
git_commit: 3013d474af2f23030d4b7ce60f0e0aa738bebde2
branch: reserach_cach
repository: policy-controller
topic: "In-memory cache implementation to reduce external requests"
tags: [research, codebase, caching, performance, webhook, cosign, registry]
status: complete
last_updated: "2026-02-15"
last_updated_by: claude
---

# Research: In-Memory Cache Implementation for Policy-Controller

**Date**: 2026-02-15T20:51:58Z
**Researcher**: claude
**Git Commit**: 3013d474af2f23030d4b7ce60f0e0aa738bebde2
**Branch**: reserach_cach
**Repository**: policy-controller

## Research Question

How can we implement in-memory caching in the policy-controller to reduce external requests (registry, Rekor, Fulcio)? What prior art exists in the sigstore ecosystem and knative framework? What Go cache libraries are suitable?

## Summary

The policy-controller **already has a complete caching interface** (`ResultCache`) at `pkg/webhook/cache.go` but only a no-op placeholder (`NoCache`) implementation. The interface is designed to cache per-policy validation results keyed by image reference + policy UID + policy ResourceVersion. No real cache has ever been implemented despite the issue being open since March 2023. Knative does **not** provide usable in-memory caching utilities. Several Go cache libraries are viable, with `hashicorp/golang-lru` already being an indirect dependency. There are multiple implementation approaches ranging from minimal (implement the existing interface) to comprehensive (multi-layer caching).

## Detailed Findings

### 1. Existing Cache Infrastructure

#### Cache Interface (`pkg/webhook/cache.go:47-56`)

```go
type ResultCache interface {
    Set(ctx context.Context, image, name, uid, resourceVersion string, cacheResult *CacheResult)
    Get(ctx context.Context, image, uid, resourceVersion string) *CacheResult
}
```

The `CacheResult` wraps both policy results and errors:
```go
type CacheResult struct {
    PolicyResult *PolicyResult
    Errors       []error
}
```

#### Context-Based Injection (`pkg/webhook/cache.go:35-45`)

The cache is injected via Go context using `ToContext`/`FromContext`. If no cache is set, `FromContext` returns a `NoCache` instance.

**Key finding**: `webhook.ToContext` is **never called anywhere** in the codebase. The cache always falls back to `NoCache`.

#### No-Op Implementation (`pkg/webhook/nocache.go`)

`NoCache.Get()` always returns `nil` (cache miss), `NoCache.Set()` is a no-op.

#### Cache Usage Points

- **Set**: `pkg/webhook/validator.go:422-425` - After `ValidatePolicy()` completes, the result is cached
- **Get**: `pkg/webhook/validator.go:481-484` - At the start of `ValidatePolicy()`, cache is checked before doing any work

#### Cache Key Design

The cache key is composed of:
- `image` (string) - image reference
- `uid` (string) - ClusterImagePolicy UID
- `resourceVersion` (string) - ClusterImagePolicy ResourceVersion

This means cache invalidation happens automatically when a policy is modified (new ResourceVersion).

**Note**: There is a subtle inconsistency - `Set` uses `ref.Name()` while `Get` uses `ref.String()`. This would need to be harmonized in any real implementation.

### 2. What Gets Called Externally (What the Cache Would Avoid)

The full validation flow triggers these external calls:

#### High-Impact Registry Calls (per image per policy):
1. **`cosign.VerifyImageSignatures()`** (`pkg/webhook/validation.go:75`) - Fetches and verifies signatures from registry
2. **`cosign.VerifyImageAttestations()`** (`pkg/webhook/validation.go:88`) - Fetches and verifies attestations from registry
3. **`ociremote.ResolveDigest()`** (`pkg/webhook/validator.go:1075`) - Resolves image tags to digests
4. **`ociremote.Referrers()`** (`pkg/webhook/validation.go:93-98`) - OCI 1.1 referrers API
5. **`remote.Get()`** (`pkg/webhook/validator.go:1258`) - Fetches image manifests/config files

#### Medium-Impact Sigstore Infrastructure Calls:
6. **Rekor transparency log queries** - During signature/attestation verification when Rekor is configured
7. **`fulcioroots.Get()`** (`pkg/webhook/validator.go:1517`) - Fetches Fulcio root certificates from TUF
8. **`cosign.GetRekorPubs()`** (`pkg/webhook/validator.go:1589`) - Fetches Rekor public keys from TUF
9. **`cosign.GetCTLogPubs()`** (`pkg/webhook/validator.go:1524`) - Fetches CT Log keys from TUF

#### Already Partially Cached:
10. **TUF trusted root** (`pkg/tuf/repo.go:307-340`) - Has basic in-memory caching with a resync period (default 24h)

### 3. Related GitHub Issues and PRs

| # | Title | State | Relevance |
|---|-------|-------|-----------|
| [#647](https://github.com/sigstore/policy-controller/issues/647) | Implement a simple in-mem cache (with opt-in) | Open | **Primary issue** - describes the exact feature needed |
| [#490](https://github.com/sigstore/policy-controller/issues/490) | Implement a TUF cache for TrustRoot | Open | Related - TUF metadata caching in TrustRoot Status |
| [#1887](https://github.com/sigstore/policy-controller/issues/1887) | Support for Caching Validation Results | Open | Duplicate of #647, confirms no implementation exists |
| [#1820](https://github.com/sigstore/policy-controller/issues/1820) | Memory Leak on EKS Clusters | Open | **Important context** - unbounded ECR credential caching causes OOM |
| [#1825](https://github.com/sigstore/policy-controller/pull/1825) | Fix memory leak with LRU cache for AWS credentials | Open | Introduces `hashicorp/golang-lru/v2` as a direct dependency |

#### Key Insights from Issues

- **Issue #647**: The maintainer (@vaikas) designed the interface and wants opt-in caching via flags. The cache should be injected into context around `cmd/webhook/main.go:76`.
- **Issue #1820**: Real-world memory leak from unbounded ECR credential caching. The pprof analysis showed the cache growing indefinitely. This emphasizes the need for bounded caches with eviction.
- **PR #1825**: Uses `hashicorp/golang-lru/v2` with TTL to fix the ECR credential leak. This establishes a pattern already in the project.
- **Issue #490**: References a Google Design Doc about TUF caching that is not publicly accessible.

### 4. Sigstore Ecosystem Caching

#### Cosign
- Has TUF client caching (local cache at `$HOME/.sigstore`)
- Has digest resolution caching (PR #4612, v3.0.4)
- Has signing config caching (PR #4456, v3.0.3)
- Does **not** provide hooks for injecting external caches into verification

#### Sigstore-go
- Has TUF-based trusted root management with caching
- No explicit in-memory verification result caching

#### Kyverno (comparable project)
- **Implements TTL-based cache** for image verification outcomes
- Default TTL: 3600 seconds (1 hour), configurable via `--cacheTTLDurationSeconds`
- Max entries via `--cacheMaxSize` (default: 1000)
- Caches **successful** verification results only
- Does NOT cache failures (considered transient)
- Rationale: "When cache is enabled, an image once verified by a policy will be considered to be verified until TTL duration expires or there is a change in policy"

#### OPA Gatekeeper
- Caches responses from external data providers
- Default TTL: 3 minutes, configurable
- Can be disabled by setting TTL to 0

### 5. Knative Caching

**Knative does NOT provide in-memory caching utilities.**

- `knative/caching` repository contains only API definitions for caching CRDs, not implementations
- `knative/pkg` has no dedicated cache package
- The `hashicorp/golang-lru` library is already vendored in the knative ecosystem

### 6. Go Cache Library Options

#### Libraries Already in the Dependency Tree
| Library | Version | Status | Notes |
|---------|---------|--------|-------|
| `hashicorp/golang-lru` | v1.0.2 | indirect | Current dependency |
| `hashicorp/golang-lru/v2` | - | In PR #1825 | Being introduced for ECR credential cache |
| `jellydator/ttlcache/v3` | v3.3.0 | indirect | Already vendored |
| `golang/groupcache` | v0.0.0 | indirect | Already vendored |

#### Library Comparison

**hashicorp/golang-lru/v2 (Recommended)**
- Pros: Already being added to the project (PR #1825), simple API, reliable, thread-safe, `expirable` sub-package provides TTL, well-maintained by HashiCorp
- Cons: Basic features compared to Ristretto
- Fit: Excellent - simple, proven, matches existing project patterns

**jellydator/ttlcache/v3**
- Pros: Already in go.mod (indirect), generics support, TTL-focused, callbacks on expiration
- Cons: Less commonly used than golang-lru
- Fit: Good - already a dependency, designed for TTL-based caching

**dgraph-io/ristretto**
- Pros: High performance, TinyLFU admission policy, fully concurrent
- Cons: Set operations can fail under contention, complex configuration, new dependency
- Fit: Overkill for this use case

**maypok86/otter/v2**
- Pros: Modern, high performance, adaptive W-TinyLFU, better than Ristretto on diverse workloads
- Cons: Less battle-tested, new dependency
- Fit: Good alternative if performance is critical

**sync.Map (stdlib)**
- Pros: No external dependency, lock-free reads
- Cons: No TTL, no eviction, unbounded growth
- Fit: Not recommended (see issue #1820 - unbounded caching causes OOM)

## Implementation Options

### Option A: Minimal - Implement the Existing Interface (Recommended Starting Point)

Implement the existing `ResultCache` interface using `hashicorp/golang-lru/v2/expirable` and inject it via `ToContext` in `cmd/webhook/main.go`.

**What gets cached**: Final `PolicyResult` per (image + policy UID + policy ResourceVersion)

**Scope**: Small, ~100-150 lines of new code + flag wiring

**New flags**:
```
--enable-cache (bool, default: false)
--cache-size (int, default: 1024)
--cache-ttl (duration, default: 1h)
```

**Injection point** (`cmd/webhook/main.go`, in the context enrichment functions):
```go
func(ctx context.Context) context.Context {
    ctx = context.WithValue(ctx, kubeclient.Key{}, kc)
    ctx = store.ToContext(ctx)
    ctx = policyControllerConfigStore.ToContext(ctx)
    ctx = cwebhook.ToContext(ctx, cache) // <-- inject here
    // ... rest of validators
}
```

**Pros**:
- Smallest change, lowest risk
- Uses the existing architecture exactly as designed
- Caches the most expensive operation (full policy validation including all registry/Rekor calls)
- Easy to review and merge upstream

**Cons**:
- Doesn't cache across policy changes (new ResourceVersion = cache miss)
- Cache key inconsistency between Set (ref.Name()) and Get (ref.String()) needs fixing first

### Option B: TTL Cache with Separate Success/Failure TTLs

Same as Option A but with separate TTLs for successful and failed validations.

**Additional flags**:
```
--cache-ttl-success (duration, default: 1h)
--cache-ttl-failure (duration, default: 5m)
```

**Pros**:
- More control over error caching behavior
- Follows Kyverno's pattern (they don't cache failures at all)

**Cons**:
- Slightly more complex
- `hashicorp/golang-lru/v2/expirable` doesn't support per-item TTL out of the box (would need `jellydator/ttlcache/v3` or custom wrapper)

### Option C: Image-Level Cache (Policy-Version-Independent)

Cache verification results keyed only by image digest (not policy version), so that policy updates don't invalidate the cache for unchanged images.

**Cache key**: `image_digest + authority_config_hash`

**Pros**:
- More cache hits (policy metadata changes don't invalidate)
- Better for environments with frequent policy updates

**Cons**:
- Larger change to the existing interface
- Need to hash authority configuration to detect meaningful policy changes
- More complex invalidation logic

### Option D: Multi-Layer Cache

Layer 1: Image digest resolution cache (tag -> digest)
Layer 2: Signature/attestation verification result cache (per authority)
Layer 3: Policy result cache (per CIP, as in Option A)

**Pros**:
- Most comprehensive, maximizes cache hits
- Digest resolution cache helps the mutating webhook too
- Individual layers can be tuned independently

**Cons**:
- Most complex implementation
- Requires changes deeper in the validation flow
- Harder to review/merge upstream

### Option E: Use ConfigMap/CRD Status for Persistent Cache

Store cached results in Kubernetes (e.g., in TrustRoot Status or a dedicated ConfigMap) for persistence across pod restarts.

**Pros**:
- Survives pod restarts
- Shared across replicas

**Cons**:
- Significant latency for K8s API calls
- Complex implementation
- etcd storage concerns
- Not mentioned in any existing issue

## Recommendation

**Start with Option A** (implement the existing interface). It's the smallest change, uses the architecture the maintainers designed, and addresses the core issue. The existing `ResultCache` interface and the `Set`/`Get` call sites in `validator.go` are already wired up - the only missing piece is an actual cache implementation and injecting it into the context.

Use `hashicorp/golang-lru/v2/expirable` since PR #1825 is already introducing this dependency. This keeps the project consistent.

**Fix the ref.Name() vs ref.String() inconsistency** in the Set/Get calls before implementing the cache, or the cache will never hit.

After Option A is proven, consider evolving to Option B (separate TTLs) or Option D (multi-layer) based on observed cache hit rates.

## Code References

- `pkg/webhook/cache.go:47-56` - ResultCache interface definition
- `pkg/webhook/nocache.go:23-31` - NoCache placeholder implementation
- `pkg/webhook/validator.go:422-425` - Cache Set call site
- `pkg/webhook/validator.go:481-484` - Cache Get call site
- `pkg/webhook/validation.go:73-89` - cosign.VerifyImageSignatures/Attestations wrappers
- `pkg/webhook/validator.go:1517-1524` - Fulcio root certificate fetching
- `pkg/webhook/validator.go:1589-1635` - Rekor public key fetching
- `pkg/webhook/validator.go:1258-1330` - Image config file fetching
- `pkg/tuf/repo.go:307-340` - Existing TUF trusted root caching pattern
- `cmd/webhook/main.go:252-261` - Context enrichment (injection point for cache)
- `pkg/webhook/registryauth/registryauth.go:45` - ECR credential helper (memory leak source)

## Architecture Documentation

### Current Validation Flow (No Caching)

```
Admission Request
  -> NewValidatingAdmissionController (cmd/webhook/main.go:223)
    -> context enrichment (line 252-261)
    -> validator.ValidatePod/ValidatePodSpecable/etc.
      -> validatePodSpec (pkg/webhook/validator.go:259)
        -> validateContainerImage (per container, parallel)
          -> validatePolicies (if matching CIPs found)
            -> ValidatePolicy (per CIP, parallel)
              -> FromContext(ctx).Get() -> NoCache -> nil (always miss)
              -> ValidatePolicySignaturesForAuthority / ValidatePolicyAttestationsForAuthority
                -> cosign.VerifyImageSignatures / cosign.VerifyImageAttestations
                  -> Registry HTTP calls (manifests, signatures, attestations)
                  -> Rekor HTTP calls (transparency log entries)
              -> FromContext(ctx).Set() -> NoCache -> no-op
```

### Proposed Flow (With Caching)

```
Admission Request
  -> context enrichment (inject real cache via ToContext)
    -> ValidatePolicy
      -> FromContext(ctx).Get() -> LRU cache lookup
        -> HIT: return cached PolicyResult (skip all external calls)
        -> MISS: proceed with verification
          -> cosign.VerifyImage* -> registry/Rekor calls
          -> FromContext(ctx).Set() -> store in LRU cache with TTL
```

## External References

- [Issue #647 - Implement a simple in-mem cache](https://github.com/sigstore/policy-controller/issues/647)
- [Issue #490 - TUF cache for TrustRoot](https://github.com/sigstore/policy-controller/issues/490)
- [Issue #1887 - Support for Caching Validation Results](https://github.com/sigstore/policy-controller/issues/1887)
- [Issue #1820 - Memory Leak on EKS Clusters](https://github.com/sigstore/policy-controller/issues/1820)
- [PR #1825 - LRU cache for AWS credentials](https://github.com/sigstore/policy-controller/pull/1825)
- [Kyverno Issue #9580 - Cache Image Signatures](https://github.com/kyverno/kyverno/issues/9580)
- [Kyverno Issue #9503 - Cache Failed Verification](https://github.com/kyverno/kyverno/issues/9503)
- [OPA Gatekeeper External Data Caching](https://open-policy-agent.github.io/gatekeeper/website/docs/externaldata/)
- [hashicorp/golang-lru/v2](https://pkg.go.dev/github.com/hashicorp/golang-lru/v2)
- [jellydator/ttlcache/v3](https://github.com/jellydator/ttlcache)
- [maypok86/otter](https://github.com/maypok86/otter)
- [Kubernetes Admission Webhook Best Practices](https://kubernetes.io/docs/concepts/cluster-administration/admission-webhooks-good-practices/)

## Open Questions

1. **Should errors be cached?** Kyverno deliberately does NOT cache failures. The existing `CacheResult` type supports caching errors, but transient errors (network issues, auth failures) should not be cached. A separate TTL for errors (Option B) or skipping error caching entirely would be reasonable.

2. **ref.Name() vs ref.String() inconsistency** - The Set call uses `ref.Name()` but Get uses `ref.String()`. Which is correct? This must be harmonized before any cache implementation will work.

3. **Should the mutating webhook also use the cache?** Currently only the validating webhook context enrichment adds validators. The mutating webhook does tag-to-digest resolution which could also benefit from caching.

4. **Cache scope per request vs global** - Should the cache be per-admission-request or global across requests? The existing interface is global (stored in context at controller startup), which is the right approach for reducing registry calls across different admission requests for the same image.

5. **Metrics** - Should cache hit/miss metrics be exposed via Prometheus? This would help operators tune cache size and TTL. Kyverno and Gatekeeper both expose cache metrics.

6. **Helm chart integration** - The new flags (`--enable-cache`, `--cache-size`, `--cache-ttl`) need to be exposed in the Helm chart at `https://github.com/sigstore/helm-charts`.
