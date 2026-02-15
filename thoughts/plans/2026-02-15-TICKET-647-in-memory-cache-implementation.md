# TICKET-647: In-Memory Cache Implementation Plan

## Overview

Implement the existing `ResultCache` interface (`pkg/webhook/cache.go`) with an LRU+TTL cache using `hashicorp/golang-lru/v2/expirable`. Inject it into the validating webhook context via opt-in CLI flags. This closes issue #647 and #1887.

## Current State Analysis

- The `ResultCache` interface exists at `pkg/webhook/cache.go:47-56` with `Set`/`Get` methods
- Only a `NoCache` no-op implementation exists at `pkg/webhook/nocache.go`
- `Set` is called at `pkg/webhook/validator.go:422` and `Get` at `pkg/webhook/validator.go:481`
- `ToContext` is **never called** anywhere - the cache always falls back to `NoCache`
- `hashicorp/golang-lru/v2 v2.0.7` is already in `go.sum` as an indirect dependency
- No tests exist for the cache interface or its usage in the validation flow

### Key Discoveries:
- Bug: `ref.Name()` used in `Set` vs `ref.String()` in `Get` causes cache to never hit (`validator.go:422`). Already fixed in working tree.
- Kyverno uses identical pattern: TTL cache, 1000 entries, 1h TTL, no error caching
- E2E tests are bash scripts only; no Go-level integration tests for the webhook flow
- Existing `ValidatePolicy` tests use swappable function variables (`cosignVerifySignatures`, `cosignVerifyAttestations`) making cache integration tests straightforward

## Desired End State

- A working LRU+TTL cache implementation of `ResultCache`
- Cache is opt-in via `--enable-cache` flag (default: false)
- Cache size configurable via `--cache-size` (default: 1024)
- Cache TTL configurable via `--cache-ttl` (default: 1h)
- Errors are never cached (only successful validations)
- Cache is injected into the validating webhook context only
- Unit tests for cache implementation + integration tests for cache usage in ValidatePolicy
- The `ref.Name()` -> `ref.String()` bug fix is included

### Verification:
```bash
# All tests pass
go test ./pkg/webhook/... -v -run "TestLRUCache|TestValidatePolicyCache"

# Full test suite passes
go test $(go list ./... | grep -v third_party/)
```

## What We're NOT Doing

- Mutating webhook caching (separate concern, different flow)
- Prometheus metrics for cache hits/misses (no custom metrics exist today - follow-up)
- Helm chart flag exposure (separate repo: sigstore/helm-charts)
- Multi-layer caching (digest resolution, signature verification layers)
- Persistent/shared cache across replicas (K8s API-backed)
- Caching errors (transient by nature, confuses users signing images)

## Implementation Approach

**TDD**: Write tests first, verify they fail, then implement. Two test files:
1. `pkg/webhook/lrucache_test.go` - Unit tests for the cache implementation
2. Cache integration tests added to `pkg/webhook/validator_test.go` - Tests that ValidatePolicy uses the cache correctly

## Phase 1: Write Tests (TDD - Red)

### Overview
Write all tests first. They will fail because the `LRUCache` type doesn't exist yet.

### Changes Required:

#### 1. Unit Tests for LRU Cache
**File**: `pkg/webhook/lrucache_test.go` (new)

Tests to write:
- `TestLRUCacheSetGet` - Basic set and get, verify cache hit returns correct result
- `TestLRUCacheMiss` - Get on empty cache returns nil
- `TestLRUCacheSkipsErrors` - Set with non-empty errors is a no-op, subsequent Get returns nil
- `TestLRUCacheTTLExpiry` - Set with short TTL, sleep, verify Get returns nil
- `TestLRUCacheEviction` - Set more entries than cache size, verify oldest are evicted
- `TestLRUCacheKeyIsolation` - Different image/uid/resourceVersion combinations don't collide
- `TestLRUCacheResourceVersionInvalidation` - Same image+uid but different resourceVersion = cache miss (policy updated)

