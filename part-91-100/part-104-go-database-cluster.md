# Part 104: Go (Golang) กับ Database Cluster

## บทนำ

Go เป็นภาษาที่เหมาะสำหรับการสร้าง high-performance backend services ด้วย concurrency model ที่ง่ายต่อการใช้งานผ่าน goroutines และ channels ในบทนี้เราจะเรียนรู้การใช้ Go ร่วมกับ PostgreSQL (ผ่าน pgx v5), Redis (go-redis) และ MinIO/S3 (aws-sdk-go-v2) เพื่อสร้าง production-ready API

---

## สารบัญ

1. Project Setup
2. PostgreSQL ด้วย pgx v5
3. Redis ด้วย go-redis
4. S3/MinIO ด้วย aws-sdk-go-v2
5. HTTP Server ด้วย Echo
6. Complete API Example
7. Goroutines สำหรับ Parallel Queries
8. Error Handling และ Context

---

## 1. Project Setup

### 1.1 go.mod

```bash
# สร้าง project
mkdir myapp-go && cd myapp-go
go mod init github.com/myorg/myapp-go

# ติดตั้ง dependencies
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/pgxpool
go get github.com/redis/go-redis/v9
go get github.com/aws/aws-sdk-go-v2
go get github.com/aws/aws-sdk-go-v2/config
go get github.com/aws/aws-sdk-go-v2/service/s3
go get github.com/aws/aws-sdk-go-v2/credentials
go get github.com/labstack/echo/v4
go get github.com/labstack/echo/v4/middleware
go get github.com/golang-jwt/jwt/v5
go get golang.org/x/crypto
go get github.com/spf13/viper
go get go.uber.org/zap
go get github.com/google/uuid
go get github.com/go-playground/validator/v10
go get github.com/stretchr/testify
```

### 1.2 go.mod file

```go
module github.com/myorg/myapp-go

go 1.21

require (
    github.com/aws/aws-sdk-go-v2 v1.24.0
    github.com/aws/aws-sdk-go-v2/config v1.26.1
    github.com/aws/aws-sdk-go-v2/credentials v1.16.12
    github.com/aws/aws-sdk-go-v2/service/s3 v1.47.5
    github.com/go-playground/validator/v10 v10.16.0
    github.com/golang-jwt/jwt/v5 v5.2.0
    github.com/google/uuid v1.5.0
    github.com/jackc/pgx/v5 v5.5.1
    github.com/labstack/echo/v4 v4.11.4
    github.com/redis/go-redis/v9 v9.3.1
    github.com/spf13/viper v1.18.2
    github.com/stretchr/testify v1.8.4
    go.uber.org/zap v1.26.0
    golang.org/x/crypto v0.17.0
)
```

### 1.3 Project Structure

```
myapp-go/
├── cmd/
│   └── api/
│       └── main.go           # Application entry point
├── internal/
│   ├── config/
│   │   └── config.go         # Configuration
│   ├── database/
│   │   ├── postgres.go       # PostgreSQL connection
│   │   └── redis.go          # Redis connection
│   ├── handler/
│   │   ├── user.go           # User HTTP handlers
│   │   ├── product.go        # Product HTTP handlers
│   │   └── order.go          # Order HTTP handlers
│   ├── middleware/
│   │   ├── auth.go           # JWT middleware
│   │   ├── logger.go         # Request logging
│   │   └── ratelimit.go      # Rate limiting
│   ├── model/
│   │   ├── user.go           # User model
│   │   ├── product.go        # Product model
│   │   └── order.go          # Order model
│   ├── repository/
│   │   ├── user_repo.go      # User database operations
│   │   └── product_repo.go   # Product database operations
│   ├── service/
│   │   ├── user_service.go   # User business logic
│   │   ├── product_service.go # Product business logic
│   │   ├── cache_service.go  # Redis cache
│   │   └── storage_service.go # S3/MinIO storage
│   └── router/
│       └── router.go         # Route definitions
├── pkg/
│   ├── errors/
│   │   └── errors.go         # Custom error types
│   └── pagination/
│       └── pagination.go     # Pagination helpers
├── migrations/
│   ├── 001_create_users.sql
│   └── 002_create_products.sql
├── .env
├── .env.example
└── go.mod
```

---

## 2. Configuration

