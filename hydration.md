---
description: How hydration works in the Feed Platform - enriching entities with media/text evidences and store metadata after blending
---
# Hydration in the Feed Platform

**Hydration** is the process of enriching entities with additional data (media, text evidences, and store metadata) that happens **after blending**. This separation is intentional—it avoids fetching detailed data for entities that might get filtered out during blending.

## Two Types of Hydration

```
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│                              HYDRATION IN THE FEED PIPELINE                                │
└───────────────────────────────────────────────────────────────────────────────────────────┘

        ┌─────────┐      ┌─────────────────┐      ┌───────┐      ┌─────────────────┐
        │  FETCH  │ ───▶ │ PRE-BLEND       │ ───▶ │ BLEND │ ───▶ │  POST-RANK      │
        │Products │      │ HYDRATION       │      │       │      │  HYDRATION      │
        └─────────┘      │ (Collections)   │      └───────┘      │  (Evidences)    │
                         └─────────────────┘                      └─────────────────┘
                                 │                                        │
                                 ▼                                        ▼
                         ┌───────────────┐                        ┌───────────────────┐
                         │ FetchMembers()│                        │ PostRankHydrate() │
                         │ BlendMembers()│                        │  - Text evidences │
                         │               │                        │  - Media evidences│
                         │ For each      │                        │  - MDH store data │
                         │ collection    │                        │                   │
                         └───────────────┘                        └───────────────────┘
```

### 1️⃣ Pre-Blend Hydration (Collection Members)

Happens in `hydrateUnits()` → calls `FetchMembers()` and `BlendMembers()` for collections

### 2️⃣ Post-Rank Hydration (Entity Evidences)

Happens in `postRankHydrate()` → calls `PostRankHydrate()` on each entity to add media/text evidences

## Detailed Walkthrough: Post-Rank Hydration for a Store Entity

Let's trace what happens when a **Restaurant Store** entity gets hydrated:

### Step 1: Pipeline Calls `postRankHydrate()`

From `feed.go`:

```go
// postRankHydrate performs post-rank hydration on all feed units.
func postRankHydrate(ctx context.Context, feedScope scope.FeedScope, blendedFeed []scope.FeedUnit) fut.Future[map[int]bool] {
    hydrationFutures := make([]fut.Future[bool], len(blendedFeed))
    for i, unit := range blendedFeed {
        // Each unit (entity, collection, placement) gets hydrated
        hydrationFutures[i] = unit.PostRankHydrate(ctx, feedScope)
    }
    // Scatter all hydration calls in parallel
    scatterFuture := futlib.Scatter(ctx, hydrationFutures, "FeedUnit.PostRankHydrate")
    return futlib.GatherKeyed(ctx, scatterFuture, futlib.AtLeastOne())
}
```

### Step 2: Store Entity's `PostRankHydrate()` Method

From `core_store.go`:

```go
func (s *CoreStore) PostRankHydrateFromEntity(
    ctx context.Context,
    parentScope scope.ParentForEntityScope,
    entityFromProduct entity.Entity,
) fut.Future[bool] {
    
    // 🔵 All three fetches start IN PARALLEL
    mediaFuture := media.FetchAll(ctx, parentScope, entityFromProduct)  // Media evidences
    textFuture := text.FetchAll(ctx, parentScope, entityFromProduct)    // Text evidences
    mdhFuture := mdhdatacache.GetStores(ctx, parentScope, ...)          // MDH store data
    
    // Wait for all three, then apply results
    return fut.MapValue3(ctx, mediaFuture, textFuture, mdhFuture,
        func(ctx context.Context, 
            mediaEvidences []types.MediaEvidence, 
            textEvidenceColls []types.TextEvidenceCollection, 
            mdhStores map[types.StoreId]*mdhpb.Store,
        ) (bool, error) {
            // Attach evidences to the store
            s.mediaEvidences = mediaEvidences
            s.textEvidenceCollections = textEvidenceColls
            
            // Update store metadata from MDH
            s.updateStoreMetadataWithStoreDisplayInfo(mdhStore)
            s.storeMetadata.Location = mdh.ExtractStoreLocation(mdhStore)
            s.storeOperatingStatus = mdh.ExtractStoreOperatingStatus(mdhStore)
            
            return true, nil
        })
}
```

