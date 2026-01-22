# ECS-Aware Cachedb - PR Summary

## Problem
ECS (EDNS Client Subnet) responses are not cached in the cachedb Redis backend. Subnetmod marks ECS responses as "no_cache_store" to prevent mixing subnets in the global cache, but this means cachedb doesn't store them. After subnetcache flushes, ECS queries must go upstream again.

## Solution
Include ECS data in cachedb cache keys so different subnets get different entries:
- Hash includes: address family (IPv4/IPv6), source prefix mask, and address bytes
- Different subnets = different cache keys = separate Redis entries
- ECS responses now persist across cache flushes and server restarts

## Changes

### File: `cachedb/cachedb.c`

1. **Added includes** (Lines 62-63)
   - `#include "edns-subnet/edns-subnet.h"`
   - `#include "edns-subnet/subnetmod.h"`

2. **Modified `calc_hash()` function** (Line 340)
   - Signature: `calc_hash(..., struct ecs_data* ecs)`
   - Added ECS hashing block (Lines 366-396):
     - Includes ECS address family (IPv4/IPv6)
     - Includes source prefix length (CIDR mask bits)
     - Includes address bytes (4 for IPv4, 16 for IPv6)
   - Guarded with `#ifdef CLIENT_SUBNET`

3. **Updated `cachedb_extcache_lookup()`** (Lines 687-703)
   - Extracts ECS from subnetmod state before cache lookup
   - Passes ECS to `calc_hash()` for ECS-aware key generation

4. **Updated `cachedb_extcache_store()`** (Lines 729-745)
   - Extracts ECS from subnetmod state before cache storage
   - Passes ECS to `calc_hash()` for ECS-aware key generation

5. **Updated `cachedb_handle_response()`** (Lines 958-988)
   - Intelligently allows caching of ECS responses despite subnetmod's no_cache_store flag
   - Maintains backward compatibility for non-ECS queries

6. **Updated `cachedb_msg_remove_qinfo()`** (Line 1129)
   - Passes NULL for ECS (backward compatible cleanup)

## Key Features

- **Backward Compatible**: All changes guarded with `#ifdef CLIENT_SUBNET`
- **Memory Safe**: All buffer operations bounds-checked
- **Thread Safe**: No new global state
- **Efficient**: Minimal overhead (single SHA256 hash of ~40 bytes)

## Testing

- All 33 unit tests pass
- New test: `testdata/cachedb_ecs_store.crpl` validates ECS caching
- Baseline tests unaffected (no regressions)
- Binary compiles cleanly (0 warnings)

## Configuration

Works with existing cachedb + subnetcache setup:
```
module-config: "subnetcache cachedb iterator"
cachedb:
    backend: "redis"
    redis-server-host: "localhost"
```

## Deployment

No configuration changes required. Feature automatically activates when:
- CLIENT_SUBNET is enabled at compile time
- subnetcache module is in module-config
- cachedb backend is configured
