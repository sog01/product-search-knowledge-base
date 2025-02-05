# Project Structure Guidelines

## Directory Structure

```
Product-Search/
├── cmd/                    # Application entry points
│   └── app/               # Main application
├── deployments/           # Deployment configurations
│   ├── Dockerfile
│   └── docker-compose.yaml
├── docs/                  # API documentation
│   ├── swagger.json
│   └── swagger.yaml
├── indices/              # Search indices implementation
├── internal/             # Private application code
│   ├── search/          # Search feature
│   ├── shortener/       # URL shortener feature
│   └── web/            # Web handlers
├── pkg/                 # Public shared libraries
└── web/                # Web assets and templates
```

## Package Guidelines

### 1. Feature Package Structure

Each feature package should follow this structure:
```
feature/
├── model/          # Data structures and DTOs
├── repository/     # Data access layer
└── service/        # Business logic
    ├── mutation/   # Write operations
    └── query/      # Read operations
```

### 2. Model Layer

- Use clear, descriptive names for DTOs
- Include JSON struct tags
- Implement validation methods
- Keep models focused and single-purpose

Example:
```go
type SearchDTO struct {
    Query     string `json:"query"`
    Page      int    `json:"page"`
    PageSize  int    `json:"pageSize"`
    SortBy    string `json:"sortBy"`
    SortOrder string `json:"sortOrder"`
}

func (dto *SearchDTO) Validate() error {
    if dto.Query == "" {
        return errors.New("query cannot be empty")
    }
    if dto.Page < 1 {
        return errors.New("page must be greater than 0")
    }
    return nil
}
```

### 3. Repository Layer

- Interface-based design
- Clear method signatures
- Error handling
- Transaction support

Example:
```go
type SearchRepository interface {
    Search(ctx context.Context, params SearchDTO) ([]Product, error)
    GetByID(ctx context.Context, id string) (*Product, error)
    Create(ctx context.Context, product *Product) error
}
```

### 4. Service Layer

- Business logic implementation
- Dependency injection
- Error handling
- Logging and metrics

Example:
```go
type SearchService struct {
    repo    SearchRepository
    logger  *zap.Logger
    metrics MetricsClient
}

func NewSearchService(repo SearchRepository, logger *zap.Logger, metrics MetricsClient) *SearchService {
    return &SearchService{
        repo:    repo,
        logger:  logger,
        metrics: metrics,
    }
}
```