### Step 3: How `media.FetchAll()` Works

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                              media.FetchAll(ctx, scope, entity)                           │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
        ┌───────────────────────────────────────────────────────────────────────────┐
        │ 1. Get MediaEvidenceProductRegistry from context                          │
        │    productRegistry := impl.GetMediaEvidenceProductRegistry(ctx)           │
        └───────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
        ┌───────────────────────────────────────────────────────────────────────────┐
        │ 2. Get all registered fetchers for this EntityType (STORE)                │
        │    productFetchers := productRegistry.GetFetchersByType(ctx, STORE)       │
        │                                                                            │
        │    Example: { "PRODUCT_RESTAURANTS": RestaurantMediaFetcher,              │
        │               "PRODUCT_NEW_VERTICALS": NVMediaFetcher }                   │
        └───────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
        ┌───────────────────────────────────────────────────────────────────────────┐
        │ 3. Fetch from each product IN PARALLEL                                     │
        │                                                                            │
        │    for productName, fetcher := range productFetchers {                    │
        │        results[productName] = fetchEvidencesFromProduct(...)              │
        │    }                                                                       │
        │                                                                            │
        │    scatterFuture := futlib.ScatterKeyed(ctx, results, ...)               │
        │    return futlib.GatherKeyed(ctx, scatterFuture, AtLeastOne())           │
        └───────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
        ┌───────────────────────────────────────────────────────────────────────────┐
        │ 4. Each product fetcher:                                                   │
        │                                                                            │
        │    fetchEvidencesFromProduct():                                           │
        │    ┌────────────────────────────────────────────────────────────────────┐ │
        │    │ a) Retrieve: rawMedia := fetcher.Retriever.Retrieve(ctx, entity)  │ │
        │    │                                                                    │ │
        │    │ b) Rank (optional): ranked := fetcher.Ranker.Rank(ctx, rawMedia) │ │
        │    │                                                                    │ │
        │    │ c) Return: []types.MediaEvidence                                  │ │
        │    └────────────────────────────────────────────────────────────────────┘ │
        └───────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
        ┌───────────────────────────────────────────────────────────────────────────┐
        │ 5. Blend all product results (optional)                                    │
        │                                                                            │
        │    if blender != nil {                                                    │
        │        return blender.Blend(ctx, unblendedResults)                        │
        │    }                                                                       │
        │    return flattenMap(unblendedResults)  // Just combine all               │
        └───────────────────────────────────────────────────────────────────────────┘
```

### Step 4: Concrete Example - Restaurant Media Retriever

From `product/restaurants/internal/evidence/media/item/retriever.go`:

```go
type retriever struct {
    _ gr.Component[Retriever] `gr:"name=restaurant-item-image-media-retriever"`
}

func (r *retriever) Retrieve(ctx context.Context, scope scope.ParentForEntityScope, entity entity.Entity) fut.Future[[]types.MediaEvidence] {
    store, ok := entity.(*rxstore.Store)
    if !ok {
        return fut.Value([]types.MediaEvidence{})
    }
    
    // Create ItemImage media evidence for each item in the store
    mockItems := store.Items
    itemImages := make([]types.MediaEvidence, len(mockItems))
    for i, mockItem := range mockItems {
        imageURN := s3ImageUrlPrefix + "photos/45eb3303-d705-4f95-8375-fccd4342ddd8-retina-large.jpg"
        itemImages[i] = NewItemImage(imageURN, mockItem)
    }
    
    return fut.Value(itemImages)
}
```

## Complete Hydration Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           EXAMPLE: STORE HYDRATION FLOW                              │
└─────────────────────────────────────────────────────────────────────────────────────┘

 Store Entity (after blending, before hydration):
 ┌──────────────────────────────────────────────┐
 │  Store                                       │
 │  ├── ID: "store-123"                         │
 │  ├── Name: "Pizza Palace"                    │
 │  ├── mediaEvidences: []     ◀── EMPTY       │
 │  ├── textEvidences: []      ◀── EMPTY       │
 │  └── storeOperatingStatus: nil              │
 └──────────────────────────────────────────────┘
                        │
                        │  PostRankHydrate(ctx, scope)
                        ▼
 ┌────────────────────────────────────────────────────────────────────────────────────┐
 │                           THREE PARALLEL FETCHES                                    │
 └────────────────────────────────────────────────────────────────────────────────────┘
         │                           │                           │
         ▼                           ▼                           ▼
 ┌───────────────────┐     ┌───────────────────┐     ┌───────────────────┐
 │  MEDIA FETCH      │     │  TEXT FETCH       │     │  MDH FETCH        │
 │                   │     │                   │     │                   │
 │ Registry lookup   │     │ Registry lookup   │     │ Batched store     │
 │ → Get fetchers    │     │ → Get fetchers    │     │   lookup          │
 │   for STORE type  │     │   for STORE type  │     │                   │
 │                   │     │                   │     │                   │
 │ Products:         │     │ Products:         │     │ GetStores([...])  │
 │ • Restaurants     │     │ • Restaurants     │     │                   │
 │ • New Verticals   │     │ • Incentives      │     │                   │
 │ • Ads             │     │ • Pricing         │     │                   │
 │                   │     │ • ETA             │     │                   │
 │ Each: Retrieve→   │     │ Each: Retrieve→   │     │                   │
 │       Rank        │     │       Rank        │     │                   │
 │       (parallel)  │     │       (parallel)  │     │                   │
 └───────────────────┘     └───────────────────┘     └───────────────────┘
         │                           │                           │
         └───────────────────────────┼───────────────────────────┘
                                     │
                                     ▼
                    ┌────────────────────────────────────┐
                    │  MapValue3() - Wait for all three  │
                    │  then apply to store entity        │
                    └────────────────────────────────────┘
                                     │
                                     ▼
 Store Entity (after hydration):
 ┌──────────────────────────────────────────────────────────────────────────────────────┐
 │  Store                                                                               │
 │  ├── ID: "store-123"                                                                 │
 │  ├── Name: "Pizza Palace"                                                           │
 │  ├── mediaEvidences: [                                                              │
 │  │       ItemImage{url: "s3://...", item: "Pepperoni Pizza"},                       │
 │  │       ItemImage{url: "s3://...", item: "Margherita Pizza"},                      │
 │  │       StoreHeroImage{url: "s3://..."}                                            │
 │  │   ]                                                                              │
 │  ├── textEvidences: [                                                               │
 │  │       ETABadge{text: "25-35 min"},                                               │
 │  │       RatingBadge{text: "4.8 (500+)"},                                           │
 │  │       IncentiveBadge{text: "$0 delivery fee"}                                    │
 │  │   ]                                                                              │
 │  └── storeOperatingStatus: {isOpen: true, nextCloseTime: "10:00 PM"}               │
 └──────────────────────────────────────────────────────────────────────────────────────┘
```