```go
// internal/config/config.go

package config

import (
    "fmt"
    "strings"

    "github.com/spf13/viper"
)

type Config struct {
    App      AppConfig
    Database DatabaseConfig
    Redis    RedisConfig
    S3       S3Config
    JWT      JWTConfig
}

type AppConfig struct {
    Name        string `mapstructure:"name"`
    Version     string `mapstructure:"version"`
    Environment string `mapstructure:"environment"`
    Host        string `mapstructure:"host"`
    Port        int    `mapstructure:"port"`
    Debug       bool   `mapstructure:"debug"`
}

type DatabaseConfig struct {
    Host         string `mapstructure:"host"`
    Port         int    `mapstructure:"port"`
    User         string `mapstructure:"user"`
    Password     string `mapstructure:"password"`
    Name         string `mapstructure:"name"`
    SSLMode      string `mapstructure:"ssl_mode"`
    MaxConns     int32  `mapstructure:"max_conns"`
    MinConns     int32  `mapstructure:"min_conns"`
    MaxLifetime  int    `mapstructure:"max_lifetime_minutes"`
    MaxIdleTime  int    `mapstructure:"max_idle_minutes"`
    
    // Read replicas
    ReplicaHosts []string `mapstructure:"replica_hosts"`
}

type RedisConfig struct {
    Host     string `mapstructure:"host"`
    Port     int    `mapstructure:"port"`
    Password string `mapstructure:"password"`
    DB       int    `mapstructure:"db"`
    
    // Cluster mode
    ClusterMode  bool     `mapstructure:"cluster_mode"`
    ClusterAddrs []string `mapstructure:"cluster_addrs"`
    
    MaxRetries      int `mapstructure:"max_retries"`
    PoolSize        int `mapstructure:"pool_size"`
    MinIdleConns    int `mapstructure:"min_idle_conns"`
    DialTimeout     int `mapstructure:"dial_timeout_seconds"`
    ReadTimeout     int `mapstructure:"read_timeout_seconds"`
    WriteTimeout    int `mapstructure:"write_timeout_seconds"`
}

type S3Config struct {
    Endpoint        string `mapstructure:"endpoint"`
    Region          string `mapstructure:"region"`
    AccessKeyID     string `mapstructure:"access_key_id"`
    SecretAccessKey string `mapstructure:"secret_access_key"`
    BucketName      string `mapstructure:"bucket_name"`
    UseSSL          bool   `mapstructure:"use_ssl"`
    PresignExpiry   int    `mapstructure:"presign_expiry_seconds"`
}

type JWTConfig struct {
    SecretKey              string `mapstructure:"secret_key"`
    AccessTokenExpiry      int    `mapstructure:"access_token_expiry_minutes"`
    RefreshTokenExpiry     int    `mapstructure:"refresh_token_expiry_days"`
}

func (d DatabaseConfig) DSN() string {
    return fmt.Sprintf(
        "host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
        d.Host, d.Port, d.User, d.Password, d.Name, d.SSLMode,
    )
}

func (r RedisConfig) Addr() string {
    return fmt.Sprintf("%s:%d", r.Host, r.Port)
}

var cfg *Config

func Load() (*Config, error) {
    v := viper.New()
    
    v.SetConfigName(".env")
    v.SetConfigType("env")
    v.AddConfigPath(".")
    
    v.AutomaticEnv()
    v.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
    
    // Defaults
    v.SetDefault("app.name", "MyApp API")
    v.SetDefault("app.version", "1.0.0")
    v.SetDefault("app.environment", "production")
    v.SetDefault("app.host", "0.0.0.0")
    v.SetDefault("app.port", 8080)
    
    v.SetDefault("database.ssl_mode", "require")
    v.SetDefault("database.max_conns", 25)
    v.SetDefault("database.min_conns", 5)
    v.SetDefault("database.max_lifetime_minutes", 30)
    v.SetDefault("database.max_idle_minutes", 5)
    
    v.SetDefault("redis.port", 6379)
    v.SetDefault("redis.db", 0)
    v.SetDefault("redis.pool_size", 50)
    v.SetDefault("redis.min_idle_conns", 10)
    v.SetDefault("redis.dial_timeout_seconds", 5)
    v.SetDefault("redis.read_timeout_seconds", 3)
    v.SetDefault("redis.write_timeout_seconds", 3)
    
    v.SetDefault("jwt.access_token_expiry_minutes", 30)
    v.SetDefault("jwt.refresh_token_expiry_days", 7)
    
    v.SetDefault("s3.region", "ap-southeast-1")
    v.SetDefault("s3.use_ssl", true)
    v.SetDefault("s3.presign_expiry_seconds", 3600)
    
    if err := v.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            return nil, fmt.Errorf("failed to read config: %w", err)
        }
    }
    
    cfg = &Config{}
    if err := v.Unmarshal(cfg); err != nil {
        return nil, fmt.Errorf("failed to unmarshal config: %w", err)
    }
    
    return cfg, nil
}

func Get() *Config {
    if cfg == nil {
        panic("config not loaded")
    }
    return cfg
}
```

---

## 3. PostgreSQL ด้วย pgx v5

### 3.1 Connection Pool Setup

```go
// internal/database/postgres.go

package database

import (
    "context"
    "fmt"
    "time"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
    "go.uber.org/zap"

    "github.com/myorg/myapp-go/internal/config"
)

var (
    primaryPool *pgxpool.Pool
    replicaPool *pgxpool.Pool
)

// InitPostgres เริ่มต้น connection pool
func InitPostgres(cfg *config.DatabaseConfig, logger *zap.Logger) error {
    var err error
    
    primaryPool, err = createPool(cfg.DSN(), cfg, logger, "primary")
    if err != nil {
        return fmt.Errorf("failed to create primary pool: %w", err)
    }
    
    // Create replica pool ถ้ามี replicas
    if len(cfg.ReplicaHosts) > 0 {
        replicaDSN := fmt.Sprintf(
            "host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
            cfg.ReplicaHosts[0], cfg.Port, cfg.User, cfg.Password, cfg.Name, cfg.SSLMode,
        )
        replicaPool, err = createPool(replicaDSN, cfg, logger, "replica")
        if err != nil {
            logger.Warn("Failed to create replica pool, using primary", zap.Error(err))
            replicaPool = primaryPool
        }
    } else {
        replicaPool = primaryPool
    }
    
    return nil
}

func createPool(dsn string, cfg *config.DatabaseConfig, logger *zap.Logger, role string) (*pgxpool.Pool, error) {
    poolConfig, err := pgxpool.ParseConfig(dsn)
    if err != nil {
        return nil, fmt.Errorf("failed to parse DSN: %w", err)
    }
    
    // Pool size
    poolConfig.MaxConns = cfg.MaxConns
    poolConfig.MinConns = cfg.MinConns
    poolConfig.MaxConnLifetime = time.Duration(cfg.MaxLifetime) * time.Minute
    poolConfig.MaxConnIdleTime = time.Duration(cfg.MaxIdleTime) * time.Minute
    poolConfig.HealthCheckPeriod = 30 * time.Second
    
    // Connection configuration
    poolConfig.ConnConfig.RuntimeParams = map[string]string{
        "application_name": fmt.Sprintf("myapp-go-%s", role),
    }
    
    // Logging callback
    poolConfig.BeforeConnect = func(ctx context.Context, cc *pgx.ConnConfig) error {
        logger.Debug("Creating new database connection", zap.String("role", role))
        return nil
    }
    
    poolConfig.AfterConnect = func(ctx context.Context, conn *pgx.Conn) error {
        logger.Debug("Database connection established", zap.String("role", role))
        return nil
    }
    
    // Create pool
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    pool, err := pgxpool.NewWithConfig(ctx, poolConfig)
    if err != nil {
        return nil, fmt.Errorf("failed to create pool: %w", err)
    }
    
    // Test connection
    if err := pool.Ping(ctx); err != nil {
        return nil, fmt.Errorf("failed to ping database: %w", err)
    }
    
    logger.Info("Database pool created",
        zap.String("role", role),
        zap.Int32("max_conns", cfg.MaxConns),
        zap.Int32("min_conns", cfg.MinConns),
    )
    
    return pool, nil
}

// GetPrimary returns primary pool (for writes)
func GetPrimary() *pgxpool.Pool {
    return primaryPool
}

// GetReplica returns replica pool (for reads)
func GetReplica() *pgxpool.Pool {
    return replicaPool
}

// Close closes all pools
func Close() {
    if primaryPool != nil {
        primaryPool.Close()
    }
    if replicaPool != nil && replicaPool != primaryPool {
        replicaPool.Close()
    }
}

// Stats returns pool statistics
func Stats() map[string]*pgxpool.Stat {
    stats := make(map[string]*pgxpool.Stat)
    if primaryPool != nil {
        stats["primary"] = primaryPool.Stat()
    }
    if replicaPool != nil && replicaPool != primaryPool {
        stats["replica"] = replicaPool.Stat()
    }
    return stats
}
```

### 3.2 User Repository

