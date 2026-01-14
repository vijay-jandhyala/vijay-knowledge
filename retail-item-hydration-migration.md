# Retail Item Hydration Migration Guide

## Overview

This document outlines the migration strategy for `RetailItemFetchingV2.kt` from the Kotlin feed-service to Pedregal's Feed Hydration framework, **reusing the existing `newverticals` product structure**.

## Source Analysis: RetailItemFetchingV2.kt

The Kotlin file performs parallel data fetching for retail items with these responsibilities:

| Kotlin Component | Description | Data Fetched |
|------------------|-------------|--------------|
| `fetchItemData` | Base item information | Item metadata from catalog service |
| `fetchPromotions` | Promotional offers | Discounts, deals, promotional badges |
| `fetchBadges` | Display badges | Quality badges, certifications |
| `fetchVariants` | Item variants | Size, color, flavor options |
| `fetchAdsMetadata` | Advertising info | Sponsored placement data |
| `fetchSavedAtInfo` | User engagement | Save timestamps, list membership |

## Target Architecture: Pedregal Feed Hydration

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Feed Orchestration Pipeline                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  FETCH → HYDRATE → BLEND → POSTPROCESS → SERIALIZE                          │
│                ↑                                                             │
│                │                                                             │
│    ┌───────────┴───────────┐                                                │
│    │   PostRankHydrate()   │                                                │
│    └───────────┬───────────┘                                                │
│                │                                                             │
│    ┌───────────┴───────────────────────────────────┐                        │
│    │         Evidence Fetchers (Parallel)          │                        │
│    ├───────────────────────────────────────────────┤                        │
│    │  • MediaRetriever  → Images, thumbnails       │                        │
│    │  • TextRetriever   → Descriptions, labels     │                        │
│    │  • PromotionRetriever → Deals, discounts      │ ← NEW                  │
│    │  • BadgeRetriever  → Quality indicators       │ ← NEW                  │
│    │  • VariantRetriever → Size/color options      │ ← NEW                  │
│    │  • AdsRetriever    → Sponsored metadata       │ ← NEW                  │
│    │  • SavedAtRetriever → User engagement         │ ← NEW                  │
│    └───────────────────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Reusing `newverticals` Product Structure

The `newverticals` product already provides the scaffolding we need. We extend it rather than create a new product.

### Existing Structure to Reuse

```
product/newverticals/
├── pkg/
│   ├── entity/
│   │   ├── item/
│   │   │   └── item.go          # ✅ Reuse - embeds platform Item
│   │   └── store/
│   │       └── store.go         # ✅ Reuse - embeds platform Store
│   └── feed/
│       └── feed.go              # ✅ Reuse - FetcherConfig implementation
├── internal/
│   ├── feed/
│   │   ├── entity_retriever.go  # ✅ Reuse - fetches store metadata
│   │   └── entity_pruner.go     # ✅ Reuse - pruning logic
│   └── evidence/
│       ├── text/item/
│       │   └── item_text_retriever.go  # Template for new retrievers
│       └── media/item/
│           └── item_media_retriever.go # Template for new retrievers
```

### New Evidence Retrievers to Add

```
product/newverticals/internal/evidence/
├── text/item/
│   └── item_text_retriever.go       # Existing
├── media/item/
│   └── item_media_retriever.go      # Existing
├── promotion/item/                   # ← NEW
│   └── item_promotion_retriever.go
├── badge/item/                       # ← NEW
│   └── item_badge_retriever.go
├── variant/item/                     # ← NEW
│   └── item_variant_retriever.go
├── ads/item/                         # ← NEW
│   └── item_ads_retriever.go
└── savedat/item/                     # ← NEW
    └── item_savedat_retriever.go
```

## Component Mapping: Kotlin → Go

### 1. Base Item Fetching

**Kotlin:**
```kotlin
suspend fun fetchItemData(itemIds: List<String>): Map<String, ItemData>
```

**Go (Already handled by `entity_retriever.go`):**
```go
func (r *entityRetriever) Retrieve(ctx context.Context, feedScope scope.FeedScope) fut.Future[[]entity.Entity]
```

### 2. Promotions → PromotionRetriever

**Kotlin:**
```kotlin
suspend fun fetchPromotions(itemIds: List<String>): Map<String, List<Promotion>>
```