```go
package webhook

import (
	"context"
	"errors"
	"testing"
	"time"
)

func TestLRUCacheSetGet(t *testing.T) {
	cache := NewLRUCache(10, 1*time.Hour)
	ctx := context.Background()

	want := &CacheResult{
		PolicyResult: &PolicyResult{
			AuthorityMatches: map[string]AuthorityMatch{},
		},
	}
	cache.Set(ctx, "gcr.io/foo/bar@sha256:abc", "my-policy", "uid-1", "v1", want)

	got := cache.Get(ctx, "gcr.io/foo/bar@sha256:abc", "uid-1", "v1")
	if got == nil {
		t.Fatal("expected cache hit, got nil")
	}
	if got.PolicyResult == nil {
		t.Fatal("expected PolicyResult, got nil")
	}
}

func TestLRUCacheMiss(t *testing.T) {
	cache := NewLRUCache(10, 1*time.Hour)
	ctx := context.Background()

	got := cache.Get(ctx, "gcr.io/foo/bar@sha256:abc", "uid-1", "v1")
	if got != nil {
		t.Fatalf("expected cache miss (nil), got %v", got)
	}
}

func TestLRUCacheSkipsErrors(t *testing.T) {
	cache := NewLRUCache(10, 1*time.Hour)
	ctx := context.Background()

	cache.Set(ctx, "gcr.io/foo/bar@sha256:abc", "my-policy", "uid-1", "v1", &CacheResult{
		Errors: []error{errors.New("image not signed")},
	})

	got := cache.Get(ctx, "gcr.io/foo/bar@sha256:abc", "uid-1", "v1")
	if got != nil {
		t.Fatalf("expected cache miss for error result, got %v", got)
	}
}

func TestLRUCacheTTLExpiry(t *testing.T) {
	cache := NewLRUCache(10, 50*time.Millisecond)
	ctx := context.Background()

	cache.Set(ctx, "gcr.io/foo/bar@sha256:abc", "my-policy", "uid-1", "v1", &CacheResult{
		PolicyResult: &PolicyResult{
			AuthorityMatches: map[string]AuthorityMatch{},
		},
	})

	// Should hit immediately
	if got := cache.Get(ctx, "gcr.io/foo/bar@sha256:abc", "uid-1", "v1"); got == nil {
		t.Fatal("expected cache hit before TTL expiry")
	}

	// Wait for TTL to expire
	time.Sleep(100 * time.Millisecond)

	if got := cache.Get(ctx, "gcr.io/foo/bar@sha256:abc", "uid-1", "v1"); got != nil {
		t.Fatalf("expected cache miss after TTL expiry, got %v", got)
	}
}

func TestLRUCacheEviction(t *testing.T) {
	cache := NewLRUCache(2, 1*time.Hour)
	ctx := context.Background()
	result := &CacheResult{
		PolicyResult: &PolicyResult{
			AuthorityMatches: map[string]AuthorityMatch{},
		},
	}

	cache.Set(ctx, "image-1", "p", "uid-1", "v1", result)
	cache.Set(ctx, "image-2", "p", "uid-1", "v1", result)
	cache.Set(ctx, "image-3", "p", "uid-1", "v1", result) // evicts image-1

	if got := cache.Get(ctx, "image-1", "uid-1", "v1"); got != nil {
		t.Fatal("expected image-1 to be evicted")
	}
	if got := cache.Get(ctx, "image-2", "uid-1", "v1"); got == nil {
		t.Fatal("expected image-2 to still be cached")
	}
	if got := cache.Get(ctx, "image-3", "uid-1", "v1"); got == nil {
		t.Fatal("expected image-3 to still be cached")
	}
}

func TestLRUCacheKeyIsolation(t *testing.T) {
	cache := NewLRUCache(10, 1*time.Hour)
	ctx := context.Background()
	result := &CacheResult{
		PolicyResult: &PolicyResult{
			AuthorityMatches: map[string]AuthorityMatch{},
		},
	}

	cache.Set(ctx, "image-a", "p", "uid-1", "v1", result)

	// Different image
	if got := cache.Get(ctx, "image-b", "uid-1", "v1"); got != nil {
		t.Fatal("expected miss for different image")
	}
	// Different UID
	if got := cache.Get(ctx, "image-a", "uid-2", "v1"); got != nil {
		t.Fatal("expected miss for different UID")
	}
	// Correct key
	if got := cache.Get(ctx, "image-a", "uid-1", "v1"); got == nil {
		t.Fatal("expected hit for matching key")
	}
}

func TestLRUCacheResourceVersionInvalidation(t *testing.T) {
	cache := NewLRUCache(10, 1*time.Hour)
	ctx := context.Background()
	result := &CacheResult{
		PolicyResult: &PolicyResult{
			AuthorityMatches: map[string]AuthorityMatch{},
		},
	}

	cache.Set(ctx, "image-a", "my-policy", "uid-1", "v1", result)

	// Same image+uid but new resourceVersion (policy was updated)
	if got := cache.Get(ctx, "image-a", "uid-1", "v2"); got != nil {
		t.Fatal("expected miss for updated resourceVersion")
	}
	// Original version still hits
	if got := cache.Get(ctx, "image-a", "uid-1", "v1"); got == nil {
		t.Fatal("expected hit for original resourceVersion")
	}
}
```