```go
// internal/repository/user_repo.go

package repository

import (
    "context"
    "errors"
    "fmt"
    "time"

    "github.com/google/uuid"
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"

    "github.com/myorg/myapp-go/internal/model"
    appErrors "github.com/myorg/myapp-go/pkg/errors"
)

type UserRepository struct {
    db      *pgxpool.Pool
    readDB  *pgxpool.Pool
}

func NewUserRepository(db, readDB *pgxpool.Pool) *UserRepository {
    return &UserRepository{db: db, readDB: readDB}
}

func (r *UserRepository) Create(ctx context.Context, user *model.User) error {
    query := `
        INSERT INTO users (
            id, email, username, hashed_password, full_name,
            is_active, is_verified, is_superuser, created_at, updated_at
        ) VALUES (
            $1, $2, $3, $4, $5, $6, $7, $8, $9, $10
        )
        RETURNING id, created_at, updated_at
    `
    
    now := time.Now().UTC()
    if user.ID == uuid.Nil {
        user.ID = uuid.New()
    }
    
    row := r.db.QueryRow(ctx, query,
        user.ID,
        user.Email,
        user.Username,
        user.HashedPassword,
        user.FullName,
        user.IsActive,
        user.IsVerified,
        user.IsSuperuser,
        now,
        now,
    )
    
    if err := row.Scan(&user.ID, &user.CreatedAt, &user.UpdatedAt); err != nil {
        if isDuplicateKeyError(err) {
            return appErrors.ErrAlreadyExists
        }
        return fmt.Errorf("failed to create user: %w", err)
    }
    
    return nil
}

func (r *UserRepository) GetByID(ctx context.Context, id uuid.UUID) (*model.User, error) {
    query := `
        SELECT
            id, email, username, hashed_password, full_name,
            avatar_url, is_active, is_verified, is_superuser,
            metadata, last_login_at, created_at, updated_at
        FROM users
        WHERE id = $1 AND is_active = true
    `
    
    user := &model.User{}
    row := r.readDB.QueryRow(ctx, query, id)
    
    err := row.Scan(
        &user.ID,
        &user.Email,
        &user.Username,
        &user.HashedPassword,
        &user.FullName,
        &user.AvatarURL,
        &user.IsActive,
        &user.IsVerified,
        &user.IsSuperuser,
        &user.Metadata,
        &user.LastLoginAt,
        &user.CreatedAt,
        &user.UpdatedAt,
    )
    
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, appErrors.ErrNotFound
        }
        return nil, fmt.Errorf("failed to get user by id: %w", err)
    }
    
    return user, nil
}

func (r *UserRepository) GetByEmail(ctx context.Context, email string) (*model.User, error) {
    query := `
        SELECT
            id, email, username, hashed_password, full_name,
            avatar_url, is_active, is_verified, is_superuser,
            metadata, last_login_at, created_at, updated_at
        FROM users
        WHERE email = $1
    `
    
    user := &model.User{}
    row := r.readDB.QueryRow(ctx, query, email)
    
    err := row.Scan(
        &user.ID,
        &user.Email,
        &user.Username,
        &user.HashedPassword,
        &user.FullName,
        &user.AvatarURL,
        &user.IsActive,
        &user.IsVerified,
        &user.IsSuperuser,
        &user.Metadata,
        &user.LastLoginAt,
        &user.CreatedAt,
        &user.UpdatedAt,
    )
    
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, appErrors.ErrNotFound
        }
        return nil, fmt.Errorf("failed to get user by email: %w", err)
    }
    
    return user, nil
}

func (r *UserRepository) List(ctx context.Context, params ListParams) ([]*model.User, int64, error) {
    // Count query
    countQuery := `
        SELECT COUNT(*) FROM users
        WHERE is_active = true
            AND ($1::text IS NULL OR email ILIKE '%' || $1 || '%' OR username ILIKE '%' || $1 || '%')
    `
    
    var total int64
    err := r.readDB.QueryRow(ctx, countQuery, params.Search).Scan(&total)
    if err != nil {
        return nil, 0, fmt.Errorf("failed to count users: %w", err)
    }
    
    if total == 0 {
        return []*model.User{}, 0, nil
    }
    
    // List query
    query := `
        SELECT
            id, email, username, full_name, avatar_url,
            is_active, is_verified, is_superuser, created_at, updated_at
        FROM users
        WHERE is_active = true
            AND ($1::text IS NULL OR email ILIKE '%' || $1 || '%' OR username ILIKE '%' || $1 || '%')
        ORDER BY created_at DESC
        LIMIT $2 OFFSET $3
    `
    
    rows, err := r.readDB.Query(ctx, query, params.Search, params.Limit, params.Offset)
    if err != nil {
        return nil, 0, fmt.Errorf("failed to list users: %w", err)
    }
    defer rows.Close()
    
    users := make([]*model.User, 0)
    for rows.Next() {
        user := &model.User{}
        err := rows.Scan(
            &user.ID,
            &user.Email,
            &user.Username,
            &user.FullName,
            &user.AvatarURL,
            &user.IsActive,
            &user.IsVerified,
            &user.IsSuperuser,
            &user.CreatedAt,
            &user.UpdatedAt,
        )
        if err != nil {
            return nil, 0, fmt.Errorf("failed to scan user: %w", err)
        }
        users = append(users, user)
    }
    
    if err := rows.Err(); err != nil {
        return nil, 0, fmt.Errorf("rows error: %w", err)
    }
    
    return users, total, nil
}

func (r *UserRepository) Update(ctx context.Context, user *model.User) error {
    query := `
        UPDATE users
        SET
            full_name = $2,
            avatar_url = $3,
            metadata = $4,
            updated_at = NOW()
        WHERE id = $1
        RETURNING updated_at
    `
    
    row := r.db.QueryRow(ctx, query, user.ID, user.FullName, user.AvatarURL, user.Metadata)
    if err := row.Scan(&user.UpdatedAt); err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return appErrors.ErrNotFound
        }
        return fmt.Errorf("failed to update user: %w", err)
    }
    
    return nil
}

func (r *UserRepository) Delete(ctx context.Context, id uuid.UUID) error {
    query := `UPDATE users SET is_active = false, updated_at = NOW() WHERE id = $1`
    
    result, err := r.db.Exec(ctx, query, id)
    if err != nil {
        return fmt.Errorf("failed to delete user: %w", err)
    }
    
    if result.RowsAffected() == 0 {
        return appErrors.ErrNotFound
    }
    
    return nil
}

// BulkInsert ใช้ pgx.CopyFrom สำหรับ batch inserts
func (r *UserRepository) BulkInsert(ctx context.Context, users []*model.User) (int64, error) {
    rows := make([][]interface{}, len(users))
    for i, user := range users {
        if user.ID == uuid.Nil {
            user.ID = uuid.New()
        }
        now := time.Now().UTC()
        rows[i] = []interface{}{
            user.ID,
            user.Email,
            user.Username,
            user.HashedPassword,
            user.FullName,
            user.IsActive,
            user.IsVerified,
            now,
            now,
        }
    }
    
    copyCount, err := r.db.CopyFrom(
        ctx,
        pgx.Identifier{"users"},
        []string{"id", "email", "username", "hashed_password", "full_name", "is_active", "is_verified", "created_at", "updated_at"},
        pgx.CopyFromRows(rows),
    )
    
    if err != nil {
        return 0, fmt.Errorf("bulk insert failed: %w", err)
    }
    
    return copyCount, nil
}

// WithTransaction execute ใน transaction
func (r *UserRepository) WithTransaction(ctx context.Context, fn func(tx pgx.Tx) error) error {
    tx, err := r.db.Begin(ctx)
    if err != nil {
        return fmt.Errorf("failed to begin transaction: %w", err)
    }
    
    defer func() {
        if p := recover(); p != nil {
            _ = tx.Rollback(ctx)
            panic(p)
        }
    }()
    
    if err := fn(tx); err != nil {
        if rbErr := tx.Rollback(ctx); rbErr != nil {
            return fmt.Errorf("tx error: %w, rollback error: %v", err, rbErr)
        }
        return err
    }
    
    return tx.Commit(ctx)
}

type ListParams struct {
    Search string
    Limit  int
    Offset int
}

func isDuplicateKeyError(err error) bool {
    return err != nil && err.Error() != "" &&
        (containsString(err.Error(), "23505") || containsString(err.Error(), "unique"))
}

func containsString(s, substr string) bool {
    return len(s) >= len(substr) && (s == substr || len(s) > 0)
}
```

