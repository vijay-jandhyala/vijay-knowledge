---
description: Detailed architecture of the feed pipeline including data flow, hydration phases, and MDH integration patterns
---

# Feed Pipeline Architecture

This document provides a detailed technical architecture of the feed pipeline implementation. For a high-level overview, see [orchestration-overview.md](./orchestration-overview.md).

**Implementation**: [`platform/pkg/feed/impl/feed.go`](../../platform/pkg/feed/impl/feed.go)

---

## Pipeline Overview

The feed pipeline is a 6-stage asynchronous processing flow that transforms product data into a serialized feed response.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              FEED PIPELINE                                          │
│                                                                                     │
│   Request ──► Fetch ──► Hydrate ──► Blend ──► Post-Process ──► Hydrate ──► Proto   │
│                 │          │          │           │              │           │      │
│              Phase 1    Phase 2    Phase 3     Phase 4        Phase 5     Phase 6   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Phase Details

### Phase 1: Fetch From All Products (`fetchFromAllProductsAndDedupe`)

**Purpose**: Retrieve entities from all registered product fetchers in parallel.

**Implementation**: `feed.go:39-106`

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 1: FETCH                                                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   FeedFetcherConfigRegistry                                                         │
│            │                                                                        │
│            ├──► restaurants.FetcherConfig ──► SingleProductFetcher ──► []FeedUnit   │
│            │                                                                        │
│            ├──► newverticals.FetcherConfig ──► SingleProductFetcher ──► []FeedUnit  │
│            │                                                                        │
│            ├──► ads.FetcherConfig ──► SingleProductFetcher ──► []FeedUnit           │
│            │                                                                        │
│            └──► growth.FetcherConfig ──► SingleProductFetcher ──► []FeedUnit        │
│                                                                                     │
│                                     │                                               │
│                                     ▼                                               │
│                           dedupUnits() ──► map[product][]FeedUnit                   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**Key Components**:
- `FeedFetcherConfigRegistry`: Registry of all product fetchers
- `SingleProductFetcher`: Orchestrates retrieve → rank → prune for one product
- `dedupUnits()`: Deduplicates entities across products by composite key

**Output**: `map[string][]FeedUnit` - Map of product name to feed units

---

### Phase 2: Hydrate Units (`hydrateUnits`)

**Purpose**: Populate collection members and filter empty collections.

**Implementation**: `feed.go:166-253`

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 2: HYDRATE UNITS                                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   For each FeedUnit:                                                                │
│                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  If Collection:                                                             │   │
│   │                                                                             │   │
│   │    collection.FetchMembers()                                                │   │
│   │         │                                                                   │   │
│   │         ▼                                                                   │   │
│   │    MemberRetriever.Retrieve()  ──► []Member (entities or nested)           │   │
│   │         │                                                                   │   │
│   │         ▼                                                                   │   │
│   │    collection.BlendMembers()                                                │   │
│   │         │                                                                   │   │
│   │         ▼                                                                   │   │
│   │    MemberBlender.Blend()  ──► []Member (ranked, pruned)                     │   │
│   │                                                                             │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  If Placement:                                                              │   │
│   │    Hydrate the placement's nested collection (same flow as above)           │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│   Filter out collections with no members after blending                             │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**Key Functions**:
- `hydrateCollection()`: Calls FetchMembers + BlendMembers on a collection
- `hydratePlacement()`: Hydrates a placement's nested collection

---

### Phase 3: Blend Feed (`blendFeed`)

**Purpose**: Merge units from all products into a single ordered list.

**Implementation**: `feed.go:308-331`

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 3: BLEND                                                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   map[product][]FeedUnit                                                            │
│        │                                                                            │
│        │   restaurants: [Store1, Store2, Collection1]                               │
│        │   newverticals: [Collection2, Collection3]                                 │
│        │   ads: [Placement1, Placement2]                                            │
│        │                                                                            │
│        ▼                                                                            │
│   Blender.Blend()  (surface-specific, e.g., HomePageFeedBlender)                    │
│        │                                                                            │
│        │   - Interleaves products                                                   │
│        │   - Applies ranking rules                                                  │
│        │   - Deduplicates across products                                           │
│        │   - Enforces position constraints                                          │
│        │                                                                            │
│        ▼                                                                            │
│   []FeedUnit (single ordered list)                                                  │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**Key Point**: The blender is surface-specific and provided via the `doordashProvider.GetConfig()`.

---

### Phase 4: Post-Process Feed (`postProcessFeed`)

**Purpose**: Allow collections to adjust based on final feed position (primarily for Ads).