**Go Implementation:**
```go
// product/newverticals/internal/evidence/promotion/item/item_promotion_retriever.go
package item

type PromotionRetriever struct {
    promotionClient promo.Client
}

func (r *PromotionRetriever) Retrieve(
    ctx context.Context,
    scope scope.ParentForEntityScope,
    entity entity.Entity,
) fut.Future[[]types.PromotionEvidence] {
    item, ok := entity.(*nvitem.Item)
    if !ok {
        return fut.Value([]types.PromotionEvidence{})
    }
    
    return fut.Suspendable(ctx, func(s fut.SuspendableScope) ([]types.PromotionEvidence, error) {
        promos, err := r.promotionClient.GetPromotions(ctx, item.ID())
        if err != nil {
            return nil, fmt.Errorf("fetch promotions: %w", err)
        }
        
        evidences := make([]types.PromotionEvidence, len(promos))
        for i, p := range promos {
            evidences[i] = NewPromotionEvidence(p)
        }
        return evidences, nil
    })
}
```

### 3. Badges → BadgeRetriever

**Go Implementation:**
```go
// product/newverticals/internal/evidence/badge/item/item_badge_retriever.go
package item

type BadgeRetriever struct {
    badgeClient badge.Client
}

func (r *BadgeRetriever) Retrieve(
    ctx context.Context,
    scope scope.ParentForEntityScope,
    entity entity.Entity,
) fut.Future[[]types.BadgeEvidence] {
    item, ok := entity.(*nvitem.Item)
    if !ok {
        return fut.Value([]types.BadgeEvidence{})
    }
    
    return fut.Suspendable(ctx, func(s fut.SuspendableScope) ([]types.BadgeEvidence, error) {
        badges, err := r.badgeClient.GetBadges(ctx, item.ID())
        if err != nil {
            return nil, fmt.Errorf("fetch badges: %w", err)
        }
        return toBadgeEvidences(badges), nil
    })
}
```

### 4. Variants → VariantRetriever

**Go Implementation:**
```go
// product/newverticals/internal/evidence/variant/item/item_variant_retriever.go
package item

type VariantRetriever struct {
    catalogClient catalog.Client
}

func (r *VariantRetriever) Retrieve(
    ctx context.Context,
    scope scope.ParentForEntityScope,
    entity entity.Entity,
) fut.Future[[]types.VariantEvidence] {
    item, ok := entity.(*nvitem.Item)
    if !ok {
        return fut.Value([]types.VariantEvidence{})
    }
    
    return fut.Suspendable(ctx, func(s fut.SuspendableScope) ([]types.VariantEvidence, error) {
        variants, err := r.catalogClient.GetVariants(ctx, item.ID())
        if err != nil {
            return nil, fmt.Errorf("fetch variants: %w", err)
        }
        return toVariantEvidences(variants), nil
    })
}
```

### 5. Ads Metadata → AdsRetriever

**Go Implementation:**
```go
// product/newverticals/internal/evidence/ads/item/item_ads_retriever.go
package item

type AdsRetriever struct {
    adsClient ads.Client
}

func (r *AdsRetriever) Retrieve(
    ctx context.Context,
    scope scope.ParentForEntityScope,
    entity entity.Entity,
) fut.Future[[]types.AdsEvidence] {
    item, ok := entity.(*nvitem.Item)
    if !ok {
        return fut.Value([]types.AdsEvidence{})
    }
    
    return fut.Suspendable(ctx, func(s fut.SuspendableScope) ([]types.AdsEvidence, error) {
        adsMeta, err := r.adsClient.GetMetadata(ctx, item.ID())
        if err != nil {
            // Ads are optional - don't fail the whole hydration
            return []types.AdsEvidence{}, nil
        }
        return []types.AdsEvidence{NewAdsEvidence(adsMeta)}, nil
    })
}
```

### 6. Saved-At Info → SavedAtRetriever

**Go Implementation:**
```go
// product/newverticals/internal/evidence/savedat/item/item_savedat_retriever.go
package item

type SavedAtRetriever struct {
    userDataClient userdata.Client
}

func (r *SavedAtRetriever) Retrieve(
    ctx context.Context,
    scope scope.ParentForEntityScope,
    entity entity.Entity,
) fut.Future[[]types.SavedAtEvidence] {
    item, ok := entity.(*nvitem.Item)
    if !ok {
        return fut.Value([]types.SavedAtEvidence{})
    }
    
    consumerID := scope.FeedScope().Consumer().Id
    
    return fut.Suspendable(ctx, func(s fut.SuspendableScope) ([]types.SavedAtEvidence, error) {
        savedInfo, err := r.userDataClient.GetSavedAt(ctx, consumerID, item.ID())
        if err != nil {
            return []types.SavedAtEvidence{}, nil
        }
        return []types.SavedAtEvidence{NewSavedAtEvidence(savedInfo)}, nil
    })
}
```

## Registration Pattern

Register all retrievers in the surface node:

```go
// surface/retail/node.go or surface/homepage/node.go

func (n *node) registerRetailHydration(ctx context.Context) {
    // Existing registrations
    orchestration.RegisterMediaFetcher(ctx, productpb.Product_PRODUCT_NEW_VERTICALS, mediaRetriever)
    orchestration.RegisterTextFetcher(ctx, productpb.Product_PRODUCT_NEW_VERTICALS, textRetriever)
    
    // New retail hydration registrations
    orchestration.RegisterPromotionFetcher(ctx, productpb.Product_PRODUCT_NEW_VERTICALS, promotionRetriever)
    orchestration.RegisterBadgeFetcher(ctx, productpb.Product_PRODUCT_NEW_VERTICALS, badgeRetriever)
    orchestration.RegisterVariantFetcher(ctx, productpb.Product_PRODUCT_NEW_VERTICALS, variantRetriever)
    orchestration.RegisterAdsFetcher(ctx, productpb.Product_PRODUCT_NEW_VERTICALS, adsRetriever)
    orchestration.RegisterSavedAtFetcher(ctx, productpb.Product_PRODUCT_NEW_VERTICALS, savedAtRetriever)
}
```

## Item Entity Extension

Extend the existing `newverticals` Item to hold the new evidence types:

```go
// product/newverticals/pkg/entity/item/item.go

type Item struct {
    *platformitem.Item
    
    // Retail-specific evidence (populated during hydration)
    promotions []types.PromotionEvidence
    badges     []types.BadgeEvidence
    variants   []types.VariantEvidence
    adsInfo    *types.AdsEvidence
    savedAt    *types.SavedAtEvidence
}

func (i *Item) PostRankHydrate(ctx context.Context, parentScope scope.ParentForEntityScope) fut.Future[bool] {
    // Platform hydration (text + media)
    baseFuture := i.Item.PostRankHydrateByProductName(ctx, parentScope, i.ProductName())
    
    // Retail-specific hydration (parallel)
    promoFuture := promotion.FetchAll(ctx, parentScope, i)
    badgeFuture := badge.FetchAll(ctx, parentScope, i)
    variantFuture := variant.FetchAll(ctx, parentScope, i)
    adsFuture := ads.FetchAll(ctx, parentScope, i)
    savedAtFuture := savedat.FetchAll(ctx, parentScope, i)
    
    return fut.MapValue6(ctx, baseFuture, promoFuture, badgeFuture, variantFuture, adsFuture, savedAtFuture,
        func(ctx context.Context, base bool, promos []types.PromotionEvidence, badges []types.BadgeEvidence,
            variants []types.VariantEvidence, ads []types.AdsEvidence, savedAt []types.SavedAtEvidence) (bool, error) {
            i.promotions = promos
            i.badges = badges
            i.variants = variants
            if len(ads) > 0 {
                i.adsInfo = &ads[0]
            }
            if len(savedAt) > 0 {
                i.savedAt = &savedAt[0]
            }
            return true, nil
        })
}
```

## Migration Steps

### Phase 1: Evidence Type Definitions
1. Define protobuf types for `PromotionEvidence`, `BadgeEvidence`, `VariantEvidence`, `AdsEvidence`, `SavedAtEvidence`
2. Add evidence types to `platform/pkg/types/`

### Phase 2: Retriever Implementations
1. Create `promotion/item/item_promotion_retriever.go`
2. Create `badge/item/item_badge_retriever.go`
3. Create `variant/item/item_variant_retriever.go`
4. Create `ads/item/item_ads_retriever.go`
5. Create `savedat/item/item_savedat_retriever.go`

### Phase 3: Platform Registration
1. Add registration functions to `platform/pkg/orchestration/register.go`
2. Add fetcher infrastructure similar to `media/` and `text/`

### Phase 4: Entity Integration
1. Update `newverticals/pkg/entity/item/item.go` with new fields
2. Override `PostRankHydrate` to include retail-specific fetching

### Phase 5: Surface Integration
1. Register all retrievers in the appropriate surface node
2. Wire up component dependencies via Graph Runner DI

### Phase 6: Testing
1. Unit tests for each retriever
2. Integration tests using testcontainers
3. Guardian tests for end-to-end validation

## Key Benefits of This Approach

| Benefit | Description |
|---------|-------------|
| **Reuse** | Leverages existing `newverticals` entity and feed infrastructure |
| **Parallel Execution** | All evidence fetchers run concurrently via `future` package |
| **Timeout Protection** | Each fetcher has its own timeout via `context.WithTimeout` |
| **Graceful Degradation** | Optional data (ads, savedAt) won't fail the entire hydration |
| **Type Safety** | Go interfaces enforce correct implementation |
| **Testability** | Each retriever is independently testable with mocked clients |

## References

- [Feed Platform Architecture](../reference/dir-structure.md)
- [Feed Terminology](../reference/terminology.md)
- [FDM Documentation](../reference/fdm/0-intros.md)
- [New Verticals README](../../product/newverticals/README.md)