---

## 4. Redis ด้วย go-redis

```go
// internal/database/redis.go

package database

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
    "go.uber.org/zap"

    "github.com/myorg/myapp-go/internal/config"
)

var (
    redisClient redis.UniversalClient
)

func InitRedis(cfg *config.RedisConfig, logger *zap.Logger) error {
    var client redis.UniversalClient
    
    if cfg.ClusterMode && len(cfg.ClusterAddrs) > 0 {
        // Cluster mode
        client = redis.NewClusterClient(&redis.ClusterOptions{
            Addrs:        cfg.ClusterAddrs,
            Password:     cfg.Password,
            MaxRetries:   cfg.MaxRetries,
            PoolSize:     cfg.PoolSize,
            MinIdleConns: cfg.MinIdleConns,
            DialTimeout:  time.Duration(cfg.DialTimeout) * time.Second,
            ReadTimeout:  time.Duration(cfg.ReadTimeout) * time.Second,
            WriteTimeout: time.Duration(cfg.WriteTimeout) * time.Second,
            
            // Route reads to replicas
            RouteByLatency: true,
            RouteRandomly:  false,
        })
    } else {
        // Single or Sentinel mode
        client = redis.NewClient(&redis.Options{
            Addr:         cfg.Addr(),
            Password:     cfg.Password,
            DB:           cfg.DB,
            MaxRetries:   cfg.MaxRetries,
            PoolSize:     cfg.PoolSize,
            MinIdleConns: cfg.MinIdleConns,
            DialTimeout:  time.Duration(cfg.DialTimeout) * time.Second,
            ReadTimeout:  time.Duration(cfg.ReadTimeout) * time.Second,
            WriteTimeout: time.Duration(cfg.WriteTimeout) * time.Second,
        })
    }
    
    // Test connection
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    if err := client.Ping(ctx).Err(); err != nil {
        return fmt.Errorf("failed to connect to Redis: %w", err)
    }
    
    redisClient = client
    
    logger.Info("Redis connected",
        zap.Bool("cluster_mode", cfg.ClusterMode),
        zap.Int("pool_size", cfg.PoolSize),
    )
    
    return nil
}

func GetRedis() redis.UniversalClient {
    return redisClient
}

func CloseRedis() error {
    if redisClient != nil {
        return redisClient.Close()
    }
    return nil
}

// internal/service/cache_service.go

package service

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
    "go.uber.org/zap"
)

type CacheService struct {
    client redis.UniversalClient
    logger *zap.Logger
}

func NewCacheService(client redis.UniversalClient, logger *zap.Logger) *CacheService {
    return &CacheService{client: client, logger: logger}
}

func (c *CacheService) Get(ctx context.Context, key string, dest interface{}) (bool, error) {
    val, err := c.client.Get(ctx, key).Bytes()
    if err != nil {
        if err == redis.Nil {
            return false, nil
        }
        c.logger.Warn("Cache GET failed", zap.String("key", key), zap.Error(err))
        return false, nil
    }
    
    if err := json.Unmarshal(val, dest); err != nil {
        c.logger.Warn("Cache unmarshal failed", zap.String("key", key), zap.Error(err))
        return false, nil
    }
    
    return true, nil
}

func (c *CacheService) Set(ctx context.Context, key string, value interface{}, ttl time.Duration) error {
    data, err := json.Marshal(value)
    if err != nil {
        return fmt.Errorf("failed to marshal cache value: %w", err)
    }
    
    if err := c.client.Set(ctx, key, data, ttl).Err(); err != nil {
        c.logger.Warn("Cache SET failed", zap.String("key", key), zap.Error(err))
        return nil // Don't fail on cache errors
    }
    
    return nil
}

func (c *CacheService) Delete(ctx context.Context, keys ...string) error {
    if err := c.client.Del(ctx, keys...).Err(); err != nil {
        c.logger.Warn("Cache DELETE failed", zap.Strings("keys", keys), zap.Error(err))
    }
    return nil
}

func (c *CacheService) DeletePattern(ctx context.Context, pattern string) (int64, error) {
    var cursor uint64
    var deleted int64
    
    for {
        keys, nextCursor, err := c.client.Scan(ctx, cursor, pattern, 100).Result()
        if err != nil {
            return deleted, fmt.Errorf("scan failed: %w", err)
        }
        
        if len(keys) > 0 {
            count, err := c.client.Del(ctx, keys...).Result()
            if err != nil {
                c.logger.Warn("Delete batch failed", zap.Error(err))
            } else {
                deleted += count
            }
        }
        
        cursor = nextCursor
        if cursor == 0 {
            break
        }
    }
    
    return deleted, nil
}

func (c *CacheService) Incr(ctx context.Context, key string, ttl time.Duration) (int64, error) {
    pipe := c.client.Pipeline()
    incrCmd := pipe.Incr(ctx, key)
    ttlCmd := pipe.TTL(ctx, key)
    
    _, err := pipe.Exec(ctx)
    if err != nil {
        return 0, fmt.Errorf("pipeline exec failed: %w", err)
    }
    
    count := incrCmd.Val()
    
    // Set TTL if key is new (TTL = -1 means no expiry)
    if ttlCmd.Val() == -1 {
        c.client.Expire(ctx, key, ttl)
    }
    
    return count, nil
}

func (c *CacheService) HGetAll(ctx context.Context, key string) (map[string]string, error) {
    result, err := c.client.HGetAll(ctx, key).Result()
    if err != nil {
        if err == redis.Nil {
            return nil, nil
        }
        return nil, fmt.Errorf("hgetall failed: %w", err)
    }
    return result, nil
}

// Pipeline operations
func (c *CacheService) Pipeline(ctx context.Context, fn func(pipe redis.Pipeliner) error) error {
    _, err := c.client.Pipelined(ctx, fn)
    return err
}

// Pub/Sub
func (c *CacheService) Publish(ctx context.Context, channel string, message interface{}) error {
    data, err := json.Marshal(message)
    if err != nil {
        return fmt.Errorf("marshal failed: %w", err)
    }
    return c.client.Publish(ctx, channel, data).Err()
}

func (c *CacheService) Subscribe(ctx context.Context, channels ...string) *redis.PubSub {
    return c.client.Subscribe(ctx, channels...)
}
```