**Implementation**: `feed.go:333-377`

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 4: POST-PROCESS                                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   For each FeedUnit in blendedFeed:                                                 │
│                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  If Collection:                                                             │   │
│   │    collection.PostProcessMembers(blendedFeed)                               │   │
│   │                                                                             │   │
│   │    - Receives full blended feed context                                     │   │
│   │    - Can adjust members based on vertical position                          │   │
│   │    - Used by Ads for position-based adjustments                             │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  If Placement:                                                              │   │
│   │    Post-process the placement's collection                                  │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

### Phase 5: Post-Rank Hydration (`postRankHydrate`) ⭐

**Purpose**: Hydrate entity core data and fetch evidence.

**Implementation**: `feed.go:416-451`

**This is the most important phase for entity data fetching.**

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 5: POST-RANK HYDRATION                                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   For each FeedUnit:                                                                │
│        │                                                                            │
│        ▼                                                                            │
│   unit.PostRankHydrate(ctx, feedScope)                                              │
│                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  Store.PostRankHydrate()                                                    │   │
│   │                                                                             │   │
│   │    Phase A: CoreStore.PostRankHydrate()                                     │   │
│   │             │                                                               │   │
│   │             └──► MDH.GetStores()  ──► name, address, hours, etc.            │   │
│   │                                                                             │   │
│   │    Phase B: Store.HydrateItems() (via override)                             │   │
│   │             │                                                               │   │
│   │             └──► Create items, hydrate each                                 │   │
│   │                                                                             │   │
│   │    Phase C: evidence.FetchEvidences()                                       │   │
│   │             │                                                               │   │
│   │             └──► Text + Media evidence retrievers                           │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  Collection.PostRankHydrate()                                               │   │
│   │                                                                             │   │
│   │    For each Member:                                                         │   │
│   │      member.PostRankHydrate()                                               │   │
│   │                                                                             │   │
│   │    ┌───────────────────────────────────────────────────────────────────┐    │   │
│   │    │  nvitem.Item.PostRankHydrate()  ◄── YOUR NV ITEM HYDRATION       │    │   │
│   │    │                                                                   │    │   │
│   │    │    Phase A: MDH.GetOffers() ──► price, name, image (CORE DATA)   │    │   │
│   │    │                                                                   │    │   │
│   │    │    Phase B: evidence.FetchEvidences() ──► ratings, badges        │    │   │
│   │    └───────────────────────────────────────────────────────────────────┘    │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│   Filters out units that failed hydration                                           │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**Key Pattern**: Multi-phase hydration within PostRankHydrate:
1. **Phase A**: Fetch core data from MDH (price, name, image)
2. **Phase B**: Fetch supplementary evidence (ratings, badges)

---

### Phase 6: Convert to Proto (`convertFeedUnitsToProto`)

**Purpose**: Serialize feed units to proto format.

**Implementation**: `feed.go:453-471`

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 6: SERIALIZE                                                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   For each FeedUnit:                                                                │
│        │                                                                            │
│        ▼                                                                            │
│   unit.ConvertToFeedUnit(ctx, feedScope)                                            │
│        │                                                                            │
│        ├──► Serialize entity core data                                              │
│        │                                                                            │
│        ├──► Serialize text evidence                                                 │
│        │                                                                            │
│        └──► Serialize media evidence                                                │
│                                                                                     │
│   Output: []*feedpb.FeedUnitData                                                    │
│                                                                                     │
│   Wrap in FeedData proto:                                                           │
│   corefeedpb.FeedData_builder{FeedUnits: feedUnitData}.Build()                      │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## MDH Integration Patterns

### Where to Call MDH

| Data Type | Phase | Location | Example |
|-----------|-------|----------|---------|
| **Store Core Data** | 5 (PostRankHydrate) | `CoreStore.PostRankHydrate()` | name, address, hours |
| **Item Core Data** | 5 (PostRankHydrate) | `nvitem.Item.PostRankHydrate()` | price, name, image |
| **Category Items** | 2 (Hydrate Units) | `MdhCategoryItemRetriever.Retrieve()` | item list for category |
| **Evidence (Supplementary)** | 5 (PostRankHydrate) | `evidence.FetchEvidences()` | ratings, badges |

### MDH Call Hierarchy

```
PostRankHydrate (Phase 5)
    │
    ├── CoreStore.PostRankHydrate()
    │       │
    │       └── mdhdatacache.GetStores() ──► MDH.GetStores()  ✅ EARLY
    │
    ├── nvitem.Item.PostRankHydrate()  (TO IMPLEMENT)
    │       │
    │       └── mdh.GetOffers() ──► price, name, image        ✅ EARLY
    │
    └── evidence.FetchEvidences()
            │
            └── EvidenceRetriever.Retrieve() ──► MDH.GetOffers()  ❌ LATE (for core data)
```

