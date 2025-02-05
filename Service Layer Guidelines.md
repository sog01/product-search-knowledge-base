# Service Layer Guidelines

## Overview

The service layer implements business logic and orchestrates operations between different components. This document outlines the best practices and patterns observed in the Product Search project.

## Service Structure

### 1. Query Operations

Query operations should be organized in the `query` package:

```go
// search_catalogs.go
type SearchCatalogsQuery struct {
    repo       repository.SearchCatalogRepository
    logger     *zap.Logger
}

func NewSearchCatalogsQuery(
    repo repository.SearchCatalogRepository,
    logger *zap.Logger,
) *SearchCatalogsQuery {
    return &SearchCatalogsQuery{
        repo:   repo,
        logger: logger,
    }
}

func (q *SearchCatalogsQuery) Execute(ctx context.Context, dto *model.SearchCatalogsDTO) (*model.SearchResult, error) {
    // Implementation
}
```

### 2. Mutation Operations

Mutation operations should be in the `mutation` package:

```go
// bulk_insert.go
type BulkInsertMutation struct {
    repo       repository.BulkInsertRepository
    logger     *zap.Logger
}

func NewBulkInsertMutation(
    repo repository.BulkInsertRepository,
    logger *zap.Logger,
) *BulkInsertMutation {
    return &BulkInsertMutation{
        repo:   repo,
        logger: logger,
    }
}

func (m *BulkInsertMutation) Execute(ctx context.Context, dto *model.BulkInsertDTO) error {
    // Implementation
}
```

## Best Practices

### 1. Error Handling

- Use domain-specific errors
- Wrap lower-level errors
- Log errors with context
- Return clean error messages

```go
type DomainError struct {
    Code    string
    Message string
    Cause   error
}

func (s *SearchService) handleError(err error, op string) error {
    return &DomainError{
        Code:    "SEARCH_ERROR",
        Message: fmt.Sprintf("error during %s operation", op),
        Cause:   err,
    }
}
```

### 2. Validation

- Validate input DTOs
- Check business rules
- Return early on validation errors

```go
func (s *SearchService) validateInput(dto *SearchDTO) error {
    if dto == nil {
        return errors.New("dto cannot be nil")
    }
    if err := dto.Validate(); err != nil {
        return fmt.Errorf("invalid dto: %w", err)
    }
    return nil
}
```

### 3. Logging

- Log operation start/end
- Include relevant context
- Use appropriate log levels
- Structure log messages

```go
func (s *SearchService) Search(ctx context.Context, dto *SearchDTO) (*SearchResult, error) {
    s.logger.Info("starting search operation",
        zap.String("query", dto.Query),
        zap.Int("page", dto.Page))
    
    // Operation logic
    
    s.logger.Info("completed search operation",
        zap.Int("results", len(result.Items)))
    return result, nil
}
```

### 4. Metrics

- Track operation duration
- Count success/failure
- Monitor resource usage
- Set up alerts

```go
func (s *SearchService) trackMetrics(start time.Time, op string, err error) {
    duration := time.Since(start)
    s.metrics.Timing(fmt.Sprintf("search.%s.duration", op), duration)
    if err != nil {
        s.metrics.Increment(fmt.Sprintf("search.%s.error", op))
    }
}
```