---

## 5. S3/MinIO Storage

```go
// internal/service/storage_service.go

package service

import (
    "context"
    "fmt"
    "io"
    "mime"
    "path/filepath"
    "time"

    "github.com/aws/aws-sdk-go-v2/aws"
    awsConfig "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/credentials"
    "github.com/aws/aws-sdk-go-v2/service/s3"
    "github.com/aws/aws-sdk-go-v2/service/s3/types"
    "github.com/google/uuid"
    "go.uber.org/zap"

    "github.com/myorg/myapp-go/internal/config"
)

type StorageService struct {
    client     *s3.Client
    presigner  *s3.PresignClient
    bucketName string
    logger     *zap.Logger
    expiry     time.Duration
}

func NewStorageService(cfg *config.S3Config, logger *zap.Logger) (*StorageService, error) {
    var awsCfg aws.Config
    var err error
    
    if cfg.Endpoint != "" {
        // MinIO หรือ custom S3-compatible
        customResolver := aws.EndpointResolverWithOptionsFunc(func(service, region string, options ...interface{}) (aws.Endpoint, error) {
            return aws.Endpoint{
                URL:               cfg.Endpoint,
                SigningRegion:     cfg.Region,
                HostnameImmutable: true,
            }, nil
        })
        
        awsCfg, err = awsConfig.LoadDefaultConfig(context.Background(),
            awsConfig.WithRegion(cfg.Region),
            awsConfig.WithEndpointResolverWithOptions(customResolver),
            awsConfig.WithCredentialsProvider(credentials.NewStaticCredentialsProvider(
                cfg.AccessKeyID,
                cfg.SecretAccessKey,
                "",
            )),
        )
    } else {
        // AWS S3
        awsCfg, err = awsConfig.LoadDefaultConfig(context.Background(),
            awsConfig.WithRegion(cfg.Region),
            awsConfig.WithCredentialsProvider(credentials.NewStaticCredentialsProvider(
                cfg.AccessKeyID,
                cfg.SecretAccessKey,
                "",
            )),
        )
    }
    
    if err != nil {
        return nil, fmt.Errorf("failed to load AWS config: %w", err)
    }
    
    client := s3.NewFromConfig(awsCfg, func(o *s3.Options) {
        if cfg.Endpoint != "" {
            o.UsePathStyle = true
        }
    })
    
    return &StorageService{
        client:     client,
        presigner:  s3.NewPresignClient(client),
        bucketName: cfg.BucketName,
        logger:     logger,
        expiry:     time.Duration(cfg.PresignExpiry) * time.Second,
    }, nil
}

type UploadInput struct {
    Key         string
    Body        io.Reader
    ContentType string
    Metadata    map[string]string
    Size        int64
}

type UploadResult struct {
    Key    string
    URL    string
    ETag   string
    Bucket string
}

func (s *StorageService) GenerateKey(originalName string, prefix string) string {
    ext := filepath.Ext(originalName)
    uniqueID := uuid.New().String()
    datePrefix := time.Now().UTC().Format("2006/01/02")
    
    if prefix != "" {
        return fmt.Sprintf("%s/%s/%s%s", prefix, datePrefix, uniqueID, ext)
    }
    return fmt.Sprintf("%s/%s%s", datePrefix, uniqueID, ext)
}

func (s *StorageService) Upload(ctx context.Context, input UploadInput) (*UploadResult, error) {
    putInput := &s3.PutObjectInput{
        Bucket:        aws.String(s.bucketName),
        Key:           aws.String(input.Key),
        Body:          input.Body,
        ContentType:   aws.String(input.ContentType),
        ServerSideEncryption: types.ServerSideEncryptionAes256,
    }
    
    if len(input.Metadata) > 0 {
        putInput.Metadata = input.Metadata
    }
    
    if input.Size > 0 {
        putInput.ContentLength = aws.Int64(input.Size)
    }
    
    result, err := s.client.PutObject(ctx, putInput)
    if err != nil {
        return nil, fmt.Errorf("failed to upload to S3: %w", err)
    }
    
    url := fmt.Sprintf("https://%s.s3.amazonaws.com/%s", s.bucketName, input.Key)
    
    s.logger.Info("File uploaded to S3",
        zap.String("key", input.Key),
        zap.String("bucket", s.bucketName),
    )
    
    return &UploadResult{
        Key:    input.Key,
        URL:    url,
        ETag:   aws.ToString(result.ETag),
        Bucket: s.bucketName,
    }, nil
}

func (s *StorageService) PresignGetURL(ctx context.Context, key string) (string, error) {
    presignResult, err := s.presigner.PresignGetObject(ctx, &s3.GetObjectInput{
        Bucket: aws.String(s.bucketName),
        Key:    aws.String(key),
    }, func(opts *s3.PresignOptions) {
        opts.Expires = s.expiry
    })
    
    if err != nil {
        return "", fmt.Errorf("failed to presign GET URL: %w", err)
    }
    
    return presignResult.URL, nil
}

func (s *StorageService) PresignPutURL(ctx context.Context, key string, contentType string) (string, error) {
    presignResult, err := s.presigner.PresignPutObject(ctx, &s3.PutObjectInput{
        Bucket:      aws.String(s.bucketName),
        Key:         aws.String(key),
        ContentType: aws.String(contentType),
    }, func(opts *s3.PresignOptions) {
        opts.Expires = s.expiry
    })
    
    if err != nil {
        return "", fmt.Errorf("failed to presign PUT URL: %w", err)
    }
    
    return presignResult.URL, nil
}

func (s *StorageService) Delete(ctx context.Context, key string) error {
    _, err := s.client.DeleteObject(ctx, &s3.DeleteObjectInput{
        Bucket: aws.String(s.bucketName),
        Key:    aws.String(key),
    })
    if err != nil {
        return fmt.Errorf("failed to delete S3 object: %w", err)
    }
    return nil
}

func (s *StorageService) ListObjects(ctx context.Context, prefix string) ([]string, error) {
    var keys []string
    
    paginator := s3.NewListObjectsV2Paginator(s.client, &s3.ListObjectsV2Input{
        Bucket: aws.String(s.bucketName),
        Prefix: aws.String(prefix),
    })
    
    for paginator.HasMorePages() {
        page, err := paginator.NextPage(ctx)
        if err != nil {
            return nil, fmt.Errorf("failed to list S3 objects: %w", err)
        }
        
        for _, obj := range page.Contents {
            keys = append(keys, aws.ToString(obj.Key))
        }
    }
    
    return keys, nil
}

// GetContentType returns MIME type from file extension
func GetContentType(filename string) string {
    ext := filepath.Ext(filename)
    mimeType := mime.TypeByExtension(ext)
    if mimeType == "" {
        return "application/octet-stream"
    }
    return mimeType
}
```