#### 2. Cache Integration Tests for ValidatePolicy
**File**: `pkg/webhook/validator_test.go` (append)

Tests to write:
- `TestValidatePolicyCacheHit` - Second call to ValidatePolicy returns cached result without calling cosign
- `TestValidatePolicyCacheSkipsErrors` - Failed validation is NOT cached, second call invokes cosign again

```go
func TestValidatePolicyCacheHit(t *testing.T) {
	// Save and restore global mock functions
	origCVS := cosignVerifySignatures
	defer func() { cosignVerifySignatures = origCVS }()

	callCount := 0
	cosignVerifySignatures = func(_ context.Context, _ name.Reference, _ *cosign.CheckOpts) ([]oci.Signature, bool, error) {
		callCount++
		sig, err := static.NewSignature(nil, "")
		if err != nil {
			return nil, false, err
		}
		return []oci.Signature{sig}, true, nil
	}

	ctx, _ := rtesting.SetupFakeContext(t)
	kc, err := k8schain.NewNoClient(ctx)
	if err != nil {
		t.Fatal(err)
	}

	// Inject a real cache into context
	cache := NewLRUCache(10, 1*time.Hour)
	ctx = ToContext(ctx, cache)

	cip := webhookcip.ClusterImagePolicy{
		Authorities: []webhookcip.Authority{{
			Key: &webhookcip.KeyRef{
				Data:              authorityKeyCosignPubString,
				PublicKeys:        []crypto.PublicKey{authorityKeyCosignPub},
				HashAlgorithm:     signaturealgo.DefaultSignatureAlgorithm,
				HashAlgorithmCode: crypto.SHA256,
			},
		}},
	}
	cip.UID = "test-uid"
	cip.ResourceVersion = "v1"

	// First call - should invoke cosign
	result1, errs1 := ValidatePolicy(ctx, system.Namespace(), digest, cip, kc)
	if len(errs1) > 0 {
		t.Fatalf("unexpected errors: %v", errs1)
	}
	if result1 == nil {
		t.Fatal("expected non-nil PolicyResult")
	}
	if callCount != 1 {
		t.Fatalf("expected cosign to be called once, got %d", callCount)
	}

	// Second call - should return cached result without calling cosign
	result2, errs2 := ValidatePolicy(ctx, system.Namespace(), digest, cip, kc)
	if len(errs2) > 0 {
		t.Fatalf("unexpected errors: %v", errs2)
	}
	if result2 == nil {
		t.Fatal("expected non-nil cached PolicyResult")
	}
	if callCount != 1 {
		t.Fatalf("expected cosign NOT to be called again (cache hit), got %d calls", callCount)
	}
}

func TestValidatePolicyCacheSkipsErrors(t *testing.T) {
	origCVS := cosignVerifySignatures
	defer func() { cosignVerifySignatures = origCVS }()

	callCount := 0
	cosignVerifySignatures = func(_ context.Context, _ name.Reference, _ *cosign.CheckOpts) ([]oci.Signature, bool, error) {
		callCount++
		return nil, false, errors.New("image not signed")
	}

	ctx, _ := rtesting.SetupFakeContext(t)
	kc, err := k8schain.NewNoClient(ctx)
	if err != nil {
		t.Fatal(err)
	}

	cache := NewLRUCache(10, 1*time.Hour)
	ctx = ToContext(ctx, cache)

	cip := webhookcip.ClusterImagePolicy{
		Authorities: []webhookcip.Authority{{
			Key: &webhookcip.KeyRef{
				Data:              authorityKeyCosignPubString,
				PublicKeys:        []crypto.PublicKey{authorityKeyCosignPub},
				HashAlgorithm:     signaturealgo.DefaultSignatureAlgorithm,
				HashAlgorithmCode: crypto.SHA256,
			},
		}},
	}
	cip.UID = "test-uid"
	cip.ResourceVersion = "v1"

	// First call - should fail and NOT cache the error
	_, errs1 := ValidatePolicy(ctx, system.Namespace(), digest, cip, kc)
	if len(errs1) == 0 {
		t.Fatal("expected errors on first call")
	}
	if callCount != 1 {
		t.Fatalf("expected cosign to be called once, got %d", callCount)
	}

	// Second call - should call cosign again because errors aren't cached
	_, errs2 := ValidatePolicy(ctx, system.Namespace(), digest, cip, kc)
	if len(errs2) == 0 {
		t.Fatal("expected errors on second call")
	}
	if callCount != 2 {
		t.Fatalf("expected cosign to be called again (errors not cached), got %d calls", callCount)
	}
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests compile: `go build ./pkg/webhook/...`
- [ ] Tests exist and **fail** (the `NewLRUCache` function doesn't exist yet): `go test ./pkg/webhook/... -run "TestLRUCache|TestValidatePolicyCache" 2>&1 | grep FAIL`

**Implementation Note**: After this phase, verify all tests fail with compilation errors (`NewLRUCache` undefined). This confirms the tests are correctly written and will validate the implementation.

---

## Phase 2: Implement LRU Cache (TDD - Green)

### Overview
Implement `NewLRUCache` to make all tests pass.

### Changes Required:

#### 1. LRU Cache Implementation
**File**: `pkg/webhook/lrucache.go` (new)

```go
package webhook

