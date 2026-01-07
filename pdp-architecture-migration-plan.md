# PDP Architecture & Migration Plan

This document outlines the architecture options, detailed migration plan, and implementation guide for migrating the PDP from the legacy Kotlin feed-service to FDM v2 in Pedregal.

---

## Table of Contents

1. [Architecture Options Analysis](#architecture-options-analysis)
2. [Recommended Architecture](#recommended-architecture)
3. [Layout Configuration Strategy](#layout-configuration-strategy)
4. [High-Level Go Node Architecture](#high-level-go-node-architecture)
5. [FDM Orchestration Framework](#fdm-orchestration-framework)
6. [Detailed Migration Guide](#detailed-migration-guide)
7. [Component-by-Component Migration](#component-by-component-migration)
8. [Implementation Timeline](#implementation-timeline)

---

## Architecture Options Analysis

### The Core Question: Where Does Layout Live?

In the legacy system, layout is defined in JSON configs (`top_layout.json`, `bottom_layout.json`) that control:
- Component ordering
- Component grouping
- Spacing between groups
- Eligibility conditions (A/B testing)

In FDM v2, we must decide where this logic belongs.

---

### Option 1: Pure FDM - Layout Entirely in PG

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              FEED (Pedregal)                             │
│                                                                          │
│  Provides: Entities + Evidences (ONLY DATA)                             │
│  - PdpItemData (item entity with all evidences)                         │
│  - Collections (variants, related items)                                │
│  - Placements (banners)                                                 │
│  - NO ordering hints, NO layout info                                    │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        PRESENTATION GATEWAY                              │
│                                                                          │
│  Owns: All layout decisions                                             │
│  - Component ordering (image → name → price → ...)                      │
│  - Spacing/grouping logic                                               │
│  - Platform-specific layouts (web 2-col, mobile stack)                  │
│  - A/B testing on layout                                                │
│  - Eligibility evaluation                                               │
│  - Layout configs stored in PG's config system                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Pros:**
- ✅ Clean separation of concerns (Feed = data, PG = presentation)
- ✅ Follows FDM philosophy strictly
- ✅ PG can optimize layouts per platform without Feed changes
- ✅ Layout changes don't require Feed deployments
- ✅ Simpler Feed implementation

**Cons:**
- ❌ PG needs to build layout configuration system
- ❌ PG team takes on more complexity
- ❌ Some "layout" decisions (like which evidences to include) may need Feed involvement
- ❌ Loss of Feed's current flexibility in A/B testing component presence

**Best For:** Greenfield implementations where PG has layout infrastructure

---

### Option 2: Hybrid - Evidence Ordering in Feed, Visual Layout in PG

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              FEED (Pedregal)                             │
│                                                                          │
│  Provides: Entities + Ordered Evidences + Metadata                      │
│  - text_evidences[] in priority order (Feed decides what's important)   │
│  - media_evidences[] in display order                                   │
│  - FeedUnit ordering (entity → collection → placement)                  │
│  - "evidence_group" metadata for logical grouping                       │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        PRESENTATION GATEWAY                              │
│                                                                          │
│  Owns: Visual presentation                                              │
│  - Maps evidence order to component positions                           │
│  - Applies spacing based on evidence_group boundaries                   │
│  - Platform-specific rendering (web columns, mobile stack)              │
│  - Can reorder within groups if needed                                  │
│  - Handles interactive elements (ATC, stepper)                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Pros:**
- ✅ Feed controls business priority (what's most important to show)
- ✅ PG controls visual presentation (how to show it)
- ✅ Evidence ordering can be personalized/ranked in Feed
- ✅ Gradual migration path from legacy system
- ✅ Feed can A/B test evidence inclusion; PG can A/B test rendering

**Cons:**
- ❌ Shared responsibility can cause confusion
- ❌ Need to define clear boundary (what's "content ordering" vs "layout")
- ❌ PG still needs some config for rendering rules

**Best For:** Migrating existing systems while maintaining some Feed control

---

### Option 3: Configuration-as-Collection - Layout Metadata in Feed

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              FEED (Pedregal)                             │
│                                                                          │
│  Provides: Entities + Evidences + Layout Collection                     │
│                                                                          │
│  FeedData {                                                             │
│    feed_units: [                                                        │
│      EntityData { pdp_item },           // Main product                 │
│      CollectionData { variants },       // Variant items                │
│      PlacementData { layout_config }    // ← Layout as a Placement     │
│    ]                                                                    │
│  }                                                                      │
│                                                                          │
│  LayoutConfigPlacement {                                                │
│    groups: [                                                            │
│      { evidence_types: ["image"], spacing_after: "MEDIUM" },            │
│      { evidence_types: ["item_name", "item_price"], spacing_after: ... }│
│    ]                                                                    │
│  }                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        PRESENTATION GATEWAY                              │
│                                                                          │
│  Consumes: Layout config from Feed                                      │
│  - Reads layout_config placement                                        │
│  - Renders components according to config                               │
│  - Falls back to default if config missing                              │
└─────────────────────────────────────────────────────────────────────────┘
```

**Pros:**
- ✅ Single source of truth (Feed owns all config)
- ✅ Layout can be personalized/A/B tested in Feed
- ✅ PG is simpler (just follows instructions)
- ✅ Ops can change layouts via Feed config

**Cons:**
- ❌ Violates FDM philosophy (layout is not "data")
- ❌ Feed becomes more complex
- ❌ Platform-specific layouts harder to manage
- ❌ Tight coupling between Feed and PG

**Best For:** When Feed team wants full control and PG is thin

---

### Option 4: Metadata-Driven - Evidence Types as Layout Hints (RECOMMENDED)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              FEED (Pedregal)                             │
│                                                                          │
│  Provides: Entities with typed evidences (in priority order)            │
│                                                                          │
│  PdpItemData {                                                          │
│    core_item: { ... }                                                   │
│    text_evidences: [                                                    │
│      { item_name: {...} },      // Feed orders by importance            │
│      { price: {...} },                                                  │
│      { brand: {...} },                                                  │
│      { nutrition: {...} },                                              │
│    ]                                                                    │
│    media_evidences: [                                                   │
│      { hero_image: {...} },                                             │
│      { gallery_images: [...] },                                         │
│    ]                                                                    │
│  }                                                                      │
│                                                                          │
│  Registry in Feed: Controls which evidences to INCLUDE                  │
│  (via evidence fetcher registration)                                    │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        PRESENTATION GATEWAY                              │
│                                                                          │
│  Layout Config (PG-owned):                                              │
│  - Maps evidence types to visual components                             │
│  - Defines grouping rules based on evidence types                       │
│  - Controls spacing, positioning, web_layout                            │
│                                                                          │
│  Example PG Config:                                                     │
│  {                                                                      │
│    "component_groups": [                                                │
│      {                                                                  │
│        "evidence_types": ["hero_image"],                                │
│        "web_layout": "FULL",                                            │
│        "spacing_after": "MEDIUM"                                        │
│      },                                                                 │
│      {                                                                  │
│        "evidence_types": ["item_name", "price", "brand"],               │
│        "web_layout": "RIGHT",                                           │
│        "spacing_after": "MEDIUM"                                        │
│      }                                                                  │
│    ]                                                                    │
│  }                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

**Pros:**
- ✅ Clean separation: Feed = what data, PG = how to render
- ✅ Evidence types are stable contracts between Feed and PG
- ✅ Feed can add/remove evidences; PG configs adapt
- ✅ PG can have platform-specific configs (web, iOS, Android)
- ✅ Feed controls business logic (which evidences to include)
- ✅ PG controls presentation logic (where to put them)
- ✅ A/B testing: Feed tests evidence inclusion, PG tests rendering

**Cons:**
- ❌ Requires coordination on evidence type names
- ❌ PG needs layout config infrastructure
- ❌ New evidence types require PG config updates

**Best For:** Production systems needing clear ownership and contracts

---

## Recommended Architecture

### **Recommendation: Option 4 (Metadata-Driven)**

This option best aligns with FDM v2 principles while providing practical migration path.

### Architecture Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            CLIENT (Mobile/Web)                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │
┌─────────────────────────────────────────────────────────────────────────┐
│                        PRESENTATION GATEWAY                              │
│                                                                          │
│  Responsibilities:                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 1. Layout Configuration (PG-owned configs)                      │    │
│  │    - Component grouping by evidence type                        │    │
│  │    - Spacing rules                                              │    │
│  │    - Platform-specific layouts (web 2-col, mobile stack)        │    │
│  │    - web_layout positioning (LEFT, RIGHT, FULL)                 │    │
│  │                                                                  │    │
│  │ 2. Component Rendering                                          │    │
│  │    - Evidence → Visual component mapping                        │    │
│  │    - Text formatting (prices, distances)                        │    │
│  │    - Localization                                               │    │
│  │                                                                  │    │
│  │ 3. Interactive Elements                                         │    │
│  │    - ATC button (uses item data from Feed)                      │    │
│  │    - Quantity stepper                                           │    │
│  │    - Navigation actions                                         │    │
│  │                                                                  │    │
│  │ 4. Lazy Loading Triggers                                        │    │
│  │    - Reviews fetch (based on has_reviews hint)                  │    │
│  │    - Related items pagination                                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │
┌─────────────────────────────────────────────────────────────────────────┐
│                         FEED (Pedregal Graph Node)                       │
│                                                                          │
│  Responsibilities:                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 1. Entity & Evidence Orchestration                              │    │
│  │    - Register evidence fetchers                                 │    │
│  │    - Fetch and hydrate entities                                 │    │
│  │    - Order evidences by business priority                       │    │
│  │                                                                  │    │
│  │ 2. Business Logic                                               │    │
│  │    - Which evidences to include (via fetcher registration)      │    │
│  │    - Personalization (user-specific evidence selection)         │    │
│  │    - A/B testing on evidence inclusion                          │    │
│  │                                                                  │    │
│  │ 3. Data Aggregation                                             │    │
│  │    - Product data from MDH/catalog                              │    │
│  │    - Badges from BSF                                            │    │
│  │    - Pricing from pricing service                               │    │
│  │    - Variants from product service                              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │
┌─────────────────────────────────────────────────────────────────────────┐
│                          Backend Services                                │
│  MDH, Catalog, Pricing, BSF (Badges), Reviews, etc.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Layout Configuration Strategy

### Where Layouts Live

| Aspect | Owner | Location | Rationale |
|--------|-------|----------|-----------|
| **Evidence Inclusion** | Feed | Fetcher registration in Go node | Business decision on what data to show |
| **Evidence Ordering** | Feed | Fetcher priority / blender | Business priority |
| **Component Grouping** | PG | PG layout configs | Visual presentation |
| **Spacing/Padding** | PG | PG layout configs | Visual presentation |
| **Platform Layout** | PG | Platform-specific configs | Device-specific rendering |
| **Eligibility A/B** | Both | Feed for data, PG for rendering | Split testing |

### Migration of Legacy Configs

#### Legacy: `registry.json` → Feed Fetcher Registration

```json
// LEGACY: pdp/subs/registry.json
{
  "component_use_cases": {
    "item_name": true,
    "item_price": true,
    "image": true,
    "nutrition_content": true
  }
}
```

```go
// NEW: Feed node fetcher registration
productFeedFetchers := []feedapi.FetcherConfig{
    n.pdpItemFetcher,  // Always included
}

textEvidenceFetchers := []textapi.FetcherConfig{
    n.itemNameText,     // Maps to "item_name": true
    n.priceText,        // Maps to "item_price": true
    n.nutritionText,    // Maps to "nutrition_content": true
}

mediaEvidenceFetchers := []mediaapi.FetcherConfig{
    n.itemImageMedia,   // Maps to "image": true
}
```

**Toggling via DV (Dynamic Values):**
```go
if dvEvaluator.IsEnabled(ctx, "pdp_show_nutrition") {
    textEvidenceFetchers = append(textEvidenceFetchers, n.nutritionText)
}
```

#### Legacy: `top_layout.json` → PG Layout Config

```json
// LEGACY: pdp/subs/top_layout.json
{
  "component_groups": [
    {
      "component_slugs": ["image"],
      "spacing_after": "MEDIUM"
    },
    {
      "component_slugs": ["item_name", "item_price"],
      "spacing_after": "MEDIUM"
    }
  ]
}
```

```yaml
# NEW: PG Layout Config (example format)
pdp_layout:
  platform: all
  groups:
    - evidence_types: [pdp_item_image_media]
      web_layout: FULL
      spacing_after: MEDIUM
    - evidence_types: [pdp_item_name_text, pdp_price_text, pdp_brand_text]
      web_layout: RIGHT
      spacing_after: MEDIUM
    - evidence_types: [pdp_nutrition_text, pdp_specifications_text]
      web_layout: FULL
      render_as: collapsible_sections
```

#### Legacy: `bottom_layout.json` → PG with Feed Hints

```json
// LEGACY: pdp/subs/bottom_layout.json
[
  {
    "target_group": "default",
    "eligibility_conditions": [],
    "ordering": ["item_details", "nutrition_content", "reviews"]
  }
]
```

**New Approach:**
- Feed provides evidences in priority order
- PG config maps evidence types to bottom section
- Eligibility moved to Feed (DV-based fetcher registration) or PG (rendering A/B)

---

## High-Level Go Node Architecture

### Directory Structure

```
nodes/consumer/feed/surface/pdp/
├── BUILD.bazel
├── node.go                           # Main PDP node
├── node_test.go
├── graphs.go                         # Graph registration
├── system_tests.go                   # Guardian tests
├── orchestration/
│   ├── BUILD.bazel
│   ├── surface_scope.go              # PDP surface scope
│   ├── feed_blender.go               # PDP-specific blending
│   └── feed_blender_test.go
├── internal/
│   ├── dv/
│   │   ├── BUILD.bazel
│   │   └── dv.go                     # Dynamic value evaluations
│   └── collection/
│       ├── BUILD.bazel
│       └── member_blender_factory.go
└── productoverrides/
    └── pdp/
        ├── evidence/
        │   ├── text/
        │   │   ├── BUILD.bazel
        │   │   ├── item_name.go      # ItemName text evidence fetcher
        │   │   ├── price.go          # Price text evidence fetcher
        │   │   ├── brand.go
        │   │   ├── nutrition.go
        │   │   └── ...
        │   └── media/
        │       ├── BUILD.bazel
        │       └── item_image.go     # Image media evidence fetcher
        ├── entity/
        │   ├── BUILD.bazel
        │   └── item_hydrator.go      # PDP item entity hydration
        └── placement/
            ├── BUILD.bazel
            └── callout_banner.go     # Banner placement fetcher
```

### Node Implementation

```go
// nodes/consumer/feed/surface/pdp/node.go
package pdp

import (
    "context"
    "fmt"

    "github.com/doordash/graph-runner-2/pkg/future"
    "github.com/doordash/graph-runner-2/pkg/gr"

    "github.com/doordash/pedregal/libraries/go/futurelib"
    orchestrationFramework "github.com/doordash/pedregal/nodes/consumer/feed/platform/internal/orchestration"
    "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/consumer"
    mediaapi "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/evidence/media/api"
    textapi "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/evidence/text/api"
    feedapi "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/feed/api"
    feedimpl "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/feed/impl"
    "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/globalrepository"
    "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/observability"
    "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/observability/log"
    "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/orchestrated"
    "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/scope"
    "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/types"
    pdporchestration "github.com/doordash/pedregal/nodes/consumer/feed/surface/pdp/orchestration"
    "github.com/doordash/pedregal/nodes/consumer/feed/surface/pdp/internal/dv"

    // Product configs
    pdppkg "github.com/doordash/pedregal/nodes/consumer/feed/product/pdp/pkg"

    // Feed Fetchers
    pdpfeed "github.com/doordash/pedregal/nodes/consumer/feed/product/pdp/pkg/feed"

    // Text Evidence Fetchers
    pdpnametext "github.com/doordash/pedregal/nodes/consumer/feed/product/pdp/pkg/evidence/text/itemname"
    pdppricetext "github.com/doordash/pedregal/nodes/consumer/feed/product/pdp/pkg/evidence/text/price"
    pdpbrandtext "github.com/doordash/pedregal/nodes/consumer/feed/product/pdp/pkg/evidence/text/brand"
    pdpnutritiontext "github.com/doordash/pedregal/nodes/consumer/feed/product/pdp/pkg/evidence/text/nutrition"
    pdpspecstext "github.com/doordash/pedregal/nodes/consumer/feed/product/pdp/pkg/evidence/text/specifications"
    pdpdesctext "github.com/doordash/pedregal/nodes/consumer/feed/product/pdp/pkg/evidence/text/description"
    pdpmetadatatext "github.com/doordash/pedregal/nodes/consumer/feed/product/pdp/pkg/evidence/text/metadata"

    // Media Evidence Fetchers
    pdpimagemedia "github.com/doordash/pedregal/nodes/consumer/feed/product/pdp/pkg/evidence/media/itemimage"

    // Proto imports
    corefeedpb "github.com/doordash/pedregal/protos/public/feed/v2/platform/core/feed"
    feedpb "github.com/doordash/pedregal/protos/public/feed/v2/platform/extensibility/feed"
    pdppb "github.com/doordash/pedregal/protos/public/feed/v2/service/pdp"
)

type (
    GetProductDetailPageFeed = pdppb.ProductDetailPage_GetProductDetailPageFeed
)

type node struct {
    _          gr.Node[GetProductDetailPageFeed] `gr:"name=feed-surface-pdp"`
    dvEvaluator dv.DVEvaluator
    globalRepo globalrepository.GlobalRepository
    feedBlender pdporchestration.PdpFeedBlender

    // Product configuration
    pdpProduct pdppkg.ProductConfig

    // Feed fetcher (retrieves product entity)
    pdpFeedFetcher pdpfeed.FetcherConfig

    // Text evidence fetchers (each maps to a legacy component)
    itemNameText     pdpnametext.FetcherConfig
    priceText        pdppricetext.FetcherConfig
    brandText        pdpbrandtext.FetcherConfig
    nutritionText    pdpnutritiontext.FetcherConfig
    specificationsText pdpspecstext.FetcherConfig
    descriptionText  pdpdesctext.FetcherConfig
    metadataText     pdpmetadatatext.FetcherConfig

    // Media evidence fetchers
    itemImageMedia pdpimagemedia.FetcherConfig

    // TODO: Placement fetchers for banners
    // freshnesssBannerPlacement ...
}

func (n *node) GetProductDetailPageFeed(
    ctx context.Context, 
    req *pdppb.GetProductDetailPageFeedRequest,
) future.Future[*pdppb.GetProductDetailPageFeedResponse] {
    // 1. Context Setup
    ctx, _ = scope.EnrichContextWithLocale(ctx, req)
    surfaceScope := pdporchestration.NewPDPSurfaceScope(ctx, req)
    ctx = log.WithLogger(ctx, surfaceScope.Logger())
    
    observability.AddDimensions(ctx,
        observability.WithDimension("pdp_type", req.GetPdpType().String()),
        observability.WithDimension("product_id", req.GetProductId()),
        observability.WithDimension("store_id", req.GetStoreId()),
    )

    surfaceScope.Logger().Info(ctx, "[GetProductDetailPageFeed] Processing request",
        log.Any("product_id", req.GetProductId()),
        log.Any("store_id", req.GetStoreId()),
    )

    // 2. Register Products
    products := map[types.ProductName]orchestrated.ProductConfig{
        types.ProductPDP: n.pdpProduct,
    }

    ctx = globalrepository.WithGlobalRepository(ctx, n.globalRepo)
    ctx = feedimpl.WithStoreRetrieverRegistry(ctx)

    // 3. Build Evidence Fetcher Configs (replaces registry.json)
    textEvidenceFetchers := n.buildTextEvidenceFetchers(ctx, surfaceScope)
    mediaEvidenceFetchers := n.buildMediaEvidenceFetchers(ctx, surfaceScope)

    ctx, err := orchestrationFramework.RegisterProducts(
        ctx, products, mediaEvidenceFetchers, textEvidenceFetchers,
    )
    if err != nil {
        return future.Error[*pdppb.GetProductDetailPageFeedResponse](err)
    }

    // 4. Register Feed Fetchers
    ctx = feedimpl.WithProductFetcherRegistry(ctx)
    feedRegistry := feedimpl.GetProductFetcherRegistry(ctx)
    if feedRegistry == nil {
        return future.Error[*pdppb.GetProductDetailPageFeedResponse](
            fmt.Errorf("product fetcher registry not set up"),
        )
    }

    productFeedFetchers := []feedapi.FetcherConfig{
        n.pdpFeedFetcher,
    }

    if err := feedRegistry.RegisterAll(productFeedFetchers); err != nil {
        return future.Error[*pdppb.GetProductDetailPageFeedResponse](err)
    }

    // 5. Create Feed Scope and Fetch
    feedScope := scope.NewFeedScope(
        ctx, 
        &consumer.Consumer{Id: surfaceScope.ConsumerId()}, 
        types.FeedType_ProductDetailPage, 
        surfaceScope,
    )
    ctx = log.WithLogger(ctx, feedScope.Logger())

    // 6. Fetch All - Orchestration happens here
    fetchFuture := feedimpl.FetchAll(ctx, feedScope, types.FeedType_ProductDetailPage, n.feedBlender)

    // 7. Build Response
    return future.FlatMapResult(ctx, fetchFuture, 
        func(ctx context.Context, result future.Result[[]scope.FeedUnit]) future.Future[*pdppb.GetProductDetailPageFeedResponse] {
            feedUnits, err := result.Unwrap()
            if err != nil {
                observability.RecordAvailability(ctx, false)
                return future.Value(buildEmptyResponse())
            }

            feedDataFuture := convertFeedUnitsToProto(ctx, feedScope, feedUnits)

            return future.MapValue(ctx, feedDataFuture, 
                func(ctx context.Context, feedData []*feedpb.FeedUnitData) (*pdppb.GetProductDetailPageFeedResponse, error) {
                    defer observability.RecordLatencySince(
                        ctx, 
                        surfaceScope.SurfaceType().String(), 
                        surfaceScope.RequestStartTime(ctx),
                    )

                    feedDataBuilder := corefeedpb.FeedData_builder{FeedUnits: feedData}.Build()
                    observability.RecordAvailability(ctx, true)

                    return pdppb.GetProductDetailPageFeedResponse_builder{
                        FeedData: feedDataBuilder,
                    }.Build(), nil
                })
        })
}

// buildTextEvidenceFetchers - Replaces legacy registry.json logic
// This is where you control which evidences to include
func (n *node) buildTextEvidenceFetchers(
    ctx context.Context, 
    surfaceScope scope.SurfaceScope,
) []textapi.FetcherConfig {
    fetchers := []textapi.FetcherConfig{
        // Always included (core evidences)
        n.itemNameText,
        n.priceText,
    }

    // Conditionally included based on DV or context
    if n.dvEvaluator.IsEnabled(ctx, "pdp_show_brand", surfaceScope.ConsumerId()) {
        fetchers = append(fetchers, n.brandText)
    }

    if n.dvEvaluator.IsEnabled(ctx, "pdp_show_nutrition", surfaceScope.ConsumerId()) {
        fetchers = append(fetchers, n.nutritionText)
    }

    if n.dvEvaluator.IsEnabled(ctx, "pdp_show_specifications", surfaceScope.ConsumerId()) {
        fetchers = append(fetchers, n.specificationsText)
    }

    // Always include description and metadata
    fetchers = append(fetchers, n.descriptionText, n.metadataText)

    return fetchers
}

// buildMediaEvidenceFetchers - Media evidence registration
func (n *node) buildMediaEvidenceFetchers(
    ctx context.Context, 
    surfaceScope scope.SurfaceScope,
) []mediaapi.FetcherConfig {
    return []mediaapi.FetcherConfig{
        n.itemImageMedia,
    }
}

// Helper functions
func buildEmptyResponse() *pdppb.GetProductDetailPageFeedResponse {
    emptyFeedData := corefeedpb.FeedData_builder{
        FeedUnits: []*feedpb.FeedUnitData{},
    }.Build()
    return pdppb.GetProductDetailPageFeedResponse_builder{
        FeedData: emptyFeedData,
    }.Build()
}

func convertFeedUnitsToProto(
    ctx context.Context, 
    scope scope.FeedScope, 
    units []scope.FeedUnit,
) future.Future[[]*feedpb.FeedUnitData] {
    futures := make([]future.Future[*feedpb.FeedUnitData], len(units))
    for i, unit := range units {
        futures[i] = unit.ConvertToFeedUnit(ctx, scope)
    }

    scatterFuture := futurelib.Scatter(ctx, futures, "pdp.convertFeedUnitsToProto()")
    return futurelib.Gather(ctx, scatterFuture, futurelib.AllOrNothing())
}
```

---

## FDM Orchestration Framework

### Key Concepts Mapping

| Legacy Kotlin | FDM Orchestration |
|---------------|-------------------|
| `PdpComponentRegistry` | `orchestrationFramework.RegisterProducts()` |
| `PdpComponentFactory` | DI-injected fetcher configs |
| `PdpComponentUseCase` | Evidence Fetcher |
| `PdpContentFetcher` | Feed Fetcher + Entity Hydrator |
| `fetchAdditionalData()` | Evidence Fetcher async operations |
| `serializeComponent()` | Evidence proto builder |
| `PdpResponseSerializer` | `convertFeedUnitsToProto()` |
| `registry.json` | Fetcher registration in node |
| `top_layout.json` | PG layout config |
| `bottom_layout.json` | PG layout config + Feed ordering |

### Orchestration Flow in FDM

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          FDM Orchestration                               │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. Surface Scope Creation                                                │
│    - NewPDPSurfaceScope(ctx, req)                                       │
│    - Contains: consumerId, productId, storeId, pdpType                  │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. Product Registration                                                  │
│    - RegisterProducts(ctx, products, mediaFetchers, textFetchers)       │
│    - Sets up evidence fetcher registries                                │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. Feed Fetcher Registration                                            │
│    - feedRegistry.RegisterAll(productFeedFetchers)                      │
│    - PDP feed fetcher retrieves product entity from MDH/catalog         │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. Feed Scope Creation                                                   │
│    - NewFeedScope(ctx, consumer, FeedType_ProductDetailPage, surfaceScope)
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. FetchAll Orchestration                                                │
│    feedimpl.FetchAll(ctx, feedScope, feedType, blender)                 │
│                                                                          │
│    Internally:                                                          │
│    a. Fetch → Retrieve entities from feed fetchers                      │
│    b. Rank  → Order entities within product                             │
│    c. Prune → Remove ineligible entities                                │
│    d. Blend → Merge across products (if multiple)                       │
│    e. PostProcess → Final adjustments                                   │
│    f. Hydrate → Attach evidences to entities                            │
│                                                                          │
│    Evidence Hydration:                                                  │
│    - For each entity, run registered evidence fetchers                  │
│    - Text fetchers: itemName, price, brand, nutrition, ...              │
│    - Media fetchers: itemImage                                          │
│    - Evidences attached to entity in registration order                 │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 6. Convert to Proto                                                      │
│    convertFeedUnitsToProto(ctx, feedScope, feedUnits)                   │
│                                                                          │
│    Each FeedUnit becomes FeedUnitData:                                  │
│    - EntityData with core_item + evidences                              │
│    - CollectionData for variants                                        │
│    - PlacementData for banners                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Detailed Migration Guide

### Phase 1: Proto Definitions

#### Step 1.1: Create Service Proto

**File**: `protos/public/feed/v2/service/pdp/service.proto`

```protobuf
syntax = "proto3";

package feed.v2.service.pdp;

option java_generic_services = true;
option java_multiple_files = true;

import "config/config.proto";
import "feed/v2/platform/core/request/common.proto";
import "feed/v2/platform/core/feed/feed.proto";

service ProductDetailPage {
  rpc GetProductDetailPageFeed(GetProductDetailPageFeedRequest) 
      returns (GetProductDetailPageFeedResponse) {
    option (config.rpc).authority = "feed-v2-pdp.workloads.dash";
  }
}

message GetProductDetailPageFeedRequest {
  feed.v2.platform.core.request.Common common_request = 1;
  string product_id = 2;
  optional string store_id = 3;
  optional string variant_id = 4;
  PdpType pdp_type = 5;
}

enum PdpType {
  PDP_TYPE_UNSPECIFIED = 0;
  PDP_TYPE_STORE = 1;
  PDP_TYPE_SUBS = 2;
}

message GetProductDetailPageFeedResponse {
  feed.v2.platform.core.feed.FeedData feed_data = 1;
}
```

#### Step 1.2: Create Entity Proto

**File**: `protos/public/feed/v2/product/pdp/entity/item/item.proto`

```protobuf
syntax = "proto3";

package feed.v2.product.pdp.entity.item;

import "feed/v2/platform/core/entity/item/item.proto";
import "feed/v2/platform/core/entity/store/store.proto";
import "feed/v2/product/pdp/evidence/media/media.proto";
import "feed/v2/product/pdp/evidence/text/text.proto";

message PdpItemData {
  feed.v2.platform.core.entity.item.CoreItemData core_item = 1;
  feed.v2.platform.core.entity.store.CoreStoreData core_store = 2;
  PdpItemExtension pdp_extension = 3;
  repeated feed.v2.product.pdp.evidence.media.PdpMediaEvidence media_evidences = 4;
  repeated feed.v2.product.pdp.evidence.text.PdpTextEvidence text_evidences = 5;
}

message PdpItemExtension {
  PurchaseConfig purchase_config = 1;
  optional VariantInfo current_variant = 2;
  repeated string matching_item_ids = 3;
  optional bool is_saved_item = 4;
  optional BundleMetadata bundle_metadata = 5;
  optional AvailabilityInfo availability = 6;
  optional int64 item_limit = 7;
}

message PurchaseConfig {
  PurchaseType purchase_type = 1;
  optional IncrementConfig increment = 2;
  optional string display_unit = 3;
  
  enum PurchaseType {
    PURCHASE_TYPE_UNSPECIFIED = 0;
    PURCHASE_TYPE_UNIT = 1;
    PURCHASE_TYPE_WEIGHT = 2;
    PURCHASE_TYPE_DYNAMIC_PRICE = 3;
  }
  
  message IncrementConfig {
    int32 decimal_places = 1;
    int32 unit_amount = 2;
  }
}

message VariantInfo {
  string variant_type = 1;
  string variant_value = 2;
  optional string variant_id = 3;
  VariantDisplayType display_type = 4;
  optional string color_hex = 5;
  optional string swatch_image_url = 6;
  
  enum VariantDisplayType {
    VARIANT_DISPLAY_TYPE_UNSPECIFIED = 0;
    VARIANT_DISPLAY_TYPE_PILL = 1;
    VARIANT_DISPLAY_TYPE_SWATCH = 2;
    VARIANT_DISPLAY_TYPE_VISUAL = 3;
    VARIANT_DISPLAY_TYPE_PILL_V2 = 4;
  }
}

message BundleMetadata {
  string bundle_store_id = 1;
  string bundle_store_name = 2;
  optional string surface_type = 3;
  optional string callout_message = 4;
  optional string logo_url = 5;
}

message AvailabilityInfo {
  StockStatus stock_status = 1;
  optional string availability_message = 2;
  
  enum StockStatus {
    STOCK_STATUS_UNSPECIFIED = 0;
    STOCK_STATUS_IN_STOCK = 1;
    STOCK_STATUS_LOW_STOCK = 2;
    STOCK_STATUS_OUT_OF_STOCK = 3;
  }
}
```

#### Step 1.3: Create Text Evidence Proto

**File**: `protos/public/feed/v2/product/pdp/evidence/text/text.proto`

```protobuf
syntax = "proto3";

package feed.v2.product.pdp.evidence.text;

message PdpTextEvidence {
  oneof value {
    PdpItemNameTextData item_name = 1;
    PdpBrandTextData brand = 2;
    PdpPriceTextData price = 3;
    PdpPriceSubtextData price_subtext = 4;
    PdpDescriptionTextData description = 5;
    PdpMetadataTextData metadata = 6;
    PdpPromotionTextData promotion = 7;
    PdpNutritionTextData nutrition = 8;
    PdpSpecificationsTextData specifications = 9;
    PdpReviewsSummaryTextData reviews_summary = 10;
    PdpAvailabilityTextData availability = 11;
    PdpAffordabilityTextData affordability = 12;
    PdpRatingsTextData ratings = 13;
    PdpBadgesTextData badges = 14;
  }
}

// Each evidence type definition...
message PdpItemNameTextData {
  string display_name = 1;
}

message PdpBrandTextData {
  string brand_name = 1;
  optional string brand_id = 2;
}

message PdpPriceTextData {
  PriceDisplayType display_type = 1;
  int64 price_cents = 2;
  optional int64 original_price_cents = 3;
  optional int32 discount_percent = 4;
  optional UnitPriceInfo unit_price = 5;
  optional bool show_tax_indicator = 6;
  repeated PriceBadge price_badges = 7;
  
  enum PriceDisplayType {
    PRICE_DISPLAY_TYPE_UNSPECIFIED = 0;
    PRICE_DISPLAY_TYPE_REGULAR = 1;
    PRICE_DISPLAY_TYPE_DISCOUNT = 2;
    PRICE_DISPLAY_TYPE_RANGE = 3;
    PRICE_DISPLAY_TYPE_MEMBER = 4;
    PRICE_DISPLAY_TYPE_ITEM_MODIFIER = 5;
  }
  
  message UnitPriceInfo {
    int64 price_cents = 1;
    string unit = 2;
  }
  
  message PriceBadge {
    string badge_type = 1;
    string display_text = 2;
  }
}

message PdpPriceSubtextData {
  optional WeightEstimate weight_estimate = 1;
  optional string disclaimer_text = 2;
  optional string secondary_price_text = 3;
  
  message WeightEstimate {
    float estimated_value = 1;
    string unit = 2;
    bool is_approximate = 3;
  }
}

message PdpDescriptionTextData {
  optional string header = 1;
  string body = 2;
}

message PdpMetadataTextData {
  repeated MetadataEntry entries = 1;
  
  message MetadataEntry {
    string key = 1;
    string value = 2;
  }
}

message PdpPromotionTextData {
  string promo_title = 1;
  optional string promo_details = 2;
  optional string campaign_id = 3;
  optional TermsInfo terms = 4;
  
  message TermsInfo {
    string title = 1;
    string action_text = 2;
    string disclaimer = 3;
  }
}

message PdpNutritionTextData {
  optional string serving_size = 1;
  optional string servings_per_container = 2;
  repeated Nutrient nutrients = 3;
  optional string disclaimer = 4;
  optional string annotation = 5;
  
  message Nutrient {
    string label = 1;
    optional string total = 2;
    optional string percent_daily_value = 3;
    repeated Nutrient subcategories = 4;
  }
}

message PdpSpecificationsTextData {
  string section_title = 1;
  repeated Specification specs = 2;
  
  message Specification {
    string key = 1;
    string value = 2;
  }
}

message PdpReviewsSummaryTextData {
  optional float average_rating = 1;
  optional int32 review_count = 2;
  optional bool has_reviews = 3;
}

message PdpAvailabilityTextData {
  string status = 1;
  optional string message = 2;
}

message PdpAffordabilityTextData {
  repeated AffordabilityBadge badges = 1;
  
  message AffordabilityBadge {
    string badge_type = 1;
    string display_text = 2;
    optional string icon_url = 3;
  }
}

message PdpRatingsTextData {
  float rating = 1;
  int32 review_count = 2;
  optional string display_text = 3;
}

message PdpBadgesTextData {
  repeated Badge badges = 1;
  
  message Badge {
    string badge_type = 1;
    string display_text = 2;
    optional string icon_url = 3;
    optional string background_color = 4;
  }
}
```

#### Step 1.4: Create Media Evidence Proto

**File**: `protos/public/feed/v2/product/pdp/evidence/media/media.proto`

```protobuf
syntax = "proto3";

package feed.v2.product.pdp.evidence.media;

import "feed/v2/platform/core/evidence/media/media.proto";

message PdpMediaEvidence {
  oneof value {
    PdpItemImageMediaData item_image = 1;
  }
}

message PdpItemImageMediaData {
  feed.v2.platform.core.evidence.media.CoreMediaData core_media = 1;
  ImageType image_type = 2;
  optional int32 sort_order = 3;
  optional VisualAislesData visual_aisles = 4;
  
  enum ImageType {
    IMAGE_TYPE_UNSPECIFIED = 0;
    IMAGE_TYPE_HERO = 1;
    IMAGE_TYPE_GALLERY = 2;
    IMAGE_TYPE_THUMBNAIL = 3;
    IMAGE_TYPE_VISUAL_AISLES = 4;
  }
  
  message VisualAislesData {
    int32 width_pixels = 1;
    int32 height_pixels = 2;
    repeated BoundingBox bounding_boxes = 3;
    
    message BoundingBox {
      float x = 1;
      float y = 2;
      float width = 3;
      float height = 4;
      optional string product_id = 5;
    }
  }
}
```

#### Step 1.5: Create Placement Proto

**File**: `protos/public/feed/v2/product/pdp/placement/placement.proto`

```protobuf
syntax = "proto3";

package feed.v2.product.pdp.placement;

import "feed/v2/platform/core/placement/placement.proto";

message PdpCalloutBannerPlacement {
  feed.v2.platform.core.placement.CorePlacementData core_placement = 1;
  
  BannerType type = 2;
  string title_text = 3;
  optional BannerAction action = 4;
  
  enum BannerType {
    BANNER_TYPE_UNSPECIFIED = 0;
    BANNER_TYPE_FRESHNESS = 1;
    BANNER_TYPE_OUT_OF_STOCK = 2;
    BANNER_TYPE_LOW_STOCK = 3;
    BANNER_TYPE_PROMOTION = 4;
    BANNER_TYPE_REORDER_SUBSTITUTION = 5;
  }
  
  message BannerAction {
    ActionType action_type = 1;
    optional string url = 2;
    optional string screen_type = 3;
    
    enum ActionType {
      ACTION_TYPE_UNSPECIFIED = 0;
      ACTION_TYPE_NAVIGATE_URL = 1;
      ACTION_TYPE_SHOW_MODAL = 2;
      ACTION_TYPE_DISMISS = 3;
    }
  }
}
```

#### Step 1.6: Register in Extensibility

**File**: `protos/public/feed/v2/platform/extensibility/entity/entity.proto` (update)

```protobuf
// Add to existing EntityData oneof
message EntityData {
  oneof entity_type {
    // ... existing types
    feed.v2.product.pdp.entity.item.PdpItemData pdp_item = N;
  }
}
```

---

### Phase 2: Go Node Implementation

#### Step 2.1: Create Surface Scope

**File**: `nodes/consumer/feed/surface/pdp/orchestration/surface_scope.go`

```go
package orchestration

import (
    "context"

    "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/scope"
    "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/types"
    pdppb "github.com/doordash/pedregal/protos/public/feed/v2/service/pdp"
)

type pdpSurfaceScope struct {
    scope.SurfaceScopeBase[*pdppb.GetProductDetailPageFeedRequest]
    productID string
    storeID   string
    pdpType   pdppb.PdpType
}

func NewPDPSurfaceScope(
    ctx context.Context, 
    request *pdppb.GetProductDetailPageFeedRequest,
) scope.SurfaceScope {
    return &pdpSurfaceScope{
        SurfaceScopeBase: scope.NewSurfaceScopeBase(
            ctx, request, types.SurfaceType_ProductDetailPage,
        ),
        productID: request.GetProductId(),
        storeID:   request.GetStoreId(),
        pdpType:   request.GetPdpType(),
    }
}

func (s *pdpSurfaceScope) ProductID() string  { return s.productID }
func (s *pdpSurfaceScope) StoreID() string    { return s.storeID }
func (s *pdpSurfaceScope) PdpType() pdppb.PdpType { return s.pdpType }
```

#### Step 2.2: Create Evidence Fetchers

**File**: `nodes/consumer/feed/product/pdp/pkg/evidence/text/itemname/fetcher.go`

```go
package itemname

import (
    "context"

    "github.com/doordash/graph-runner-2/pkg/future"
    "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/scope"
    textapi "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/evidence/text/api"
    pdptextpb "github.com/doordash/pedregal/protos/public/feed/v2/product/pdp/evidence/text"
)

type FetcherConfig struct {
    // DI dependencies here
}

type fetcher struct {
    config FetcherConfig
}

func (f *fetcher) Fetch(
    ctx context.Context,
    feedScope scope.FeedScope,
    entity scope.Entity,
) future.Future[textapi.TextEvidence] {
    // Get item data from entity
    itemData := entity.GetItemData()
    if itemData == nil || itemData.Name == "" {
        return future.Value[textapi.TextEvidence](nil)
    }

    evidence := &pdptextpb.PdpTextEvidence{
        Value: &pdptextpb.PdpTextEvidence_ItemName{
            ItemName: &pdptextpb.PdpItemNameTextData{
                DisplayName: itemData.Name,
            },
        },
    }

    return future.Value[textapi.TextEvidence](evidence)
}

func (f *fetcher) EvidenceType() string {
    return "pdp_item_name_text"
}
```

**File**: `nodes/consumer/feed/product/pdp/pkg/evidence/text/price/fetcher.go`

```go
package price

import (
    "context"

    "github.com/doordash/graph-runner-2/pkg/future"
    "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/scope"
    textapi "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/evidence/text/api"
    pdptextpb "github.com/doordash/pedregal/protos/public/feed/v2/product/pdp/evidence/text"
)

type FetcherConfig struct {
    PriceService PriceServiceClient // DI injected
}

type fetcher struct {
    config FetcherConfig
}

func (f *fetcher) Fetch(
    ctx context.Context,
    feedScope scope.FeedScope,
    entity scope.Entity,
) future.Future[textapi.TextEvidence] {
    itemData := entity.GetItemData()
    if itemData == nil {
        return future.Value[textapi.TextEvidence](nil)
    }

    // Determine price type
    priceType := determinePriceType(itemData)
    
    evidence := &pdptextpb.PdpTextEvidence{
        Value: &pdptextpb.PdpTextEvidence_Price{
            Price: &pdptextpb.PdpPriceTextData{
                DisplayType:       priceType,
                PriceCents:        itemData.PriceCents,
                OriginalPriceCents: itemData.OriginalPriceCents,
                DiscountPercent:   calculateDiscount(itemData),
                ShowTaxIndicator:  itemData.HasTax,
            },
        },
    }

    return future.Value[textapi.TextEvidence](evidence)
}

func determinePriceType(itemData *ItemData) pdptextpb.PdpPriceTextData_PriceDisplayType {
    if itemData.HasDiscount {
        return pdptextpb.PdpPriceTextData_PRICE_DISPLAY_TYPE_DISCOUNT
    }
    if itemData.HasModifiers {
        return pdptextpb.PdpPriceTextData_PRICE_DISPLAY_TYPE_ITEM_MODIFIER
    }
    return pdptextpb.PdpPriceTextData_PRICE_DISPLAY_TYPE_REGULAR
}
```

#### Step 2.3: Create Feed Fetcher

**File**: `nodes/consumer/feed/product/pdp/pkg/feed/fetcher.go`

```go
package feed

import (
    "context"

    "github.com/doordash/graph-runner-2/pkg/future"
    "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/scope"
    feedapi "github.com/doordash/pedregal/nodes/consumer/feed/platform/pkg/feed/api"
)

type FetcherConfig struct {
    CatalogClient CatalogServiceClient
    BadgeService  BadgeServiceClient
}

type fetcher struct {
    config FetcherConfig
}

func (f *fetcher) Fetch(
    ctx context.Context,
    feedScope scope.FeedScope,
) future.Future[[]scope.FeedUnit] {
    return future.Suspendable(ctx, func(s future.SuspendableScope) ([]scope.FeedUnit, error) {
        // Get request from scope
        surfaceScope := feedScope.SurfaceScope()
        productID := surfaceScope.(PDPSurfaceScope).ProductID()
        storeID := surfaceScope.(PDPSurfaceScope).StoreID()

        // Fetch product from catalog
        productFuture := f.config.CatalogClient.GetProduct(ctx, productID, storeID)
        product, err := productFuture.AwaitIn(s)
        if err != nil {
            return nil, fmt.Errorf("failed to fetch product: %w", err)
        }

        // Fetch badges
        badgesFuture := f.config.BadgeService.GetBadges(ctx, productID)
        badges, _ := badgesFuture.AwaitIn(s) // Non-critical

        // Build entities
        entities := []scope.FeedUnit{}

        // Main item entity
        mainEntity := buildPdpItemEntity(product, badges)
        entities = append(entities, mainEntity)

        // Variant entities (as collection)
        if len(product.Variants) > 0 {
            variantCollection := buildVariantCollection(product.Variants)
            entities = append(entities, variantCollection)
        }

        return entities, nil
    })
}
```

---

### Phase 3: PG Layout Configuration

#### Step 3.1: PG Config Structure

```yaml
# PG Layout Config for PDP
pdp:
  layouts:
    - name: default
      platform: all
      groups:
        # Top section - Core product info
        - name: hero_section
          evidence_types:
            - pdp_item_image_media
          web_layout: FULL
          mobile_layout: FULL
          spacing_after: MEDIUM

        - name: banner_section
          placement_types:
            - pdp_callout_banner
          web_layout: FULL
          spacing_after: MEDIUM
          # Only show if placement exists

        - name: header_section
          evidence_types:
            - pdp_item_name_text
            - pdp_brand_text
          web_layout: RIGHT
          spacing_after: MEDIUM

        - name: price_section
          evidence_types:
            - pdp_price_text
            - pdp_price_subtext_text
            - pdp_affordability_text
            - pdp_promotion_text
            - pdp_ratings_text
            - pdp_badges_text
          web_layout: RIGHT
          spacing_after: MEDIUM

        # Bottom section - Additional info
        - name: details_section
          evidence_types:
            - pdp_description_text
          web_layout: FULL
          render_as: collapsible
          spacing_after: SEPARATOR

        - name: nutrition_section
          evidence_types:
            - pdp_nutrition_text
          web_layout: FULL
          render_as: collapsible
          spacing_after: SEPARATOR

        - name: specs_section
          evidence_types:
            - pdp_specifications_text
          web_layout: FULL
          render_as: collapsible
          spacing_after: SEPARATOR

        - name: reviews_section
          evidence_types:
            - pdp_reviews_summary_text
          web_layout: FULL
          trigger_lazy_load: true
          lazy_load_endpoint: /reviews

  # Interactive components (PG-owned, not from Feed)
  interactive:
    bottom_content:
      - type: quantity_stepper
        enabled: true
        condition: "stock_status != OUT_OF_STOCK"
      - type: add_to_cart
        enabled: true
        condition: "stock_status != OUT_OF_STOCK"

  # Variant selector (renders from variant collection)
  variant_selector:
    enabled: true
    render_style: auto  # PG determines pill/swatch/visual based on variant data
```

#### Step 3.2: PG Rendering Logic (Pseudocode)

```typescript
// PG rendering logic
function renderPDP(feedData: FeedData, config: PDPLayoutConfig) {
  const mainEntity = feedData.feedUnits.find(u => u.entity?.pdpItem);
  const variantCollection = feedData.feedUnits.find(u => u.collection);
  const bannerPlacement = feedData.feedUnits.find(u => u.placement?.calloutBanner);
  
  const components = [];
  
  for (const group of config.groups) {
    // Render evidences in group
    const evidences = getEvidencesForGroup(mainEntity, group.evidenceTypes);
    if (evidences.length === 0) continue;
    
    const groupComponent = {
      webLayout: group.webLayout,
      children: evidences.map(e => renderEvidence(e, group)),
    };
    
    components.push(groupComponent);
    
    // Add spacing
    if (group.spacingAfter === 'SEPARATOR') {
      components.push(<Separator />);
    } else {
      components.push(<Spacer size={group.spacingAfter} />);
    }
  }
  
  // Render variant selector (if collection exists)
  if (variantCollection && config.variantSelector.enabled) {
    components.push(renderVariantSelector(variantCollection));
  }
  
  // Render interactive bottom content (ATC, stepper)
  const bottomContent = renderBottomContent(mainEntity, config.interactive);
  
  return { components, bottomContent };
}
```

---

## Component-by-Component Migration

### Migration Matrix

| Legacy Component | Slug | FDM Evidence Type | Go Fetcher Package | PG Rendering |
|------------------|------|-------------------|-------------------|--------------|
| `ImagePdpComponentUseCase` | `image` | `PdpItemImageMediaData` | `pdp/pkg/evidence/media/itemimage` | Image carousel |
| `ItemNamePdpComponentUseCase` | `item_name` | `PdpItemNameTextData` | `pdp/pkg/evidence/text/itemname` | Text |
| `ItemPricePdpComponentUseCase` | `item_price` | `PdpPriceTextData` | `pdp/pkg/evidence/text/price` | Formatted price |
| `BrandPdpComponentUseCase` | `brand` | `PdpBrandTextData` | `pdp/pkg/evidence/text/brand` | Text/Link |
| `DetailsPdpComponentUseCase` | `item_details` | `PdpDescriptionTextData` | `pdp/pkg/evidence/text/description` | Collapsible |
| `SubtextPdpComponentUseCase` | `price_subtext` | `PdpPriceSubtextData` | `pdp/pkg/evidence/text/pricesubtext` | Subtext |
| `NutritionContentPdpComponentUseCase` | `nutrition_content` | `PdpNutritionTextData` | `pdp/pkg/evidence/text/nutrition` | Nutrition table |
| `ItemProductMetadataPdpComponentUseCase` | `item_product_metadata` | `PdpSpecificationsTextData` | `pdp/pkg/evidence/text/specifications` | Table |
| `AffordabilityMetadataEntryPdpComponentUseCase` | `metadata_entry_affordability` | `PdpAffordabilityTextData` | `pdp/pkg/evidence/text/affordability` | Badge |
| `RatingsMetadataEntryPdpComponentUseCase` | `metadata_entry_ratings` | `PdpRatingsTextData` | `pdp/pkg/evidence/text/ratings` | Rating display |
| `BadgesMetadataEntryPdpComponentUseCase` | `metadata_entry_badges` | `PdpBadgesTextData` | `pdp/pkg/evidence/text/badges` | Badge list |
| `PdpItemPromoMetadataEntryComponentUseCase` | `metadata_entry_pdp_item_promo` | `PdpPromotionTextData` | `pdp/pkg/evidence/text/promotion` | Promo badge |
| `ReviewsPdpComponentUseCase` | `reviews` | `PdpReviewsSummaryTextData` | `pdp/pkg/evidence/text/reviews` | PG triggers |
| `VariantsSectionPdpComponentUseCase` | `variants` | Collection | `pdp/pkg/feed` | PG renders |
| `FreshnessGuaranteeCalloutBannerPdpComponentUseCase` | `freshness_guarantee_callout_banner` | `PdpCalloutBannerPlacement` | `pdp/pkg/placement/banner` | Banner |
| `ReorderSubstitutionCalloutBannerComponentUseCase` | `reorder_substitution_callout_banner` | `PdpCalloutBannerPlacement` | `pdp/pkg/placement/banner` | Banner |
| `PdpUpsellCarouselsComponentUseCase` | `upsell_carousels` | Collection | Separate feed | Carousel |

### Components NOT Migrated to FDM (PG Only)

| Legacy Component | Reason | PG Action |
|------------------|--------|-----------|
| `Spacer` | Pure layout | PG config |
| `Separator` | Pure layout | PG config |
| `QuantityStepper` | Interaction | PG renders using entity data |
| `AddToCart` | Interaction | PG renders using entity data |
| `web_layout` | Layout position | PG config |
| `shouldTruncate` | Presentation | PG decides |
| `shouldCollapseInitially` | Presentation | PG decides |

---

## Implementation Timeline

### Phase 1: Foundation (Week 1-2)
- [ ] Create proto definitions
- [ ] Run Gazelle, verify builds
- [ ] Create surface scope
- [ ] Create basic node structure

### Phase 2: Core Evidences (Week 3-4)
- [ ] Implement feed fetcher (product data)
- [ ] Implement image media evidence
- [ ] Implement item_name text evidence
- [ ] Implement price text evidence
- [ ] Write unit tests

### Phase 3: Additional Evidences (Week 5-6)
- [ ] Implement brand, description evidences
- [ ] Implement nutrition, specifications evidences
- [ ] Implement badge/affordability evidences
- [ ] Implement promotion evidence

### Phase 4: Collections & Placements (Week 7-8)
- [ ] Implement variant collection
- [ ] Implement banner placements
- [ ] Implement related items collection

### Phase 5: PG Integration (Week 9-10)
- [ ] Define PG layout config schema
- [ ] Implement PG rendering logic
- [ ] Test evidence → component mapping
- [ ] Handle interactive elements

### Phase 6: Testing & Rollout (Week 11-12)
- [ ] Guardian system tests
- [ ] A/B testing setup
- [ ] Shadow traffic testing
- [ ] Gradual rollout

---

## Summary

### Key Decisions

1. **Layout Ownership**: PG owns visual layout configs; Feed owns evidence inclusion
2. **Evidence-Based Architecture**: Components become evidences with typed contracts
3. **Registry → Fetcher Registration**: Legacy JSON configs become Go code with DV support
4. **Variants → Collections**: Variant section becomes a Collection of Item Entities
5. **Banners → Placements**: Callout banners become PDP-specific Placements

### Benefits

- ✅ Clean separation of data (Feed) and presentation (PG)
- ✅ Typed contracts via proto definitions
- ✅ Flexible A/B testing at each layer
- ✅ Platform-specific layouts in PG
- ✅ Evidence ordering controlled by Feed
- ✅ Simpler Feed implementation

---

*Document Version: 1.0*
*Last Updated: December 2024*