---

## 6. Echo HTTP Server

```go
// cmd/api/main.go

package main

import (
    "context"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
    "go.uber.org/zap"

    "github.com/myorg/myapp-go/internal/config"
    "github.com/myorg/myapp-go/internal/database"
    "github.com/myorg/myapp-go/internal/handler"
    appMiddleware "github.com/myorg/myapp-go/internal/middleware"
    "github.com/myorg/myapp-go/internal/repository"
    "github.com/myorg/myapp-go/internal/service"
)

func main() {
    // Logger
    logger, _ := zap.NewProduction()
    defer logger.Sync()
    
    // Config
    cfg, err := config.Load()
    if err != nil {
        logger.Fatal("Failed to load config", zap.Error(err))
    }
    
    // Database
    if err := database.InitPostgres(&cfg.Database, logger); err != nil {
        logger.Fatal("Failed to initialize PostgreSQL", zap.Error(err))
    }
    defer database.Close()
    
    // Redis
    if err := database.InitRedis(&cfg.Redis, logger); err != nil {
        logger.Fatal("Failed to initialize Redis", zap.Error(err))
    }
    defer database.CloseRedis()
    
    // Services
    cacheService := service.NewCacheService(database.GetRedis(), logger)
    
    storageService, err := service.NewStorageService(&cfg.S3, logger)
    if err != nil {
        logger.Fatal("Failed to initialize storage service", zap.Error(err))
    }
    
    // Repositories
    userRepo := repository.NewUserRepository(
        database.GetPrimary(),
        database.GetReplica(),
    )
    
    productRepo := repository.NewProductRepository(
        database.GetPrimary(),
        database.GetReplica(),
    )
    
    // Handler Services
    userService := service.NewUserService(userRepo, cacheService, logger)
    productService := service.NewProductService(productRepo, cacheService, storageService, logger)
    
    // Handlers
    userHandler := handler.NewUserHandler(userService, logger)
    productHandler := handler.NewProductHandler(productService, logger)
    
    // Echo
    e := echo.New()
    e.HideBanner = true
    e.HidePort = true
    
    // Global Middleware
    e.Use(middleware.Recover())
    e.Use(middleware.RequestID())
    e.Use(appMiddleware.Logger(logger))
    e.Use(middleware.CORS())
    e.Use(middleware.Gzip())
    
    // Routes
    e.GET("/health", healthHandler)
    
    api := e.Group("/api/v1")
    
    // Auth
    authMiddleware := appMiddleware.Auth(cfg.JWT.SecretKey, logger)
    
    // Users
    users := api.Group("/users")
    users.GET("", userHandler.List, authMiddleware)
    users.GET("/:id", userHandler.GetByID, authMiddleware)
    users.POST("", userHandler.Create)
    users.PATCH("/:id", userHandler.Update, authMiddleware)
    users.DELETE("/:id", userHandler.Delete, authMiddleware)
    
    // Products
    products := api.Group("/products")
    products.GET("", productHandler.List)
    products.GET("/:id", productHandler.GetByID)
    products.POST("", productHandler.Create, authMiddleware)
    products.PATCH("/:id", productHandler.Update, authMiddleware)
    products.DELETE("/:id", productHandler.Delete, authMiddleware)
    products.POST("/:id/images", productHandler.UploadImage, authMiddleware)
    
    // Start server
    addr := fmt.Sprintf("%s:%d", cfg.App.Host, cfg.App.Port)
    
    go func() {
        logger.Info("Starting server", zap.String("addr", addr))
        if err := e.Start(addr); err != nil && err != http.ErrServerClosed {
            logger.Fatal("Server failed", zap.Error(err))
        }
    }()
    
    // Graceful shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    
    logger.Info("Shutting down server...")
    
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    if err := e.Shutdown(ctx); err != nil {
        logger.Fatal("Server shutdown failed", zap.Error(err))
    }
    
    logger.Info("Server stopped")
}

func healthHandler(c echo.Context) error {
    return c.JSON(http.StatusOK, map[string]interface{}{
        "status":  "ok",
        "version": config.Get().App.Version,
        "db_stats": database.Stats(),
    })
}
```

---

## 7. Goroutines สำหรับ Parallel Queries