import (
	"context"
	"fmt"
	"time"

	expirable "github.com/hashicorp/golang-lru/v2/expirable"
)

// LRUCache implements ResultCache using an LRU cache with TTL expiration.
// Errors are not cached - only successful validation results are stored.
type LRUCache struct {
	cache *expirable.LRU[string, *CacheResult]
}

// NewLRUCache creates a new LRU cache with the given size and TTL.
func NewLRUCache(size int, ttl time.Duration) *LRUCache {
	return &LRUCache{
		cache: expirable.NewLRU[string, *CacheResult](size, nil, ttl),
	}
}

func cacheKeyFor(image, uid, resourceVersion string) string {
	return fmt.Sprintf("%s/%s/%s", image, uid, resourceVersion)
}

func (c *LRUCache) Get(ctx context.Context, image, uid, resourceVersion string) *CacheResult {
	result, ok := c.cache.Get(cacheKeyFor(image, uid, resourceVersion))
	if !ok {
		return nil
	}
	return result
}

func (c *LRUCache) Set(ctx context.Context, image, name, uid, resourceVersion string, cacheResult *CacheResult) {
	// Do not cache errors - they are transient and caching them risks
	// confusing users who are actively signing images.
	if len(cacheResult.Errors) > 0 {
		return
	}
	c.cache.Add(cacheKeyFor(image, uid, resourceVersion), cacheResult)
}
```

#### 2. Bug Fix
**File**: `pkg/webhook/validator.go:422`
**Change**: `ref.Name()` -> `ref.String()` (already in working tree)

#### 3. Add Direct Dependency
**Command**: `go get github.com/hashicorp/golang-lru/v2@v2.0.7`

This promotes the indirect dependency to a direct one in `go.mod`.

### Success Criteria:

#### Automated Verification:
- [ ] All unit tests pass: `go test ./pkg/webhook/... -v -run "TestLRUCache"`
- [ ] All integration tests pass: `go test ./pkg/webhook/... -v -run "TestValidatePolicyCache"`
- [ ] Full test suite passes: `go test $(go list ./... | grep -v third_party/)`
- [ ] Build succeeds: `go build ./...`

**Implementation Note**: After this phase, all tests from Phase 1 should pass. Verify before proceeding.

---

## Phase 3: Wire Flags and Context Injection

### Overview
Add CLI flags to `cmd/webhook/main.go` and inject the cache into the validating webhook context.

### Changes Required:

#### 1. Add CLI Flags and Cache Injection
**File**: `cmd/webhook/main.go`

Add flags in the `var` block (after `trustrootResyncPeriod`):

```go
// Cache configuration for validating webhook results.
// https://github.com/sigstore/policy-controller/issues/647
enableCache = flag.Bool("enable-cache", false, "Enable in-memory LRU cache for validation results.")
cacheSize   = flag.Int("cache-size", 1024, "Maximum number of entries in the validation result cache.")
cacheTTL    = flag.Duration("cache-ttl", 1*time.Hour, "TTL for cached validation results.")
```

Add cache creation and injection in `NewValidatingAdmissionController`, inside the context enrichment function (after `policyControllerConfigStore.ToContext(ctx)`):

```go
func NewValidatingAdmissionController(ctx context.Context, cmw configmap.Watcher) *controller.Impl {
	// ... existing code ...

	// Create cache if enabled
	var cache cwebhook.ResultCache
	if *enableCache {
		cache = cwebhook.NewLRUCache(*cacheSize, *cacheTTL)
		logging.FromContext(ctx).Infof("Validation result cache enabled: size=%d, ttl=%v", *cacheSize, *cacheTTL)
	}

	// ... existing kc, validator setup ...

	return validation.NewAdmissionController(ctx,
		*webhookName,
		"/validations",
		types,
		func(ctx context.Context) context.Context {
			ctx = context.WithValue(ctx, kubeclient.Key{}, kc)
			ctx = store.ToContext(ctx)
			ctx = policyControllerConfigStore.ToContext(ctx)
			if cache != nil {
				ctx = cwebhook.ToContext(ctx, cache)
			}
			// ... rest of validators ...
			return ctx
		},
		false,
		nil,
	)
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Build succeeds: `go build ./cmd/webhook/...`
- [ ] Full test suite passes: `go test $(go list ./... | grep -v third_party/)`
- [ ] Flag help shows new flags: `go run ./cmd/webhook/ --help 2>&1 | grep -E "enable-cache|cache-size|cache-ttl"`

#### Manual Verification:
- [ ] Deploy to a test cluster with `--enable-cache` flag
- [ ] Verify that repeated admission requests for the same image are faster (check webhook logs for cache-related debug messages or reduced registry call latency)
- [ ] Verify that signing a new image and re-deploying correctly validates (error not cached)
- [ ] Verify that without `--enable-cache`, behavior is identical to before (NoCache fallback)

---

## Testing Strategy

### Unit Tests (`pkg/webhook/lrucache_test.go`):
- Basic Set/Get round-trip
- Cache miss on empty cache
- Error results are not cached
- TTL expiry evicts entries
- LRU eviction when cache is full
- Key isolation (different image/uid/version don't collide)
- ResourceVersion change invalidates cache (policy updated)

### Integration Tests (`pkg/webhook/validator_test.go`):
- `ValidatePolicy` cache hit: second call returns cached result, cosign not invoked again
- `ValidatePolicy` error bypass: failed validation not cached, cosign invoked on retry

### Existing E2E Tests:
- Bash-based scripts in `test/` - should continue to pass unchanged since cache defaults to disabled

## Performance Considerations

- LRU cache is O(1) for Get/Set operations
- Cache is thread-safe (expirable.LRU uses internal sync.Mutex)
- Memory bounded by `--cache-size` (default 1024 entries)
- Each entry is a `*CacheResult` containing `*PolicyResult` (small - just authority match metadata, no image data)
- TTL prevents stale results from persisting indefinitely

## References

- Research: `thoughts/research/2026-02-15-TICKET-647-in-memory-cache-implementation.md`
- Issue #647: https://github.com/sigstore/policy-controller/issues/647
- Issue #1887: https://github.com/sigstore/policy-controller/issues/1887
- Cache interface: `pkg/webhook/cache.go:47-56`
- NoCache: `pkg/webhook/nocache.go`
- Cache Set call site: `pkg/webhook/validator.go:422`
- Cache Get call site: `pkg/webhook/validator.go:481`
- Context injection point: `cmd/webhook/main.go:252-261`
- hashicorp/golang-lru/v2: https://pkg.go.dev/github.com/hashicorp/golang-lru/v2