### Recommended Pattern for NV Item Hydration

```go
// nvitem.Item.PostRankHydrate() implementation pattern
func (i *Item) PostRankHydrate(ctx context.Context, parentScope scope.ParentForEntityScope) fut.Future[bool] {
    return fut.Suspendable(ctx, func(scope fut.SuspendableScope) (bool, error) {
        // Phase A: Hydrate core item data from MDH (EARLY)
        mdh := // get MDH from context
        result, err := mdh.GetOffers(ctx, parentScope.SurfaceScope(), 
            string(i.StoreID), 
            []*mdhpb.OfferLookupId{{ItemId: lo.ToPtr(string(i.ItemId))}},
            &mdhpb.OfferInclusions{Pricing: true, DisplayInfo: true},
        ).AwaitIn(scope)
        
        if err == nil {
            offers, _ := result.Unpack()
            if len(offers) > 0 {
                i.Name = offers[0].GetDisplayInfo().GetTitle()
                i.PriceInCents = offers[0].GetPricing().GetUnitAmount()
                i.ImageURL = offers[0].GetDisplayInfo().GetImageUrl()
            }
        }

        // Phase B: Fetch supplementary evidence (ratings, social)
        evidenceResult, err := evidence.FetchEvidences(ctx, parentScope, i).AwaitIn(scope)
        if err != nil {
            return false, err
        }
        i.Item.SetEvidences(evidenceResult.MediaEvidences, evidenceResult.TextEvidenceCollections)
        return true, nil
    })
}
```

---

## Async Execution Model

The pipeline uses Graph Runner's `future` package for async execution:

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  ASYNC EXECUTION PATTERNS                                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   Parallel Fetch (Phase 1):                                                         │
│   ┌────────────────────────────────────────────────────────────────────────────┐    │
│   │  fut1 := restaurants.Fetch()  ───┐                                         │    │
│   │  fut2 := newverticals.Fetch() ───┼──► ScatterKeyed() ──► GatherKeyed()     │    │
│   │  fut3 := ads.Fetch()          ───┘                                         │    │
│   └────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
│   Parallel Hydration (Phase 5):                                                     │
│   ┌────────────────────────────────────────────────────────────────────────────┐    │
│   │  for i, unit := range blendedFeed {                                        │    │
│   │      futures[i] = unit.PostRankHydrate()  // Start all in parallel         │    │
│   │  }                                                                         │    │
│   │  Scatter() ──► Gather(AtLeastOne)                                          │    │
│   └────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
│   Sequential Chaining:                                                              │
│   ┌────────────────────────────────────────────────────────────────────────────┐    │
│   │  fetchResult                                                               │    │
│   │    └──► FlatMapValue ──► hydrateUnits()                                    │    │
│   │           └──► FlatMapValue ──► blendFeed()                                │    │
│   │                  └──► FlatMapValue ──► postProcessFeed()                   │    │
│   │                         └──► FlatMapValue ──► postRankHydrate()            │    │
│   │                                └──► MapValue ──► convertToProto()          │    │
│   └────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Error Handling

The pipeline implements graceful degradation:

| Phase | Error Behavior |
|-------|----------------|
| Fetch | `AtLeastOne()` - succeeds if any product returns data |
| Hydrate | Empty collections filtered out, doesn't fail pipeline |
| Blend | Failures convert to empty results |
| PostRankHydrate | Failed units filtered from final output |
| Serialize | Failures convert to empty feed with availability=false |

```go
// Error recovery pattern (feed.go:529-532)
fetchFuture = fut.MapError(ctx, fetchFuture, func(ctx context.Context, err error) ([]scope.FeedUnit, error) {
    observability.RecordAvailability(ctx, false)
    return []scope.FeedUnit{}, nil // Convert error to success with empty units
})
```

---

## Observability

Each phase records latency metrics:

| Metric | Phase |
|--------|-------|
| `fetch.fetchFromAllProductsAndDedupe` | Phase 1 |
| `fetch.hydrateUnits` | Phase 2 |
| `fetch.blendFeed` | Phase 3 |
| `fetch.postProcessFeed` | Phase 4 |
| `fetch.postRankHydration` | Phase 5 |

Spans are created using the surface name as tracer:
```go
tracerName := feedScope.SurfaceScope().SurfaceType().String()
prefixedOperationName := fmt.Sprintf("feed.fetch.%s", operationName)
```

---

## Related Documents

- [Orchestration Overview](./orchestration-overview.md) - High-level overview
- [FDM Entity](../reference/fdm/4-entity.md) - Entity data model
- [FDM Evidence](../reference/fdm/6a-evidence-media.md) - Evidence concepts
- [Terminology](../reference/terminology.md) - Feed platform terminology