```go
// internal/service/product_service.go

package service

import (
    "context"
    "fmt"
    "sync"
    "time"

    "go.uber.org/zap"

    "github.com/myorg/myapp-go/internal/model"
    "github.com/myorg/myapp-go/internal/repository"
)

type ProductService struct {
    repo    *repository.ProductRepository
    cache   *CacheService
    storage *StorageService
    logger  *zap.Logger
}

// GetProductWithDetails ดึงข้อมูล product พร้อม related data แบบ parallel
func (s *ProductService) GetProductWithDetails(ctx context.Context, productID string) (*model.ProductDetails, error) {
    var (
        product  *model.Product
        reviews  []*model.Review
        related  []*model.Product
        
        productErr error
        reviewsErr error
        relatedErr error
        
        wg sync.WaitGroup
    )
    
    // Check cache first
    cacheKey := fmt.Sprintf("product:details:%s", productID)
    var cached model.ProductDetails
    
    found, _ := s.cache.Get(ctx, cacheKey, &cached)
    if found {
        return &cached, nil
    }
    
    // Fetch all data concurrently
    wg.Add(3)
    
    go func() {
        defer wg.Done()
        product, productErr = s.repo.GetByID(ctx, productID)
    }()
    
    go func() {
        defer wg.Done()
        reviews, reviewsErr = s.repo.GetReviews(ctx, productID, 10)
    }()
    
    go func() {
        defer wg.Done()
        related, relatedErr = s.repo.GetRelatedProducts(ctx, productID, 5)
    }()
    
    wg.Wait()
    
    // Check errors
    if productErr != nil {
        return nil, fmt.Errorf("failed to get product: %w", productErr)
    }
    
    if reviewsErr != nil {
        s.logger.Warn("Failed to get reviews", zap.Error(reviewsErr))
        reviews = []*model.Review{}
    }
    
    if relatedErr != nil {
        s.logger.Warn("Failed to get related products", zap.Error(relatedErr))
        related = []*model.Product{}
    }
    
    details := &model.ProductDetails{
        Product: product,
        Reviews: reviews,
        Related: related,
    }
    
    // Cache result
    s.cache.Set(ctx, cacheKey, details, 5*time.Minute)
    
    return details, nil
}

// BulkGetProducts ดึง multiple products แบบ parallel ด้วย worker pool
func (s *ProductService) BulkGetProducts(ctx context.Context, ids []string) ([]*model.Product, error) {
    type result struct {
        index   int
        product *model.Product
        err     error
    }
    
    results := make([]*model.Product, len(ids))
    resultCh := make(chan result, len(ids))
    
    // Worker pool
    workerCount := 5
    if len(ids) < workerCount {
        workerCount = len(ids)
    }
    
    idCh := make(chan struct {
        index int
        id    string
    }, len(ids))
    
    // Send work
    for i, id := range ids {
        idCh <- struct {
            index int
            id    string
        }{i, id}
    }
    close(idCh)
    
    // Start workers
    var wg sync.WaitGroup
    for i := 0; i < workerCount; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for work := range idCh {
                product, err := s.repo.GetByID(ctx, work.id)
                resultCh <- result{work.index, product, err}
            }
        }()
    }
    
    // Close result channel when workers done
    go func() {
        wg.Wait()
        close(resultCh)
    }()
    
    // Collect results
    var firstErr error
    for r := range resultCh {
        if r.err != nil {
            if firstErr == nil {
                firstErr = r.err
            }
            continue
        }
        results[r.index] = r.product
    }
    
    if firstErr != nil {
        return nil, firstErr
    }
    
    return results, nil
}

// ProcessOrders processes multiple orders concurrently with context cancellation
func (s *ProductService) ProcessInventoryUpdates(
    ctx context.Context,
    updates []InventoryUpdate,
) ([]InventoryResult, error) {
    results := make([]InventoryResult, len(updates))
    errCh := make(chan error, 1)
    
    // Use errgroup for clean error handling
    var wg sync.WaitGroup
    var mu sync.Mutex
    
    sem := make(chan struct{}, 10) // Semaphore: max 10 concurrent
    
    for i, update := range updates {
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        default:
        }
        
        wg.Add(1)
        sem <- struct{}{} // Acquire semaphore
        
        go func(idx int, u InventoryUpdate) {
            defer wg.Done()
            defer func() { <-sem }() // Release semaphore
            
            result, err := s.repo.UpdateInventory(ctx, u.ProductID, u.Delta)
            
            mu.Lock()
            defer mu.Unlock()
            
            if err != nil {
                select {
                case errCh <- fmt.Errorf("update %d failed: %w", idx, err):
                default:
                }
                results[idx] = InventoryResult{Error: err.Error()}
            } else {
                results[idx] = InventoryResult{
                    ProductID:       u.ProductID,
                    NewQuantity: result,
                }
            }
        }(i, update)
    }
    
    wg.Wait()
    
    // Check for errors
    select {
    case err := <-errCh:
        return results, err
    default:
        return results, nil
    }
}

type InventoryUpdate struct {
    ProductID string
    Delta     int // positive = add, negative = remove
}

type InventoryResult struct {
    ProductID       string
    NewQuantity int
    Error       string
}
```

---

## 8. Complete Handler Example

```go
// internal/handler/product.go

package handler

import (
    "net/http"
    "strconv"

    "github.com/google/uuid"
    "github.com/labstack/echo/v4"
    "go.uber.org/zap"

    "github.com/myorg/myapp-go/internal/model"
    "github.com/myorg/myapp-go/internal/service"
    appErrors "github.com/myorg/myapp-go/pkg/errors"
    "github.com/myorg/myapp-go/pkg/pagination"
)

type ProductHandler struct {
    service *service.ProductService
    logger  *zap.Logger
}

func NewProductHandler(svc *service.ProductService, logger *zap.Logger) *ProductHandler {
    return &ProductHandler{service: svc, logger: logger}
}

// List godoc
// @Summary List products
// @Tags products
// @Accept json
// @Produce json
// @Param page query int false "Page number" default(1)
// @Param per_page query int false "Items per page" default(20)
// @Param search query string false "Search term"
// @Success 200 {object} pagination.Page[model.ProductResponse]
// @Router /api/v1/products [get]
func (h *ProductHandler) List(c echo.Context) error {
    page, _ := strconv.Atoi(c.QueryParam("page"))
    if page < 1 {
        page = 1
    }
    
    perPage, _ := strconv.Atoi(c.QueryParam("per_page"))
    if perPage < 1 || perPage > 100 {
        perPage = 20
    }
    
    search := c.QueryParam("search")
    
    params := service.ProductListParams{
        Search:   search,
        Page:     page,
        PerPage:  perPage,
        IsActive: true,
    }
    
    products, total, err := h.service.List(c.Request().Context(), params)
    if err != nil {
        h.logger.Error("Failed to list products", zap.Error(err))
        return echo.NewHTTPError(http.StatusInternalServerError, "Failed to list products")
    }
    
    return c.JSON(http.StatusOK, pagination.NewPage(products, total, page, perPage))
}

func (h *ProductHandler) GetByID(c echo.Context) error {
    idStr := c.Param("id")
    productID, err := uuid.Parse(idStr)
    if err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, "Invalid product ID")
    }
    
    product, err := h.service.GetByID(c.Request().Context(), productID)
    if err != nil {
        if appErrors.Is(err, appErrors.ErrNotFound) {
            return echo.NewHTTPError(http.StatusNotFound, "Product not found")
        }
        return echo.NewHTTPError(http.StatusInternalServerError, "Failed to get product")
    }
    
    return c.JSON(http.StatusOK, product)
}

func (h *ProductHandler) Create(c echo.Context) error {
    var input model.CreateProductInput
    if err := c.Bind(&input); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, "Invalid request body")
    }
    
    if err := c.Validate(&input); err != nil {
        return echo.NewHTTPError(http.StatusUnprocessableEntity, err.Error())
    }
    
    product, err := h.service.Create(c.Request().Context(), &input)
    if err != nil {
        if appErrors.Is(err, appErrors.ErrAlreadyExists) {
            return echo.NewHTTPError(http.StatusConflict, "Product SKU already exists")
        }
        h.logger.Error("Failed to create product", zap.Error(err))
        return echo.NewHTTPError(http.StatusInternalServerError, "Failed to create product")
    }
    
    return c.JSON(http.StatusCreated, product)
}

func (h *ProductHandler) Update(c echo.Context) error {
    idStr := c.Param("id")
    productID, err := uuid.Parse(idStr)
    if err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, "Invalid product ID")
    }
    
    var input model.UpdateProductInput
    if err := c.Bind(&input); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, "Invalid request body")
    }
    
    product, err := h.service.Update(c.Request().Context(), productID, &input)
    if err != nil {
        if appErrors.Is(err, appErrors.ErrNotFound) {
            return echo.NewHTTPError(http.StatusNotFound, "Product not found")
        }
        return echo.NewHTTPError(http.StatusInternalServerError, "Failed to update product")
    }
    
    return c.JSON(http.StatusOK, product)
}

func (h *ProductHandler) Delete(c echo.Context) error {
    idStr := c.Param("id")
    productID, err := uuid.Parse(idStr)
    if err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, "Invalid product ID")
    }
    
    if err := h.service.Delete(c.Request().Context(), productID); err != nil {
        if appErrors.Is(err, appErrors.ErrNotFound) {
            return echo.NewHTTPError(http.StatusNotFound, "Product not found")
        }
        return echo.NewHTTPError(http.StatusInternalServerError, "Failed to delete product")
    }
    
    return c.NoContent(http.StatusNoContent)
}

func (h *ProductHandler) UploadImage(c echo.Context) error {
    idStr := c.Param("id")
    productID, err := uuid.Parse(idStr)
    if err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, "Invalid product ID")
    }
    
    file, err := c.FormFile("image")
    if err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, "Image file is required")
    }
    
    // Validate file size (max 10MB)
    if file.Size > 10*1024*1024 {
        return echo.NewHTTPError(http.StatusRequestEntityTooLarge, "File size exceeds 10MB")
    }
    
    src, err := file.Open()
    if err != nil {
        return echo.NewHTTPError(http.StatusInternalServerError, "Failed to open file")
    }
    defer src.Close()
    
    result, err := h.service.UploadImage(
        c.Request().Context(),
        productID,
        src,
        file.Filename,
        file.Header.Get("Content-Type"),
        file.Size,
    )
    if err != nil {
        h.logger.Error("Failed to upload image", zap.Error(err))
        return echo.NewHTTPError(http.StatusInternalServerError, "Failed to upload image")
    }
    
    return c.JSON(http.StatusOK, result)
}
```

