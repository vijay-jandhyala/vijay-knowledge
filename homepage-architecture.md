---
description: Complete E2E architecture documentation for the Homepage Feed node, from client request to response
---

# Homepage Feed Architecture

This document provides a comprehensive end-to-end architecture overview of the Homepage Feed system, from client request through data retrieval and response serialization.

## Table of Contents

1. [Overview](#overview)
2. [E2E Request Flow](#e2e-request-flow)
3. [Client Layer](#1-client-layer)
4. [Gateway Layer](#2-gateway-layer)
5. [Homepage Node Architecture](#3-homepage-node-architecture)
6. [Feed Orchestration Pipeline](#4-feed-orchestration-pipeline)
7. [Data Sources](#5-data-sources)
8. [Response Structure](#6-response-structure)
9. [App Configuration](#7-app-configuration)
10. [Key Design Decisions](#8-key-design-decisions)

---

## Overview

The Homepage Feed is the primary discovery surface for DoorDash and Wolt consumer apps. It displays personalized restaurants, carousels, and promotional content based on consumer location and preferences.

| Attribute | Value |
|-----------|-------|
| **Graph Name** | `feed-v2-discovery-get-homepage-feed` |
| **Node Name** | `feed-surface-homepage-v2` |
| **gRPC Service** | `feed.v2.service.discovery.Discovery` |
| **gRPC Method** | `GetHomepageFeed` |
| **Authority** | `feed-v2-discovery-get-homepage-feed.workloads.dash` |

---

## E2E Request Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    HOMEPAGE E2E ARCHITECTURE                                                 │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐    │
│  │                                        1. CLIENT LAYER                                               │    │
│  │                                                                                                      │    │
│  │   iOS / Android / Web App                                                                            │    │
│  │       │                                                                                              │    │
│  │       └──► KMP Business Logic ──► gRPC Request                                                      │    │
│  │                                    Discovery.GetHomepageFeed(GetHomepageFeedRequest)                │    │
│  └─────────────────────────────────────────────────────────────────────────────────────────────────────┘    │
│                                                   │                                                          │
│                                                   ▼                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐    │
│  │                                     2. UNIFIED GATEWAY (EDGE)                                        │    │
│  │                                                                                                      │    │
│  │   • Authentication (OAuth/Session)                                                                   │    │
│  │   • Authorization                                                                                    │    │
│  │   • Rate limiting & throttling                                                                       │    │
│  │   • Request routing → feed.v2.service.discovery.Discovery/GetHomepageFeed                           │    │
│  └─────────────────────────────────────────────────────────────────────────────────────────────────────┘    │
│                                                   │                                                          │
│                                                   ▼                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐    │
│  │                                  3. HOMEPAGE NODE (GRAPH RUNNER)                                     │    │
│  │                                                                                                      │    │
│  │   Graph: feed-v2-discovery-get-homepage-feed                                                         │    │
│  │   Node: feed-surface-homepage-v2                                                                     │    │
│  │                                                                                                      │    │
│  │   ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │    │
│  │   │  HomepageV2(ctx, GetHomepageFeedRequest) → Future[GetHomepageFeedResponse]                   │  │    │
│  │   │                                                                                               │  │    │
│  │   │  1. Create SurfaceScope & FeedScope                                                          │  │    │
│  │   │  2. Get App Config (DoorDash or Wolt)                                                        │  │    │
│  │   │  3. Register Products & Evidence Fetchers                                                    │  │    │
│  │   │  4. Call feedimpl.Fetch() ──────────────────────────────────────────────────────────────────┼──┼────┐
│  │   │  5. Return GetHomepageFeedResponse                                                           │  │    │
│  │   └──────────────────────────────────────────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                                              │
│                                                   ◄────────────────────────────────────────────────────────┼─┘
│                                                   │                                                          │
│                                                   ▼                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐    │
│  │                                 4. FEED ORCHESTRATION PIPELINE                                       │    │
│  │                                                                                                      │    │
│  │   FETCH ──► HYDRATE ──► BLEND ──► POST-PROCESS ──► POST-RANK HYDRATE ──► SERIALIZE                  │    │
│  │     │          │           │            │                  │                  │                      │    │
│  │     ▼          ▼           ▼            ▼                  ▼                  ▼                      │    │
│  │  Products   Members    Blender      Ads Pos.         Text/Media          FeedData                   │    │
│  │  (Parallel) (Collections)           Updates          Evidence            Proto                       │    │
│  └─────────────────────────────────────────────────────────────────────────────────────────────────────┘    │
│                                                   │                                                          │
│                                                   ▼                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐    │
│  │                                        5. DATA SOURCES                                               │    │
│  │                                                                                                      │    │
│  │   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                    │    │
│  │   │  Argo Broker   │  │ First Pass     │  │      MDH       │  │  Evidence      │                    │    │
│  │   │  (Store Search)│  │ Ranker (FPR)   │  │ (Store Data)   │  │  Services      │                    │    │
│  │   └────────────────┘  └────────────────┘  └────────────────┘  └────────────────┘                    │    │
│  └─────────────────────────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                                              │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Client Layer

### Request Format

The client sends a `GetHomepageFeedRequest` via gRPC:

```protobuf
message GetHomepageFeedRequest {
  // Consumer context with ID and location
  ConsumerContext consumer_context = 1;
  
  // App context (DoorDash or Wolt)
  AppContext app_context = 2;
  
  // Optional filters
  FeedFilters filters = 3;
}

message ConsumerContext {
  string consumer_id = 1;
  Location location = 2;  // lat, lng
}

message AppContext {
  AppName app_name = 1;  // DOORDASH | WOLT
}
```

### Client Platforms

| Platform | Technology | Request Path |
|----------|------------|--------------|
| iOS | Swift + KMP | KMP → gRPC Stub → Unified Gateway |
| Android | Kotlin + KMP | KMP → gRPC Stub → Unified Gateway |
| Web | TypeScript | HTTP/2 → gRPC-Web → Unified Gateway |

---

## 2. Gateway Layer

The Unified Gateway handles edge concerns before routing to the Homepage node:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           UNIFIED GATEWAY                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. AUTHENTICATION                                                           │
│     └── OAuth token validation                                               │
│     └── Session verification                                                 │
│     └── Guest user handling                                                  │
│                                                                              │
│  2. AUTHORIZATION                                                            │
│     └── Consumer permissions check                                           │
│     └── Feature flag evaluation                                              │
│                                                                              │
│  3. RATE LIMITING                                                            │
│     └── Per-consumer rate limits                                             │
│     └── Global rate limits                                                   │
│                                                                              │
│  4. REQUEST ROUTING                                                          │
│     └── Service discovery                                                    │
│     └── Load balancing                                                       │
│     └── Routes to: feed-v2-discovery-get-homepage-feed.workloads.dash       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Homepage Node Architecture

### Node Structure

```
surface/homepage/
├── node.go                          # Main entry point
├── app/
│   ├── config.go                    # Config interface
│   ├── doordash.go                  # DoorDash config
│   └── wolt.go                      # Wolt config
├── orchestration/
│   ├── surface_scope.go             # Surface scope
│   ├── feed_blender.go              # Feed blending
│   └── media/
│       └── store_media_blender.go   # Media blending
├── productoverrides/
│   ├── restaurants/
│   │   ├── storeretrieve/config.go  # Store retrieval
│   │   └── storehydrate/config.go   # Store hydration
│   └── newverticals/
│       └── storeretrieve/config.go  # NV store retrieval
├── internal/
│   ├── collection/
│   │   └── member_blender_factory.go
│   └── dv/
│       └── dv.go                    # Feature flags
└── wolt/                            # Wolt-specific overrides
```

### Node Dependencies (Injected via Graph Runner DI)

```go
type node struct {
    _            gr.Node[HomepageV2] `gr:"name=feed-surface-homepage-v2"`
    
    // Feature flag evaluation
    dvEvaluator  dv.DVEvaluator
    
    // Shared data repository
    globalRepo   globalrepository.GlobalRepository
    
    // Ads caching
    adsRetriever ads.CachedAdRepository
    
    // App-specific configs
    doordashConfig app.DoordashProvider
    woltConfig     app.WoltProvider
}
```

### Node Initialization Flow

```go
func (i *node) HomepageV2(ctx context.Context, req *GetHomepageFeedRequest) Future[*GetHomepageFeedResponse] {
    // 1. Enrich context with locale
    ctx, _ = scope.EnrichContextWithLocale(ctx, req)
    
    // 2. Create surface scope
    surfaceScope := orchestration.NewHomepageSurfaceScope(ctx, req)
    
    // 3. Get app-specific config (DoorDash or Wolt)
    config := i.getAppConfig(surfaceScope.AppName())
    
    // 4. Add observability dimensions
    observability.AddDimensions(ctx, "is_guest", false, "app_name", appName)
    
    // 5. Setup global repository and ads cache
    ctx = globalrepository.WithGlobalRepository(ctx, i.globalRepo)
    ctx = i.adsRetriever.RegisterCache(ctx)
    
    // 6. Register all products and evidence fetchers
    ctx, _ = orchestrationFramework.RegisterProducts(ctx, 
        config.FeedFetchers, 
        config.MediaEvidenceFetchers, 
        config.TextEvidenceFetchers, 
        config.Overrides)
    
    // 7. Register media blender (if configured)
    if config.StoreMediaBlender != nil {
        mediaBlenderRegistry.RegisterBlender(STORE, RESTAURANTS, config.StoreMediaBlender)
    }
    
    // 8. Create feed scope
    feedScope := scope.NewFeedScope(ctx, consumer, FeedType_HomePageFeed, surfaceScope)
    
    // 9. Execute feed pipeline
    feedDataFuture := feedimpl.Fetch(ctx, feedScope, FeedType_HomePageFeed, config.Blender)
    
    // 10. Map to response
    return future.MapValue(ctx, feedDataFuture, func(feedData) (*GetHomepageFeedResponse, nil))
}
```

---

## 4. Feed Orchestration Pipeline

The feed pipeline is the core of the Homepage node, orchestrating data fetching, processing, and serialization.

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    FEED ORCHESTRATION PIPELINE                                               │
│                                   (platform/pkg/feed/impl/feed.go)                                          │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                              │
│  feedimpl.Fetch(ctx, feedScope, feedType, blender)                                                          │
│                                                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  STEP 1: FETCH                   fetchFromAllProductsAndDedupe()                                │  │  │
│  │  │                                                                                                  │  │  │
│  │  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐│  │  │
│  │  │  │ Restaurants │ │    Ads      │ │NewVerticals │ │   Growth    │ │SmartSuggest │ │  Incentive  ││  │  │
│  │  │  │   Fetcher   │ │   Fetcher   │ │   Fetcher   │ │   Fetcher   │ │   Fetcher   │ │   Fetcher   ││  │  │
│  │  │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘│  │  │
│  │  │         │               │               │               │               │               │       │  │  │
│  │  │         └───────────────┴───────────────┴───────────────┴───────────────┴───────────────┘       │  │  │
│  │  │                                         │                                                        │  │  │
│  │  │                              PARALLEL EXECUTION                                                  │  │  │
│  │  │                         (future.ScatterKeyed + GatherKeyed)                                     │  │  │
│  │  │                                         │                                                        │  │  │
│  │  │                                         ▼                                                        │  │  │
│  │  │                          map[productName][]FeedUnit                                              │  │  │
│  │  │                                         │                                                        │  │  │
│  │  │                                    Deduplication                                                 │  │  │
│  │  │                          (Same entity across products → single instance)                        │  │  │
│  │  └─────────────────────────────────────────┼────────────────────────────────────────────────────────┘  │  │
│  │                                            │                                                           │  │
│  │                                            ▼                                                           │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  STEP 2: HYDRATE (Pre-Blend)         hydrateUnits()                                             │  │  │
│  │  │                                                                                                  │  │  │
│  │  │  For each Collection:                                                                            │  │  │
│  │  │    ┌────────────────────────────────────────────────────────────────────────────────────────┐   │  │  │
│  │  │    │  collection.FetchMembers(ctx, collectionScope)                                         │   │  │  │
│  │  │    │    → Retrieve items/stores for carousel from data source                               │   │  │  │
│  │  │    │                                                                                        │   │  │  │
│  │  │    │  collection.BlendMembers(ctx, collectionScope)                                         │   │  │  │
│  │  │    │    → Rank and order items within the collection                                        │   │  │  │
│  │  │    └────────────────────────────────────────────────────────────────────────────────────────┘   │  │  │
│  │  └─────────────────────────────────────────┼────────────────────────────────────────────────────────┘  │  │
│  │                                            │                                                           │  │
│  │                                            ▼                                                           │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  STEP 3: BLEND                       HomePageFeedBlender.Blend()                                │  │  │
│  │  │                                                                                                  │  │  │
│  │  │  Blending Strategy:                                                                              │  │  │
│  │  │    ┌──────────────────────────────────────────────────────────────────────────────────────────┐ │  │  │
│  │  │    │  1. TOP NAVIGATION UNITS                                                                 │ │  │  │
│  │  │    │     ├── Vertical Filter (id: "nv_vertical_filter")                                      │ │  │  │
│  │  │    │     └── Smart Suggestions                                                                │ │  │  │
│  │  │    │                                                                                          │ │  │  │
│  │  │    │  2. COLLECTIONS (Carousels)                                                              │ │  │  │
│  │  │    │     ├── Restaurants collections                                                          │ │  │  │
│  │  │    │     ├── Ads collections (Sponsored)                                                      │ │  │  │
│  │  │    │     ├── Incentive collections                                                            │ │  │  │
│  │  │    │     ├── Growth collections                                                               │ │  │  │
│  │  │    │     └── NewVerticals collections                                                         │ │  │  │
│  │  │    │                                                                                          │ │  │  │
│  │  │    │  3. NON-RANKABLE UNITS                                                                   │ │  │  │
│  │  │    │                                                                                          │ │  │  │
│  │  │    │  4. ENTITIES (Standalone Stores)                                                         │ │  │  │
│  │  │    │     └── Store entities in product order                                                  │ │  │  │
│  │  │    └──────────────────────────────────────────────────────────────────────────────────────────┘ │  │  │
│  │  └─────────────────────────────────────────┼────────────────────────────────────────────────────────┘  │  │
│  │                                            │                                                           │  │
│  │                                            ▼                                                           │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  STEP 4: POST-PROCESS                postProcessFeed()                                          │  │  │
│  │  │                                                                                                  │  │  │
│  │  │  For each Collection:                                                                            │  │  │
│  │  │    └── collection.PostProcessMembers(ctx, collectionScope, blendedFeed)                         │  │  │
│  │  │        └── Ads position updates based on vertical position in feed                              │  │  │
│  │  └─────────────────────────────────────────┼────────────────────────────────────────────────────────┘  │  │
│  │                                            │                                                           │  │
│  │                                            ▼                                                           │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  STEP 5: POST-RANK HYDRATE           postRankHydrate()  ◄─── MAIN HYDRATION STEP               │  │  │
│  │  │                                                                                                  │  │  │
│  │  │  For each FeedUnit (PARALLEL via future.Scatter):                                                │  │  │
│  │  │    ┌────────────────────────────────────────────────────────────────────────────────────────┐   │  │  │
│  │  │    │  unit.PostRankHydrate(ctx, feedScope)                                                  │   │  │  │
│  │  │    │                                                                                        │   │  │  │
│  │  │    │  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐         │   │  │  │
│  │  │    │  │   text.FetchAll()    │  │   media.FetchAll()   │  │  mdh.GetStores()     │         │   │  │  │
│  │  │    │  │                      │  │                      │  │                      │         │   │  │  │
│  │  │    │  │  • ETA ("15-25 min") │  │  • Store cover image │  │  • Business info     │         │   │  │  │
│  │  │    │  │  • Distance ("1.2mi")│  │  • Item thumbnails   │  │  • Operational status│         │   │  │  │
│  │  │    │  │  • Fee ("$0 fee")    │  │  • Hero images       │  │  • Menu availability │         │   │  │  │
│  │  │    │  │  • Ratings ("4.8")   │  │                      │  │                      │         │   │  │  │
│  │  │    │  │  • Merchant text     │  │                      │  │                      │         │   │  │  │
│  │  │    │  │  • Social ("Friend") │  │                      │  │                      │         │   │  │  │
│  │  │    │  │  • Ads ("Sponsored") │  │                      │  │                      │         │   │  │  │
│  │  │    │  │  • Incentive ("$5")  │  │                      │  │                      │         │   │  │  │
│  │  │    │  └──────────────────────┘  └──────────────────────┘  └──────────────────────┘         │   │  │  │
│  │  │    └────────────────────────────────────────────────────────────────────────────────────────┘   │  │  │
│  │  └─────────────────────────────────────────┼────────────────────────────────────────────────────────┘  │  │
│  │                                            │                                                           │  │
│  │                                            ▼                                                           │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  STEP 6: SERIALIZE                   convertFeedUnitsToProto()                                  │  │  │
│  │  │                                                                                                  │  │  │
│  │  │  For each FeedUnit:                                                                              │  │  │
│  │  │    └── unit.ConvertToFeedUnit(ctx, feedScope) → FeedUnitData proto                              │  │  │
│  │  │                                                                                                  │  │  │
│  │  │  Output: FeedData { feed_units: []FeedUnitData }                                                │  │  │
│  │  └─────────────────────────────────────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                                              │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Data Sources

### Store Retrieval: Argo Broker

The primary data source for store discovery is the Argo Search Broker.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            ARGO BROKER                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Authority: argo-search-discovery-broker.service.prod.ddsd                   │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  BrokerRequest                                                          │ │
│  │                                                                         │ │
│  │  {                                                                      │ │
│  │    namespace: "discovery",                                              │ │
│  │    experience: "doordash" | "wolt",                                     │ │
│  │    pipeline: "store_discovery_pipeline",                                │ │
│  │    size: 1500,                                                          │ │
│  │    return_fields: [                                                     │ │
│  │      "business_id",                                                     │ │
│  │      "name",                                                            │ │
│  │      "store_id",                                                        │ │
│  │      "tier_level",                                                      │ │
│  │      "price_range",                                                     │ │
│  │      "starting_point",                                                  │ │
│  │      "is_consumer_subscription_eligible",                               │ │
│  │      "is_partner",                                                      │ │
│  │      "location",                                                        │ │
│  │      "brand_business_group_id",                                         │ │
│  │      "vertical_ids",                                                    │ │
│  │      "delivery_radius_circle",                                          │ │
│  │      "primary_vertical_ids",                                            │ │
│  │      "business_vertical_id",                                            │ │
│  │      "catering",                                                        │ │
│  │      "is_snap_eligible",                                                │ │
│  │      "is_hsa_fsa_eligible",                                             │ │
│  │      "hide_from_homepage_list",                                         │ │
│  │      "avg_subtotal_per_person",                                         │ │
│  │      "saf_st_price_inflation_rate"                                      │ │
│  │    ]                                                                    │ │
│  │  }                                                                      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  BrokerResponse                                                         │ │
│  │                                                                         │ │
│  │  SearchResults {                                                        │ │
│  │    Documents: []Document                                                │ │
│  │  }                                                                      │ │
│  │                                                                         │ │
│  │  → Mapped to []StoreMetadata                                            │ │
│  │  → Deduplicated by business (closest store per brand)                   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### First Pass Ranker (FPR)

Personalization layer for store ordering:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FIRST PASS RANKER (FPR)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Input:                                                                      │
│    • consumer_id: Consumer identifier                                        │
│    • geohash: H3 geohash (resolution 7) from lat/lng                        │
│    • model: "COPURCHASE"                                                     │
│                                                                              │
│  Output:                                                                     │
│    • store_ids: Personalized list of store IDs                              │
│                                                                              │
│  Purpose:                                                                    │
│    Provides personalized store ordering based on consumer purchase history   │
│    and co-purchase patterns.                                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Evidence Fetchers

| Category | Fetcher | Data Provided | Source |
|----------|---------|---------------|--------|
| **Text** | ETA | "15-25 min" | ETA service |
| **Text** | Distance | "1.2 mi away" | Geo calculation |
| **Text** | Fee | "$0 delivery fee" | Pricing service |
| **Text** | Ratings (Store) | "4.8 (2.3K ratings)" | Ratings service |
| **Text** | Ratings (Item) | "4.5 stars" | Ratings service |
| **Text** | Merchant | Item descriptions | MDH |
| **Text** | Social | "Your friend ordered here" | Social graph |
| **Text** | Ads | "Sponsored" | Ads service |
| **Text** | Incentive | "$5 off your order" | Campaign service |
| **Media** | MerchantMedia | Store cover images | MDH |
| **Media** | RestaurantsMedia | Store/item photos | Asset service |

### MDH (Menu Data Hub)

Store hydration with business data:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MDH DATA                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  mdhDataCache.GetStores(ctx, storeIds, nil)                                  │
│                                                                              │
│  Returns:                                                                    │
│    • Business information                                                    │
│    • Operational status                                                      │
│    • Menu availability                                                       │
│    • Pricing information                                                     │
│    • Store metadata                                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Response Structure

### Proto Definition

```protobuf
message GetHomepageFeedResponse {
  FeedData feed_data = 1;
}

message FeedData {
  repeated FeedUnitData feed_units = 1;
}

message FeedUnitData {
  oneof unit {
    CollectionData collection = 1;
    EntityData entity = 2;
    PlacementData placement = 3;
  }
}
```

### Response Layout

```
GetHomepageFeedResponse {
  feed_data: FeedData {
    feed_units: [
    
      ┌─────────────────────────────────────────────────────────────────────┐
      │  1. VERTICAL FILTER (Navigation - Position 0)                       │
      │                                                                      │
      │  CollectionData {                                                    │
      │    id: "nv_vertical_filter"                                          │
      │    members: [                                                        │
      │      { type: VERTICAL_TAB, id: "grocery", title: "Grocery" }        │
      │      { type: VERTICAL_TAB, id: "convenience", title: "Convenience" }│
      │      { type: VERTICAL_TAB, id: "retail", title: "Retail" }          │
      │      { type: VERTICAL_TAB, id: "alcohol", title: "Alcohol" }        │
      │    ]                                                                 │
      │  }                                                                   │
      └─────────────────────────────────────────────────────────────────────┘
      
      ┌─────────────────────────────────────────────────────────────────────┐
      │  2. SMART SUGGESTIONS (Navigation - Position 1)                     │
      │                                                                      │
      │  CollectionData {                                                    │
      │    id: "smart_suggestions"                                           │
      │    members: [                                                        │
      │      { type: SUGGESTION, id: "recent", title: "Recent Orders" }     │
      │      { type: SUGGESTION, id: "favorites", title: "Favorites" }      │
      │    ]                                                                 │
      │  }                                                                   │
      └─────────────────────────────────────────────────────────────────────┘
      
      ┌─────────────────────────────────────────────────────────────────────┐
      │  3. CAROUSELS (Collections)                                         │
      │                                                                      │
      │  CollectionData {                                                    │
      │    id: "top_picks"                                                   │
      │    title: "Top Picks for You"                                        │
      │    members: [                                                        │
      │      EntityData {                                                    │
      │        type: STORE                                                   │
      │        id: "store_123"                                               │
      │        metadata: { name: "Pizza Palace", ... }                       │
      │        text_evidences: [                                             │
      │          { type: ETA, value: "15-25 min" }                           │
      │          { type: DISTANCE, value: "0.8 mi" }                         │
      │          { type: RATING, value: "4.8 (2.1K)" }                       │
      │          { type: FEE, value: "$0 delivery fee" }                     │
      │        ]                                                             │
      │        media_evidences: [                                            │
      │          { type: COVER_IMAGE, url: "https://..." }                   │
      │        ]                                                             │
      │      }                                                               │
      │      EntityData { ... }                                              │
      │    ]                                                                 │
      │  }                                                                   │
      │                                                                      │
      │  CollectionData { id: "free_delivery", title: "Free Delivery", ... }│
      │  CollectionData { id: "sponsored", title: "Sponsored", ... }        │
      │  CollectionData { id: "new_on_doordash", title: "New on DD", ... }  │
      └─────────────────────────────────────────────────────────────────────┘
      
      ┌─────────────────────────────────────────────────────────────────────┐
      │  4. STANDALONE ENTITIES (Stores)                                    │
      │                                                                      │
      │  EntityData {                                                        │
      │    type: STORE                                                       │
      │    id: "store_456"                                                   │
      │    metadata: { name: "Burger Joint", price_range: "$$", ... }       │
      │    text_evidences: [                                                 │
      │      { type: ETA, value: "20-30 min" }                               │
      │      { type: DISTANCE, value: "1.2 mi" }                             │
      │      { type: RATING, value: "4.5 (1.8K)" }                           │
      │    ]                                                                 │
      │    media_evidences: [                                                │
      │      { type: COVER_IMAGE, url: "https://..." }                       │
      │    ]                                                                 │
      │  }                                                                   │
      │                                                                      │
      │  EntityData { ... }                                                  │
      │  EntityData { ... }                                                  │
      └─────────────────────────────────────────────────────────────────────┘
      
    ]
  }
}
```

---

## 7. App Configuration

### DoorDash Configuration

| Component | Items |
|-----------|-------|
| **Feed Fetchers (6)** | restaurants, ads, newverticals, growth, smartsuggestion, incentive |
| **Text Evidence (9)** | distance, eta, fee, ratings (item), ratings (store), merchant, social, ads, incentive |
| **Media Evidence (2)** | merchantMediaStore, restaurantsMediaStore |
| **Overrides (3)** | rxStoreRetriever, nvStoreRetriever, rxStoreHydrate |
| **Blender** | HomePageFeedBlender |
| **Media Blender** | StoreMediaBlender |

### Wolt Configuration

| Component | Items |
|-----------|-------|
| **Feed Fetchers (4)** | restaurants, ads, newverticals, growth |
| **Text Evidence (8)** | distance, eta, fee, ratings (item), ratings (store), merchant, social, ads |
| **Media Evidence (2)** | merchantMediaStore, restaurantsMediaStore |
| **Overrides (3)** | rxStoreRetriever (Wolt), nvStoreRetriever (Wolt), rxStoreHydrate |
| **Blender** | WoltFeedBlender |

### Configuration Selection

```go
func (i *node) getAppConfig(appName requestpb.AppName) app.Config {
    switch appName {
    case requestpb.AppName_FEED_APP_NAME_WOLT:
        return i.woltConfig.GetConfig()
    default:
        return i.doordashConfig.GetConfig()
    }
}
```

---

## 8. Key Design Decisions

### 1. App-Specific Configuration

**Decision:** Separate configurations for DoorDash and Wolt apps.

**Rationale:**
- Different product availability (e.g., Incentive only on DoorDash)
- Different store retrieval strategies per market
- Different blending algorithms

### 2. Parallel Product Fetching

**Decision:** All product fetchers execute in parallel using `future.ScatterKeyed` and `future.GatherKeyed`.

**Rationale:**
- Minimizes total latency
- Products are independent data sources
- Graceful degradation if one product fails

### 3. Two-Phase Hydration

**Decision:** Pre-blend hydration (FetchMembers/BlendMembers) and post-rank hydration (evidence fetching).

**Rationale:**
- Collection members must be fetched before blending
- Evidence is expensive, only fetch for final ranked items
- Parallel evidence fetching for all final units

### 4. Blending Strategy

**Decision:** Fixed ordering: Navigation → Collections → Non-rankable → Entities.

**Rationale:**
- Navigation units (vertical filter) must be at top
- Collections (carousels) provide visual hierarchy
- Entities fill remaining space

### 5. Result Caching

**Decision:** Store retrieval results are cached per (consumer_id, location) key.

**Rationale:**
- Multiple products may request same stores
- Avoids duplicate Argo calls
- Reduces latency and cost

### 6. Graceful Degradation

**Decision:** Pipeline continues even if individual products or evidence fetchers fail.

**Rationale:**
- Better to show partial results than error
- `GatherKeyed(AtLeastOne())` allows partial success
- Failures logged but don't block response

---

## Related Documentation

- [Feed Platform Directory Structure](../reference/dir-structure.md)
- [Feed Terminology](../reference/terminology.md)
- [Hydration Guide](./hydration.md)
- [FDM Documentation](../reference/fdm/0-intros.md)
- [Homepage README](../../surface/homepage/README.md)