## Key Design Patterns

### 1. Registry Pattern

Products register their evidence fetchers at surface initialization time via `RegisterProducts()`:

```go
// In orchestration/register.go
for _, config := range mediaFetcherConfigs {
    productName := config.Name()
    entityType := config.EntityType()
    fetcher := mediaapi.NewSingleProductFetcher(config.Retriever(), config.Ranker())
    mediaRegistry.RegisterFetcher(entityType, productName, fetcher)
}
```

### 2. Parallel Execution

All evidence fetching happens in parallel using `ScatterKeyed` + `GatherKeyed`:

```go
// Fire off all product fetches in parallel
resultsFutures := make(map[string]future.Future[...])
for productName, fetcher := range productFetchers {
    resultsFutures[productName] = fetchEvidencesFromProduct(...)
}
// Scatter (fire all) and gather (wait for all)
scatterFuture := futlib.ScatterKeyed(ctx, resultsFutures, ...)
return futlib.GatherKeyed(ctx, scatterFuture, futlib.AtLeastOne())
```

### 3. Timeout Protection

Each product fetch has a 1-second timeout to prevent slow products from blocking:

```go
const MediaEvidenceFetchTimeout = 1 * time.Second

func fetchEvidencesFromProduct(...) {
    fetchCtx, cancel := context.WithTimeout(ctx, MediaEvidenceFetchTimeout)
    defer cancel()
    
    rawMedia := productFetcher.Retriever.Retrieve(fetchCtx, scope, entity)
    members, err := rawMedia.AwaitIn(futScope)
    if err != nil {
        // Timeout → log and skip gracefully
        scope.Logger().Warn(ctx, "media_evidence_fetch_timeout_skipping_product", ...)
        return []types.MediaEvidence{}, nil
    }
}
```

### 4. Retrieve → Rank Pipeline

Each product can optionally rank its evidences:

```go
// Retrieve raw data
rawMedia := productFetcher.Retriever.Retrieve(fetchCtx, scope, entity)
members, _ := rawMedia.AwaitIn(futScope)

// Rank (optional)
if productFetcher.Ranker != nil {
    ranked := productFetcher.Ranker.Rank(fetchCtx, scope, entity, members)
    return ranked.AwaitIn(futScope)
}
return members
```

## Summary

| Hydration Type | When | What | Method |
|---------------|------|------|--------|
| **Pre-Blend** | After fetch, before blend | Collection members | `FetchMembers()` + `BlendMembers()` |
| **Post-Rank** | After blend, before serialize | Entity evidences (media, text, MDH data) | `PostRankHydrate()` |

The key insight is that **hydration is deferred until after blending** so we only fetch detailed data for entities that will actually appear in the final feed—saving unnecessary network calls for filtered-out entities.

## Related Files

- `platform/pkg/feed/impl/feed.go` - Main orchestration pipeline with `hydrateUnits()` and `postRankHydrate()`
- `platform/internal/orchestration/evidence/media/fetcher.go` - Media evidence fetching
- `platform/internal/orchestration/evidence/text/fetcher.go` - Text evidence fetching
- `platform/pkg/entity/store/core_store.go` - Store entity `PostRankHydrate()` implementation
- `platform/pkg/collection/impl/collection.go` - Collection `FetchMembers()` and `BlendMembers()`