---

## 9. Custom Errors และ Middleware

```go
// pkg/errors/errors.go

package errors

import (
    "errors"
    "fmt"
)

var (
    ErrNotFound      = errors.New("not found")
    ErrAlreadyExists = errors.New("already exists")
    ErrUnauthorized  = errors.New("unauthorized")
    ErrForbidden     = errors.New("forbidden")
    ErrBadRequest    = errors.New("bad request")
    ErrInternal      = errors.New("internal error")
)

type AppError struct {
    Code    string
    Message string
    Cause   error
}

func (e *AppError) Error() string {
    if e.Cause != nil {
        return fmt.Sprintf("%s: %v", e.Message, e.Cause)
    }
    return e.Message
}

func (e *AppError) Unwrap() error {
    return e.Cause
}

func Is(err, target error) bool {
    return errors.Is(err, target)
}

func Wrap(err error, msg string) error {
    return fmt.Errorf("%s: %w", msg, err)
}

// internal/middleware/auth.go

package middleware

import (
    "net/http"
    "strings"

    "github.com/golang-jwt/jwt/v5"
    "github.com/labstack/echo/v4"
    "go.uber.org/zap"
)

type Claims struct {
    UserID string `json:"sub"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func Auth(secretKey string, logger *zap.Logger) echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            authHeader := c.Request().Header.Get("Authorization")
            if authHeader == "" {
                return echo.NewHTTPError(http.StatusUnauthorized, "Authorization header required")
            }
            
            parts := strings.SplitN(authHeader, " ", 2)
            if len(parts) != 2 || parts[0] != "Bearer" {
                return echo.NewHTTPError(http.StatusUnauthorized, "Invalid authorization format")
            }
            
            token, err := jwt.ParseWithClaims(
                parts[1],
                &Claims{},
                func(token *jwt.Token) (interface{}, error) {
                    if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                        return nil, echo.NewHTTPError(http.StatusUnauthorized, "Unexpected signing method")
                    }
                    return []byte(secretKey), nil
                },
            )
            
            if err != nil {
                logger.Debug("JWT validation failed", zap.Error(err))
                return echo.NewHTTPError(http.StatusUnauthorized, "Invalid or expired token")
            }
            
            claims, ok := token.Claims.(*Claims)
            if !ok || !token.Valid {
                return echo.NewHTTPError(http.StatusUnauthorized, "Invalid token claims")
            }
            
            // Store claims in context
            c.Set("user_id", claims.UserID)
            c.Set("user_email", claims.Email)
            c.Set("user_role", claims.Role)
            
            return next(c)
        }
    }
}
```

---

## 10. Build และ Run

```bash
# Build
go build -o bin/api ./cmd/api

# Run
./bin/api

# Run with hot reload (air)
air

# Tests
go test ./... -v -race -count=1

# Benchmark
go test ./... -bench=. -benchmem

# Docker build
docker build -t myapp-go:latest .

# Dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o bin/api ./cmd/api

FROM alpine:3.18
RUN apk --no-cache add ca-certificates tzdata
WORKDIR /app
COPY --from=builder /app/bin/api ./api
EXPOSE 8080
ENTRYPOINT ["./api"]
```

---

## สรุป

Go ร่วมกับ pgx, go-redis และ aws-sdk-go-v2 เป็น combination ที่ทรงพลังสำหรับการสร้าง high-performance backend:

1. **pgx v5** - PostgreSQL driver ที่เร็วที่สุดสำหรับ Go พร้อม pgxpool สำหรับ connection pooling
2. **go-redis** - Redis client ที่รองรับทั้ง standalone และ cluster mode
3. **aws-sdk-go-v2** - AWS SDK ที่ support context-aware operations และ presigned URLs
4. **Echo** - Fast HTTP framework ที่ใช้งานง่าย
5. **Goroutines** - ทำให้ parallel I/O operations ง่ายมาก

จุดเด่นของ Go คือ performance และ concurrency ที่ดีเยี่ยม ทำให้เหมาะสำหรับ high-throughput APIs และ microservices

---

*เนื้อหาส่วนนี้เป็นส่วนหนึ่งของ Database Cluster Course - Advanced Content*
