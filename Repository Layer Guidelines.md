# Repository Layer Guidelines

## Overview

The repository layer handles data persistence and retrieval operations. This document outlines the patterns and best practices for implementing repositories in the Product Search project.

## Repository Interface Design

### 1. Interface Definition

Each repository should have a clear interface:

```go
type SearchRepository interface {
    // Read operations
    Search(ctx context.Context, params SearchParams) ([]Product, error)
    GetByID(ctx context.Context, id string) (*Product, error)
    
    // Write operations
    Create(ctx context.Context, product *Product) error
    Update(ctx context.Context, product *Product) error
    Delete(ctx context.Context, id string) error
    
    // Bulk operations
    BulkInsert(ctx context.Context, products []Product) error
    BulkUpdate(ctx context.Context, products []Product) error
}
```

### 2. Implementation Structure

```go
type searchRepository struct {
    db        *elasticsearch.Client
    logger    *zap.Logger
    indexName string
}

func NewSearchRepository(
    db *elasticsearch.Client,
    logger *zap.Logger,
    indexName string,
) SearchRepository {
    return &searchRepository{
        db:        db,
        logger:    logger,
        indexName: indexName,
    }
}
```

## Best Practices

### 1. Query Building

- Use query builders
- Implement pagination
- Handle sorting
- Apply filters

```go
func (r *searchRepository) buildSearchQuery(params SearchParams) map[string]interface{} {
    query := map[string]interface{}{
        "query": map[string]interface{}{
            "bool": map[string]interface{}{
                "must": []map[string]interface{}{
                    {
                        "multi_match": map[string]interface{}{
                            "query":  params.Query,
                            "fields": []string{"name^2", "description"},
                        },
                    },
                },
            },
        },
        "from": (params.Page - 1) * params.PageSize,
        "size": params.PageSize,
    }
    
    if params.SortBy != "" {
        query["sort"] = []map[string]interface{}{
            {
                params.SortBy: map[string]interface{}{
                    "order": params.SortOrder,
                },
            },
        }
    }
    
    return query
}
```

### 2. Error Handling

- Use domain-specific errors
- Handle database-specific errors
- Provide meaningful error messages

```go
func (r *searchRepository) handleError(err error, op string) error {
    if elasticsearch.IsNotFound(err) {
        return &NotFoundError{
            Message: "product not found",
            Cause:   err,
        }
    }
    
    if elasticsearch.IsTimeout(err) {
        return &TimeoutError{
            Message: "operation timed out",
            Cause:   err,
        }
    }
    
    return &RepositoryError{
        Message: fmt.Sprintf("error during %s operation", op),
        Cause:   err,
    }
}
```

### 3. Bulk Operations

- Use batching
- Implement retries
- Handle partial failures
- Track progress

```go
func (r *searchRepository) BulkInsert(ctx context.Context, products []Product) error {
    const batchSize = 1000
    
    for i := 0; i < len(products); i += batchSize {
        end := i + batchSize
        if end > len(products) {
            end = len(products)
        }
        
        batch := products[i:end]
        if err := r.insertBatch(ctx, batch); err != nil {
            return fmt.Errorf("error inserting batch %d-%d: %w", i, end, err)
        }
    }
    
    return nil
}

func (r *searchRepository) insertBatch(ctx context.Context, batch []Product) error {
    bulk := r.db.Bulk()
    for _, product := range batch {
        req := elasticsearch.NewBulkIndexRequest().
            Index(r.indexName).
            Id(product.ID).
            Doc(product)
        bulk.Add(req)
    }
    
    res, err := bulk.Do(ctx)
    if err != nil {
        return err
    }
    
    if res.Errors {
        // Handle failed items
        return r.handleBulkErrors(res)
    }
    
    return nil
}
```

### 4. Monitoring

- Log operations
- Track metrics
- Monitor performance
- Set up alerts

```go
func (r *searchRepository) trackMetrics(start time.Time, op string, err error) {
    duration := time.Since(start)
    r.metrics.Timing(fmt.Sprintf("repository.%s.duration", op), duration)
    if err != nil {
        r.metrics.Increment(fmt.Sprintf("repository.%s.error", op))
    }
}
```