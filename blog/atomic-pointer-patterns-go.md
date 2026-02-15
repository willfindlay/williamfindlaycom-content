---
title: "Atomic Pointer Patterns in Go"
date: "2025-11-15T09:00:00Z"
description: "Lock-free read patterns using atomic.Pointer for concurrent data structures in Go."
tags: ["go", "concurrency"]
---

Go 1.19 introduced `atomic.Pointer[T]`, a typed wrapper around `unsafe.Pointer` atomics. It's one of the cleanest ways to implement lock-free reads in concurrent Go programs.

## The Problem

You have a data structure that's read far more often than it's written. A `sync.RWMutex` works, but readers still contend on the mutex itself — calling `RLock()` and `RUnlock()` involves atomic operations on shared state.

For read-heavy workloads, we can do better.

## The Pattern

Build the entire data structure in isolation, then swap it in with a single atomic store:

```go
type Config struct {
    Features map[string]bool
    Limits   map[string]int
}

type ConfigStore struct {
    ptr atomic.Pointer[Config]
}

func (s *ConfigStore) Load() *Config {
    return s.ptr.Load()
}

func (s *ConfigStore) Update(cfg *Config) {
    s.ptr.Store(cfg)
}
```

Readers call `Load()` and get a consistent snapshot. No locks, no contention, no possibility of seeing a partially-updated structure.

## When to Use This

This pattern works when:

- **Reads vastly outnumber writes** (100:1 or more)
- **The data structure is immutable once built** — readers never modify what they get
- **Rebuilding is cheap enough** to do on every write
- **Consistency within a snapshot matters** — readers need all-or-nothing updates

It does *not* work when:

- Writes are frequent (you'd allocate too much garbage)
- The data structure is too large to rebuild
- You need read-modify-write semantics

## Real-World Example

A web server that loads configuration from a database every 30 seconds:

```go
func refreshLoop(ctx context.Context, store *ConfigStore, db *sql.DB) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            cfg, err := loadConfigFromDB(db)
            if err != nil {
                slog.Error("config refresh failed", "err", err)
                continue // keep serving stale config
            }
            store.Update(cfg)
        }
    }
}
```

Handlers read the config on every request without any synchronization overhead:

```go
func handler(store *ConfigStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        cfg := store.Load()
        if cfg.Features["new-ui"] {
            // serve new UI
        }
    }
}
```

The beauty is that a failed refresh doesn't break anything — the server keeps serving the last known good configuration until the next successful refresh.
