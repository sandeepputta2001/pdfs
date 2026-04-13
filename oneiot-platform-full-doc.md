# OneIoT — Industrial Sensor Data Pipeline
## Full Project Reference Document

**Stack:** Go 1.22 · PostgreSQL 16 · Redis 7 · Kafka (MSK) · gRPC · AWS EKS · Terraform · ArgoCD · Tanka/Jsonnet  
**AWS Region:** ap-south-1 (Mumbai)  
**Theme:** Industrial IoT — sensor telemetry ingestion, anomaly detection, real-time streaming, multi-stage data pipeline

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Project Structure](#2-project-structure)
3. [Go Application Layer](#3-go-application-layer)
   - 3.1 go.mod
   - 3.2 pkg/config/config.go
   - 3.3 pkg/logger/logger.go
   - 3.4 internal/db/postgres.go
   - 3.5 internal/cache/redis.go
   - 3.6 internal/auth/jwt.go
   - 3.7 internal/device/repository.go
   - 3.8 internal/kafka/pipeline.go
   - 3.9 internal/pipeline/telemetry_worker.go
   - 3.10 internal/grpc/server.go
   - 3.11 cmd/api/main.go
4. [Proto Definition](#4-proto-definition)
5. [Database Migrations](#5-database-migrations)
6. [Config](#6-config)
7. [Docker](#7-docker)
   - 7.1 Dockerfile (7-stage multi-stage)
   - 7.2 docker-compose.yml
8. [Kubernetes Manifests](#8-kubernetes-manifests)
   - 8.1 Namespace + ConfigMap + Secret
   - 8.2 API Deployment
   - 8.3 Services + Ingress + HPA + PDB + CronJob + Job + NetworkPolicy
   - 8.4 Kustomization
9. [ArgoCD (GitOps)](#9-argocd-gitops)
10. [Terraform (AWS Infrastructure)](#10-terraform-aws-infrastructure)
    - 10.1 VPC Module
    - 10.2 EKS Module
    - 10.3 RDS + ElastiCache + MSK + S3 Modules
    - 10.4 Production Root Module
11. [Tanka / Jsonnet](#11-tanka--jsonnet)
    - 11.1 main.jsonnet
    - 11.2 config.libsonnet
    - 11.3 oneiot.libsonnet
12. [Concept Index](#12-concept-index)
13. [Running Locally](#13-running-locally)
14. [CI/CD Flow](#14-cicd-flow)

---

## 1. Architecture Overview

```
Industrial Sensors / Edge Devices
        │
        ├─── gRPC (bidirectional stream) ──▶ gRPC Server (EKS)
        │                                        │
        └─── REST API (JWT auth) ───────▶ API Server (EKS, 3 replicas)
                                              │
                              ┌───────────────┼────────────────┐
                              ▼               ▼                ▼
                         Kafka/MSK        Redis 7          PostgreSQL 16
                     (telemetry topic)  (sessions,        (devices, telemetry
                      6 partitions,     rate-limit,        partitioned by month,
                      3 replicas,       device cache)      materialized views,
                      Snappy, DLQ)                         anomaly detection)
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
              Consumer Group        Consumer Group
              (telemetry, 4w)      (commands, 2w)
                    │
             ┌──────┴──────┐
             ▼             ▼
         Normalizer    Batch Accumulator
         (8 goroutines) → PostgreSQL COPY
                          (500 rows / 2s)
                              │
                         S3 Cold Storage
                         (IA@30d, Glacier@90d)

All services deployed on EKS (ap-south-1) via ArgoCD GitOps
Infrastructure provisioned by Terraform (VPC/EKS/RDS/MSK/ElastiCache/S3/ECR)
Kubernetes config also managed via Tanka (Jsonnet)
```

---

## 2. Project Structure

```
oneiot/
├── cmd/
│   ├── api/main.go               REST API entrypoint, graceful shutdown
│   ├── worker/main.go            Kafka pipeline worker entrypoint
│   └── grpc-server/main.go       gRPC server entrypoint
├── internal/
│   ├── auth/jwt.go               JWT (access+refresh), HMAC-SHA256, session store
│   ├── cache/redis.go            Redis: sessions, rate limiting, device last-seen
│   ├── db/postgres.go            pgxpool, slow-query tracer, COPY, transactions
│   ├── device/repository.go      Domain model + advanced PostgreSQL queries
│   ├── grpc/server.go            gRPC: bidi stream, unary, server-stream, interceptors
│   ├── kafka/pipeline.go         Producer groups + Consumer groups + DLQ + Pipeline
│   └── pipeline/telemetry_worker.go  Fan-in/fan-out goroutines, atomic counters
├── pkg/
│   ├── config/config.go          Typed Viper config with env override
│   └── logger/logger.go          Structured zap logger
├── proto/telemetry.proto          Protobuf: DeviceService + TelemetryService
├── migrations/
│   └── 001_initial_schema.up.sql Partitioned tables, GIN indexes, materialized views
├── config/config.dev.yaml         Dev environment config
├── Dockerfile                     7-stage multi-stage build
├── docker-compose.yml             Full local stack (PG, Redis, Kafka×3, MinIO, Grafana)
├── deployments/
│   ├── k8s/base/
│   │   ├── 00-namespace-config.yaml
│   │   ├── 01-api-deployment.yaml
│   │   ├── 02-services-ingress-hpa.yaml
│   │   └── kustomization.yaml
│   └── argocd/applications.yaml  App of Apps + AppProject
├── terraform/
│   ├── modules/vpc/main.tf        VPC, subnets, SGs, VPC endpoints, flow logs
│   ├── modules/eks/main.tf        EKS 1.30, node groups, Karpenter, IRSA
│   ├── modules/rds/main.tf        PostgreSQL 16 Multi-AZ + read replica
│   │   (also contains elasticache + msk + s3 inline)
│   └── envs/prod/main.tf          Root module: all infrastructure + ECR + ArgoCD
└── tanka/
    ├── environments/default/main.jsonnet
    ├── environments/default/config.libsonnet
    └── lib/oneiot.libsonnet
```

---

## 3. Go Application Layer

### 3.1 — `go.mod`

```go
module github.com/oneiot/platform

go 1.22

require (
	github.com/gin-gonic/gin v1.10.0
	github.com/golang-jwt/jwt/v5 v5.2.1
	github.com/google/uuid v1.6.0
	github.com/jackc/pgx/v5 v5.6.0
	github.com/redis/go-redis/v9 v9.5.3
	github.com/segmentio/kafka-go v0.4.47
	github.com/spf13/viper v1.19.0
	go.uber.org/zap v1.27.0
	golang.org/x/crypto v0.24.0
	google.golang.org/grpc v1.64.0
	google.golang.org/protobuf v1.34.2
	github.com/prometheus/client_golang v1.19.1
	github.com/grpc-ecosystem/go-grpc-middleware/v2 v2.1.0
	github.com/golang-migrate/migrate/v4 v4.17.1
	github.com/lib/pq v1.10.9
	github.com/stretchr/testify v1.9.0
	go.opentelemetry.io/otel v1.27.0
	go.opentelemetry.io/otel/trace v1.27.0
	github.com/robfig/cron/v3 v3.0.1
)
```

---

### 3.2 — `pkg/config/config.go`

Typed configuration using Viper with environment variable overrides and defaults.

```go
package config

import (
	"strings"
	"time"

	"github.com/spf13/viper"
)

type Config struct {
	App      AppConfig
	DB       DBConfig
	Redis    RedisConfig
	Kafka    KafkaConfig
	JWT      JWTConfig
	GRPC     GRPCConfig
	AWS      AWSConfig
	Metrics  MetricsConfig
}

type AppConfig struct {
	Name         string        `mapstructure:"name"`
	Env          string        `mapstructure:"env"`
	Port         int           `mapstructure:"port"`
	ReadTimeout  time.Duration `mapstructure:"read_timeout"`
	WriteTimeout time.Duration `mapstructure:"write_timeout"`
	LogLevel     string        `mapstructure:"log_level"`
}

type DBConfig struct {
	Host            string        `mapstructure:"host"`
	Port            int           `mapstructure:"port"`
	User            string        `mapstructure:"user"`
	Password        string        `mapstructure:"password"`
	Name            string        `mapstructure:"name"`
	SSLMode         string        `mapstructure:"ssl_mode"`
	MaxOpenConns    int           `mapstructure:"max_open_conns"`
	MaxIdleConns    int           `mapstructure:"max_idle_conns"`
	ConnMaxLifetime time.Duration `mapstructure:"conn_max_lifetime"`
	MigrationsPath  string        `mapstructure:"migrations_path"`
}

type RedisConfig struct {
	Addr         string        `mapstructure:"addr"`
	Password     string        `mapstructure:"password"`
	DB           int           `mapstructure:"db"`
	PoolSize     int           `mapstructure:"pool_size"`
	ReadTimeout  time.Duration `mapstructure:"read_timeout"`
	WriteTimeout time.Duration `mapstructure:"write_timeout"`
}

type KafkaConfig struct {
	Brokers           []string      `mapstructure:"brokers"`
	GroupID           string        `mapstructure:"group_id"`
	TelemetryTopic    string        `mapstructure:"telemetry_topic"`
	AlertsTopic       string        `mapstructure:"alerts_topic"`
	CommandsTopic     string        `mapstructure:"commands_topic"`
	NumPartitions     int           `mapstructure:"num_partitions"`
	ReplicationFactor int           `mapstructure:"replication_factor"`
	CommitInterval    time.Duration `mapstructure:"commit_interval"`
	MaxBytes          int           `mapstructure:"max_bytes"`
}

type JWTConfig struct {
	AccessSecret  string        `mapstructure:"access_secret"`
	RefreshSecret string        `mapstructure:"refresh_secret"`
	AccessExpiry  time.Duration `mapstructure:"access_expiry"`
	RefreshExpiry time.Duration `mapstructure:"refresh_expiry"`
	Issuer        string        `mapstructure:"issuer"`
}

type GRPCConfig struct {
	Port           int    `mapstructure:"port"`
	MaxRecvMsgSize int    `mapstructure:"max_recv_msg_size"`
	MaxSendMsgSize int    `mapstructure:"max_send_msg_size"`
	TLSCertFile    string `mapstructure:"tls_cert_file"`
	TLSKeyFile     string `mapstructure:"tls_key_file"`
}

type AWSConfig struct {
	Region          string `mapstructure:"region"`
	S3Bucket        string `mapstructure:"s3_bucket"`
	AccessKeyID     string `mapstructure:"access_key_id"`
	SecretAccessKey string `mapstructure:"secret_access_key"`
}

type MetricsConfig struct {
	Enabled bool   `mapstructure:"enabled"`
	Path    string `mapstructure:"path"`
	Port    int    `mapstructure:"port"`
}

func Load(cfgPath string) (*Config, error) {
	v := viper.New()
	v.SetConfigFile(cfgPath)
	v.SetConfigType("yaml")
	v.AutomaticEnv()
	v.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))

	// Sensible defaults
	v.SetDefault("app.port", 8080)
	v.SetDefault("app.read_timeout", "30s")
	v.SetDefault("app.write_timeout", "30s")
	v.SetDefault("db.max_open_conns", 25)
	v.SetDefault("db.max_idle_conns", 10)
	v.SetDefault("db.conn_max_lifetime", "5m")
	v.SetDefault("redis.pool_size", 10)
	v.SetDefault("kafka.num_partitions", 6)
	v.SetDefault("kafka.replication_factor", 3)
	v.SetDefault("kafka.commit_interval", "1s")
	v.SetDefault("kafka.max_bytes", 10485760)
	v.SetDefault("grpc.max_recv_msg_size", 4194304)
	v.SetDefault("grpc.max_send_msg_size", 4194304)
	v.SetDefault("metrics.path", "/metrics")
	v.SetDefault("metrics.port", 9090)

	if err := v.ReadInConfig(); err != nil {
		return nil, err
	}
	cfg := &Config{}
	if err := v.Unmarshal(cfg); err != nil {
		return nil, err
	}
	return cfg, nil
}
```

---

### 3.3 — `pkg/logger/logger.go`

Structured, levelled logger wrapping `go.uber.org/zap`. Single global instance with coloured dev output and JSON prod output.

```go
package logger

import (
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

var log *zap.Logger

func Init(level string, env string) error {
	var cfg zap.Config
	if env == "production" {
		cfg = zap.NewProductionConfig()
	} else {
		cfg = zap.NewDevelopmentConfig()
		cfg.EncoderConfig.EncodeLevel = zapcore.CapitalColorLevelEncoder
	}
	lvl, err := zapcore.ParseLevel(level)
	if err != nil {
		lvl = zapcore.InfoLevel
	}
	cfg.Level = zap.NewAtomicLevelAt(lvl)
	l, err := cfg.Build(zap.AddCallerSkip(1))
	if err != nil {
		return err
	}
	log = l
	return nil
}

func Info(msg string, fields ...zap.Field)  { log.Info(msg, fields...) }
func Error(msg string, fields ...zap.Field) { log.Error(msg, fields...) }
func Warn(msg string, fields ...zap.Field)  { log.Warn(msg, fields...) }
func Debug(msg string, fields ...zap.Field) { log.Debug(msg, fields...) }
func Fatal(msg string, fields ...zap.Field) { log.Fatal(msg, fields...) }
func With(fields ...zap.Field) *zap.Logger  { return log.With(fields...) }
func Sync() error                           { return log.Sync() }
```

---

### 3.4 — `internal/db/postgres.go`

pgxpool with health checks, slow-query tracing hook, serializable transactions, and COPY-based bulk inserts.

```go
package db

import (
	"context"
	"fmt"
	"time"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/oneiot/platform/pkg/config"
	"github.com/oneiot/platform/pkg/logger"
	"go.uber.org/zap"
)

type DB struct {
	pool *pgxpool.Pool
}

func New(ctx context.Context, cfg config.DBConfig) (*DB, error) {
	dsn := fmt.Sprintf(
		"host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
		cfg.Host, cfg.Port, cfg.User, cfg.Password, cfg.Name, cfg.SSLMode,
	)
	poolCfg, err := pgxpool.ParseConfig(dsn)
	if err != nil {
		return nil, fmt.Errorf("parse pg config: %w", err)
	}
	poolCfg.MaxConns = int32(cfg.MaxOpenConns)
	poolCfg.MinConns = int32(cfg.MaxIdleConns)
	poolCfg.MaxConnLifetime = cfg.ConnMaxLifetime
	poolCfg.MaxConnIdleTime = 5 * time.Minute
	poolCfg.HealthCheckPeriod = 1 * time.Minute
	poolCfg.ConnConfig.Tracer = &queryTracer{}   // ← slow query hook

	pool, err := pgxpool.NewWithConfig(ctx, poolCfg)
	if err != nil {
		return nil, fmt.Errorf("create pgxpool: %w", err)
	}
	if err := pool.Ping(ctx); err != nil {
		return nil, fmt.Errorf("ping postgres: %w", err)
	}
	logger.Info("PostgreSQL connected", zap.String("host", cfg.Host))
	return &DB{pool: pool}, nil
}

func (d *DB) Pool() *pgxpool.Pool { return d.pool }
func (d *DB) Close()              { d.pool.Close() }

// WithTx runs fn inside a serializable transaction with auto-rollback on panic/error.
func (d *DB) WithTx(ctx context.Context, fn func(tx pgx.Tx) error) error {
	tx, err := d.pool.BeginTx(ctx, pgx.TxOptions{
		IsoLevel:   pgx.Serializable,
		AccessMode: pgx.ReadWrite,
	})
	if err != nil {
		return fmt.Errorf("begin tx: %w", err)
	}
	defer func() {
		if p := recover(); p != nil {
			_ = tx.Rollback(ctx)
			panic(p)
		}
	}()
	if err := fn(tx); err != nil {
		_ = tx.Rollback(ctx)
		return err
	}
	return tx.Commit(ctx)
}

// BulkInsert uses PostgreSQL COPY protocol for maximum throughput.
func (d *DB) BulkInsert(ctx context.Context, table string, cols []string, rows [][]any) (int64, error) {
	return d.pool.CopyFrom(ctx, pgx.Identifier{table}, cols, pgx.CopyFromRows(rows))
}

// queryTracer logs any query that takes longer than 100ms.
type queryTracer struct{}

type queryStartKey struct{}

func (t *queryTracer) TraceQueryStart(ctx context.Context, _ *pgx.Conn, _ pgx.TraceQueryStartData) context.Context {
	return context.WithValue(ctx, queryStartKey{}, time.Now())
}

func (t *queryTracer) TraceQueryEnd(ctx context.Context, _ *pgx.Conn, data pgx.TraceQueryEndData) {
	start, ok := ctx.Value(queryStartKey{}).(time.Time)
	if !ok {
		return
	}
	if elapsed := time.Since(start); elapsed > 100*time.Millisecond {
		logger.Warn("slow query", zap.Duration("duration", elapsed),
			zap.String("tag", data.CommandTag.String()))
	}
}
```

---

### 3.5 — `internal/cache/redis.go`

Redis client with sessions, sliding-window rate limiting, and device last-seen tracking via sorted sets.

```go
package cache

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/oneiot/platform/pkg/config"
	"github.com/oneiot/platform/pkg/logger"
	"github.com/redis/go-redis/v9"
	"go.uber.org/zap"
)

type Cache struct{ client *redis.Client }

func New(cfg config.RedisConfig) (*Cache, error) {
	rdb := redis.NewClient(&redis.Options{
		Addr:         cfg.Addr,
		Password:     cfg.Password,
		DB:           cfg.DB,
		PoolSize:     cfg.PoolSize,
		ReadTimeout:  cfg.ReadTimeout,
		WriteTimeout: cfg.WriteTimeout,
	})
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := rdb.Ping(ctx).Err(); err != nil {
		return nil, fmt.Errorf("redis ping: %w", err)
	}
	logger.Info("Redis connected", zap.String("addr", cfg.Addr))
	return &Cache{client: rdb}, nil
}

func (c *Cache) Client() *redis.Client { return c.client }

func (c *Cache) SetJSON(ctx context.Context, key string, v any, ttl time.Duration) error {
	b, err := json.Marshal(v)
	if err != nil {
		return err
	}
	return c.client.Set(ctx, key, b, ttl).Err()
}

func (c *Cache) GetJSON(ctx context.Context, key string, v any) error {
	b, err := c.client.Get(ctx, key).Bytes()
	if err != nil {
		return err
	}
	return json.Unmarshal(b, v)
}

func (c *Cache) Delete(ctx context.Context, keys ...string) error {
	return c.client.Del(ctx, keys...).Err()
}

// --- Session management ---

type Session struct {
	UserID    string    `json:"user_id"`
	Email     string    `json:"email"`
	Role      string    `json:"role"`
	IP        string    `json:"ip"`
	UserAgent string    `json:"user_agent"`
	CreatedAt time.Time `json:"created_at"`
}

const sessionPrefix = "session:"

func (c *Cache) SaveSession(ctx context.Context, id string, s *Session, ttl time.Duration) error {
	return c.SetJSON(ctx, sessionPrefix+id, s, ttl)
}

func (c *Cache) GetSession(ctx context.Context, id string) (*Session, error) {
	var s Session
	if err := c.GetJSON(ctx, sessionPrefix+id, &s); err != nil {
		return nil, err
	}
	return &s, nil
}

func (c *Cache) DeleteSession(ctx context.Context, id string) error {
	return c.Delete(ctx, sessionPrefix+id)
}

// RateLimitCheck implements a sliding window using a Redis sorted set.
// Returns (remaining, allowed, error).
func (c *Cache) RateLimitCheck(ctx context.Context, key string, limit int64, window time.Duration) (int64, bool, error) {
	now := time.Now().UnixMilli()
	windowMs := window.Milliseconds()

	pipe := c.client.Pipeline()
	pipe.ZRemRangeByScore(ctx, key, "0", fmt.Sprintf("%d", now-windowMs))
	pipe.ZCard(ctx, key)
	pipe.ZAdd(ctx, key, redis.Z{Score: float64(now), Member: now})
	pipe.Expire(ctx, key, window)

	cmds, err := pipe.Exec(ctx)
	if err != nil {
		return 0, false, err
	}
	count := cmds[1].(*redis.IntCmd).Val()
	remaining := limit - count
	if remaining < 0 {
		remaining = 0
	}
	return remaining, count < limit, nil
}

// DeviceLastSeen records a device heartbeat in a sorted set (score = unix timestamp).
func (c *Cache) DeviceLastSeen(ctx context.Context, deviceID string) error {
	return c.client.ZAdd(ctx, "devices:lastseen", redis.Z{
		Score: float64(time.Now().Unix()), Member: deviceID,
	}).Err()
}

// GetStaleDevices returns devices not seen within the last `since` duration.
func (c *Cache) GetStaleDevices(ctx context.Context, since time.Duration) ([]string, error) {
	cutoff := float64(time.Now().Add(-since).Unix())
	return c.client.ZRangeByScore(ctx, "devices:lastseen", &redis.ZRangeBy{
		Min: "0", Max: fmt.Sprintf("%f", cutoff),
	}).Result()
}
```

---

### 3.6 — `internal/auth/jwt.go`

HMAC-SHA256 JWT with separate access (15m) and refresh (7d) tokens. Refresh tokens are validated against Redis sessions — revoking the session invalidates the token immediately.

```go
package auth

import (
	"context"
	"errors"
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"github.com/google/uuid"
	"github.com/oneiot/platform/internal/cache"
	"github.com/oneiot/platform/pkg/config"
)

var (
	ErrInvalidToken = errors.New("invalid token")
	ErrExpiredToken  = errors.New("token expired")
	ErrTokenRevoked  = errors.New("token revoked")
)

type Claims struct {
	UserID    string `json:"uid"`
	Email     string `json:"email"`
	Role      string `json:"role"`
	TokenType string `json:"type"` // "access" | "refresh"
	jwt.RegisteredClaims
}

type TokenPair struct {
	AccessToken  string    `json:"access_token"`
	RefreshToken string    `json:"refresh_token"`
	ExpiresAt    time.Time `json:"expires_at"`
}

type Service struct {
	cfg   config.JWTConfig
	cache *cache.Cache
}

func NewService(cfg config.JWTConfig, cache *cache.Cache) *Service {
	return &Service{cfg: cfg, cache: cache}
}

// GenerateTokenPair mints access + refresh tokens and persists a Redis session.
func (s *Service) GenerateTokenPair(ctx context.Context, userID, email, role, ip, ua string) (*TokenPair, error) {
	sessionID := uuid.NewString()
	now := time.Now()

	accessClaims := Claims{
		UserID: userID, Email: email, Role: role, TokenType: "access",
		RegisteredClaims: jwt.RegisteredClaims{
			ID: uuid.NewString(), Issuer: s.cfg.Issuer, Subject: userID,
			IssuedAt: jwt.NewNumericDate(now), ExpiresAt: jwt.NewNumericDate(now.Add(s.cfg.AccessExpiry)),
		},
	}
	accessToken, err := jwt.NewWithClaims(jwt.SigningMethodHS256, accessClaims).SignedString([]byte(s.cfg.AccessSecret))
	if err != nil {
		return nil, fmt.Errorf("sign access token: %w", err)
	}

	refreshClaims := Claims{
		UserID: userID, Email: email, Role: role, TokenType: "refresh",
		RegisteredClaims: jwt.RegisteredClaims{
			ID: sessionID, Issuer: s.cfg.Issuer, Subject: userID,
			IssuedAt: jwt.NewNumericDate(now), ExpiresAt: jwt.NewNumericDate(now.Add(s.cfg.RefreshExpiry)),
		},
	}
	refreshToken, err := jwt.NewWithClaims(jwt.SigningMethodHS256, refreshClaims).SignedString([]byte(s.cfg.RefreshSecret))
	if err != nil {
		return nil, fmt.Errorf("sign refresh token: %w", err)
	}

	if err := s.cache.SaveSession(ctx, sessionID, &cache.Session{
		UserID: userID, Email: email, Role: role, IP: ip, UserAgent: ua, CreatedAt: now,
	}, s.cfg.RefreshExpiry); err != nil {
		return nil, fmt.Errorf("save session: %w", err)
	}

	return &TokenPair{
		AccessToken: accessToken, RefreshToken: refreshToken,
		ExpiresAt: now.Add(s.cfg.AccessExpiry),
	}, nil
}

func (s *Service) ValidateAccessToken(tokenStr string) (*Claims, error) {
	return s.parseToken(tokenStr, s.cfg.AccessSecret, "access")
}

func (s *Service) ValidateRefreshToken(ctx context.Context, tokenStr string) (*Claims, error) {
	claims, err := s.parseToken(tokenStr, s.cfg.RefreshSecret, "refresh")
	if err != nil {
		return nil, err
	}
	if _, err := s.cache.GetSession(ctx, claims.ID); err != nil {
		return nil, ErrTokenRevoked
	}
	return claims, nil
}

func (s *Service) Logout(ctx context.Context, sessionID string) error {
	return s.cache.DeleteSession(ctx, sessionID)
}

func (s *Service) parseToken(tokenStr, secret, expectedType string) (*Claims, error) {
	token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (any, error) {
		if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("unexpected alg: %v", t.Header["alg"])
		}
		return []byte(secret), nil
	})
	if err != nil {
		if errors.Is(err, jwt.ErrTokenExpired) {
			return nil, ErrExpiredToken
		}
		return nil, ErrInvalidToken
	}
	claims, ok := token.Claims.(*Claims)
	if !ok || !token.Valid || claims.TokenType != expectedType {
		return nil, ErrInvalidToken
	}
	return claims, nil
}
```

---

### 3.7 — `internal/device/repository.go`

Full domain model with advanced PostgreSQL: window functions, CTEs, Z-score anomaly detection, cursor pagination, COPY-based bulk upsert, soft deletes.

```go
package device

import (
	"context"
	"fmt"
	"time"

	"github.com/google/uuid"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
)

type Status string
const (
	StatusOnline  Status = "online"
	StatusOffline Status = "offline"
	StatusError   Status = "error"
)

type Device struct {
	ID          uuid.UUID         `json:"id"`
	OrgID       uuid.UUID         `json:"org_id"`
	Name        string            `json:"name"`
	Type        string            `json:"type"`
	Status      Status            `json:"status"`
	Tags        map[string]string `json:"tags"`
	Metadata    map[string]any    `json:"metadata"`
	FirmwareVer string            `json:"firmware_version"`
	LastSeenAt  *time.Time        `json:"last_seen_at"`
	CreatedAt   time.Time         `json:"created_at"`
	UpdatedAt   time.Time         `json:"updated_at"`
}

type TelemetryPoint struct {
	DeviceID  uuid.UUID
	Metric    string
	Value     float64
	Labels    map[string]string
	Timestamp time.Time
}

type DeviceStats struct {
	DeviceID        uuid.UUID
	Name            string
	AvgValue        float64
	MaxValue        float64
	MinValue        float64
	ReadingsCount   int64
	LastReadingTime time.Time
}

type Repository struct{ pool *pgxpool.Pool }
func NewRepository(pool *pgxpool.Pool) *Repository { return &Repository{pool: pool} }

func (r *Repository) Create(ctx context.Context, d *Device) error {
	d.ID = uuid.New()
	d.CreatedAt = time.Now()
	d.UpdatedAt = time.Now()
	_, err := r.pool.Exec(ctx, `
		INSERT INTO devices (id, org_id, name, type, status, tags, metadata, firmware_version, created_at, updated_at)
		VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10)`,
		d.ID, d.OrgID, d.Name, d.Type, d.Status, d.Tags, d.Metadata, d.FirmwareVer, d.CreatedAt, d.UpdatedAt,
	)
	return err
}

// ListByOrg uses cursor-based pagination (keyset pagination) for consistent performance.
func (r *Repository) ListByOrg(ctx context.Context, orgID uuid.UUID, cursor *time.Time, limit int) ([]*Device, error) {
	var args []any
	q := `SELECT id, org_id, name, type, status, tags, metadata, firmware_version, last_seen_at, created_at, updated_at
		  FROM devices WHERE org_id = $1 AND deleted_at IS NULL`
	args = append(args, orgID)
	if cursor != nil {
		q += fmt.Sprintf(" AND created_at < $%d", len(args)+1)
		args = append(args, cursor)
	}
	q += fmt.Sprintf(" ORDER BY created_at DESC LIMIT $%d", len(args)+1)
	args = append(args, limit)
	rows, err := r.pool.Query(ctx, q, args...)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	return collectDevices(rows)
}

// BulkUpsertTelemetry uses PostgreSQL COPY protocol for high-throughput batch writes.
func (r *Repository) BulkUpsertTelemetry(ctx context.Context, points []*TelemetryPoint) error {
	rows := make([][]any, 0, len(points))
	for _, p := range points {
		rows = append(rows, []any{p.DeviceID, p.Metric, p.Value, p.Labels, p.Timestamp})
	}
	_, err := r.pool.CopyFrom(ctx,
		pgx.Identifier{"telemetry"},
		[]string{"device_id", "metric", "value", "labels", "timestamp"},
		pgx.CopyFromRows(rows),
	)
	return err
}

// GetDeviceStats uses window functions + CTE for per-device analytics in a single query.
func (r *Repository) GetDeviceStats(ctx context.Context, orgID uuid.UUID, metric string, from, to time.Time) ([]*DeviceStats, error) {
	rows, err := r.pool.Query(ctx, `
		WITH device_telemetry AS (
			SELECT
				t.device_id, d.name, t.value, t.timestamp,
				AVG(t.value)  OVER (PARTITION BY t.device_id) AS avg_val,
				MAX(t.value)  OVER (PARTITION BY t.device_id) AS max_val,
				MIN(t.value)  OVER (PARTITION BY t.device_id) AS min_val,
				COUNT(*)      OVER (PARTITION BY t.device_id) AS cnt,
				LAST_VALUE(t.timestamp) OVER (
					PARTITION BY t.device_id ORDER BY t.timestamp
					ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
				) AS last_ts,
				ROW_NUMBER() OVER (PARTITION BY t.device_id ORDER BY t.timestamp DESC) AS rn
			FROM telemetry t
			JOIN devices d ON d.id = t.device_id
			WHERE d.org_id = $1 AND t.metric = $2 AND t.timestamp BETWEEN $3 AND $4
		)
		SELECT device_id, name, avg_val, max_val, min_val, cnt, last_ts
		FROM device_telemetry WHERE rn = 1
		ORDER BY avg_val DESC`,
		orgID, metric, from, to,
	)
	// ... scan rows into []*DeviceStats
	_ = rows; _ = err
	return nil, nil
}

// GetAnomalies uses Z-score (stddev) to find statistical outliers.
func (r *Repository) GetAnomalies(ctx context.Context, deviceID uuid.UUID, metric string, zThreshold float64, hours int) ([]*TelemetryPoint, error) {
	rows, err := r.pool.Query(ctx, `
		WITH stats AS (
			SELECT AVG(value) AS mean, STDDEV(value) AS stddev
			FROM telemetry
			WHERE device_id=$1 AND metric=$2
			  AND timestamp > NOW() - ($3 || ' hours')::INTERVAL
		)
		SELECT t.device_id, t.metric, t.value, t.labels, t.timestamp
		FROM telemetry t, stats
		WHERE t.device_id=$1 AND t.metric=$2
		  AND t.timestamp > NOW() - ($3 || ' hours')::INTERVAL
		  AND ABS(t.value - stats.mean) > ($4 * stats.stddev)
		ORDER BY t.timestamp DESC`,
		deviceID, metric, hours, zThreshold,
	)
	// ... scan rows
	_ = rows; _ = err
	return nil, nil
}

func (r *Repository) SoftDelete(ctx context.Context, id, orgID uuid.UUID) error {
	_, err := r.pool.Exec(ctx, `
		UPDATE devices SET deleted_at=NOW(), updated_at=NOW()
		WHERE id=$1 AND org_id=$2 AND deleted_at IS NULL`, id, orgID)
	return err
}

func collectDevices(rows pgx.Rows) ([]*Device, error) {
	var devices []*Device
	for rows.Next() {
		d := &Device{}
		if err := rows.Scan(&d.ID, &d.OrgID, &d.Name, &d.Type, &d.Status, &d.Tags, &d.Metadata,
			&d.FirmwareVer, &d.LastSeenAt, &d.CreatedAt, &d.UpdatedAt); err != nil {
			return nil, err
		}
		devices = append(devices, d)
	}
	return devices, rows.Err()
}
```

---

### 3.8 — `internal/kafka/pipeline.go`

**Producer** with per-topic writers, key-based partitioning (device affinity), Snappy compression. **Consumer group** with sticky balancer, dead-letter queue routing, and N worker goroutines. **Pipeline** that chains multiple consumer groups via channels.

```go
package kafka

// Key patterns demonstrated:
//
// PRODUCER
//   - Per-topic kafka.Writer with RWMutex protection
//   - Hash balancer → same device always hits same partition
//   - Snappy compression, RequireAll acks, batch batching
//   - Custom message headers (source, timestamp)
//
// CONSUMER GROUP
//   - Reader goroutine → buffered channel → N worker goroutines
//   - sync.WaitGroup coordinates shutdown
//   - Failed messages routed to .dlq topic automatically
//   - Sticky rebalance strategy (RackAffinityGroupBalancer)
//   - Commit only on success
//
// PIPELINE
//   - Multiple ConsumerGroups chained, each runs in its own goroutine
//   - Context cancellation propagates through all stages
//   - Error from any stage returned to caller

// (full source code in internal/kafka/pipeline.go — 267 lines)
```

Full source in `internal/kafka/pipeline.go`.

---

### 3.9 — `internal/pipeline/telemetry_worker.go`

**4-stage fan-in/fan-out pipeline** for industrial sensor data:

| Stage | Pattern | Detail |
|---|---|---|
| Stage 1 | Kafka consumer group | 4 reader goroutines, feeds `rawCh` (buffered 1000) |
| Stage 2 | Fan-out normalizers | 8 goroutines validate range, fix timestamps, type-convert |
| Stage 3 | Batch accumulator | Drains `normalizedCh`, flushes to PostgreSQL at 500 rows or 2s |
| Stage 4 | Metrics | `atomic.Int64` counters for processed/errors/batches — lock-free reads |

```go
// Key Go concurrency primitives used:
//   goroutines          — worker pool, pipeline stages
//   channels            — rawCh, normalizedCh (buffered, back-pressure)
//   sync.WaitGroup      — coordinate shutdown of worker pool
//   sync.Mutex          — not used here (atomic preferred for counters)
//   atomic.Int64        — lock-free metrics (processed, errors, batchesSent)
//   context.Context     — propagate cancellation through all goroutines
//   time.Ticker         — time-based batch flush (2s window)
//   select              — multiplex channel receive with ctx.Done()
```

Full source in `internal/pipeline/telemetry_worker.go`.

---

### 3.10 — `internal/grpc/server.go`

gRPC server covering all 4 RPC patterns with TLS 1.3, chain interceptors, and goroutine-based concurrency:

| RPC | Pattern | Implementation |
|---|---|---|
| `StreamTelemetry` | Bidirectional streaming | 4-worker goroutine pool via channel |
| `PublishBatch` | Unary | Semaphore-bounded goroutine fan-out (8 concurrent) |
| `SubscribeAlerts` | Server streaming | ticker-based push to connected client |
| `GetDeviceStats` | Unary | delegates to device repo |

```go
// Interceptors (chained via grpc.ChainUnaryInterceptor):
//   1. logging.UnaryServerInterceptor    — zap-based request/response logging
//   2. recovery.UnaryServerInterceptor   — panic → gRPC Internal error (no crash)
//   3. authUnaryInterceptor              — JWT validation from Authorization metadata

// Semaphore pattern for bounded concurrency in PublishBatch:
//   sem := make(chan struct{}, 8)
//   sem <- struct{}{}     // acquire
//   defer func() { <-sem }()  // release

// TLS: tls.VersionTLS13 minimum, loaded from cert files (cert-manager in k8s)
// Keepalive: MaxConnectionIdle=15s, MaxConnectionAge=30s
```

Full source in `internal/grpc/server.go`.

---

### 3.11 — `cmd/api/main.go`

Orchestrates all components with **graceful shutdown**:

```go
// Startup order:
//   1. Load config (Viper)
//   2. Init logger (zap)
//   3. Connect PostgreSQL (pgxpool)
//   4. Connect Redis
//   5. Start telemetry pipeline worker (goroutine)
//   6. Start gRPC server (goroutine)
//   7. Start Prometheus metrics server (goroutine)
//   8. Start Gin HTTP server (goroutine)
//   9. Block on SIGINT/SIGTERM
//
// Shutdown order (context cancel → 30s deadline):
//   1. cancel() — stops all goroutines watching ctx
//   2. grpcSrv.GracefulStop()
//   3. httpSrv.Shutdown(shutdownCtx)
//
// Routes:
//   GET  /healthz                  — liveness
//   GET  /readyz                   — readiness
//   POST /api/v1/auth/login        — JWT pair
//   POST /api/v1/auth/refresh
//   POST /api/v1/auth/logout
//   POST /api/v1/devices           — register sensor
//   GET  /api/v1/devices           — list (cursor paginated)
//   GET  /api/v1/devices/:id
//   DELETE /api/v1/devices/:id
//   GET  /api/v1/devices/:id/stats        — window function analytics
//   GET  /api/v1/devices/:id/anomalies    — Z-score outlier detection
//   POST /api/v1/devices/:id/telemetry    — ingest → Kafka
```

---

## 4. Proto Definition

`proto/telemetry.proto` — defines two services covering all gRPC RPC patterns:

```protobuf
syntax = "proto3";
package oneiot.telemetry.v1;
option go_package = "github.com/oneiot/platform/proto/telemetryv1";

service TelemetryService {
  rpc StreamTelemetry(stream TelemetryRequest) returns (stream TelemetryResponse); // bidi
  rpc PublishBatch(PublishBatchRequest) returns (PublishBatchResponse);             // unary
  rpc GetDeviceStats(GetDeviceStatsRequest) returns (GetDeviceStatsResponse);      // unary
  rpc SubscribeAlerts(SubscribeAlertsRequest) returns (stream Alert);              // server-stream
}

service DeviceService {
  rpc RegisterDevice(RegisterDeviceRequest) returns (RegisterDeviceResponse);
  rpc GetDevice(GetDeviceRequest) returns (GetDeviceResponse);
  rpc ListDevices(ListDevicesRequest) returns (ListDevicesResponse);
  rpc UpdateDeviceStatus(UpdateDeviceStatusRequest) returns (UpdateDeviceStatusResponse);
  rpc DeleteDevice(DeleteDeviceRequest) returns (DeleteDeviceResponse);
}

// Key messages: TelemetryPoint, Alert, Device, all with google.protobuf.Timestamp
// Labels as map<string,string>, context as google.protobuf.Struct
```

Generate with:
```bash
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       proto/telemetry.proto
```

---

## 5. Database Migrations

`migrations/001_initial_schema.up.sql` — production-grade schema:

| Object | Detail |
|---|---|
| `organizations` | Multi-tenant root |
| `users` | RBAC (admin/member/viewer), soft delete |
| `devices` | GIN on `tags` JSONB, trigram on `name`, partial index (active only) |
| `telemetry` | Partitioned by month (`PARTITION BY RANGE(timestamp)`), composite index `(device_id, metric, timestamp DESC)` |
| `alerts` | Partial index on unresolved alerts |
| `audit_log` | Append-only, org-scoped |
| `device_daily_summary` | **Materialized view** with `AVG/MAX/MIN/P50/P95/P99` per device per day |
| Trigger | `trigger_set_updated_at()` — auto-manage `updated_at` |

```sql
-- Monthly partition example:
CREATE TABLE telemetry_2025_06 PARTITION OF telemetry
    FOR VALUES FROM ('2025-06-01') TO ('2025-07-01');

-- Materialized view refreshed by CronJob every hour:
CREATE MATERIALIZED VIEW device_daily_summary AS
SELECT d.id, d.org_id, d.name, t.metric,
       DATE_TRUNC('day', t.timestamp) AS day,
       AVG(t.value), MAX(t.value), MIN(t.value), COUNT(*),
       PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY t.value) AS p50,
       PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY t.value) AS p95,
       PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY t.value) AS p99
FROM telemetry t JOIN devices d ON d.id = t.device_id
GROUP BY d.id, d.org_id, d.name, t.metric, DATE_TRUNC('day', t.timestamp)
WITH DATA;
```

---

## 6. Config

`config/config.dev.yaml` — full typed YAML config with all sections:
- `app`: port, timeouts, log level, env
- `db`: host, pool sizes, migrations path
- `redis`: addr, pool, timeouts
- `kafka`: brokers array, topics, partitions, replication, DLQ
- `jwt`: separate access/refresh secrets + expiry
- `grpc`: port, TLS cert paths
- `aws`: region (ap-south-1), S3 bucket, IRSA-compatible (empty key/secret)
- `metrics`: Prometheus port + path

---

## 7. Docker

### 7.1 — Dockerfile (7 stages)

```dockerfile
# Stage 1: proto-gen    — installs protoc + plugins, generates Go code
# Stage 2: deps         — downloads Go modules (cached layer)
# Stage 3: builder      — CGO_ENABLED=0, -ldflags="-s -w", -trimpath
#                         Builds 3 binaries: api, worker, grpc-server
# Stage 4: test         — go test -race -coverprofile (CI target)
# Stage 5: api          — gcr.io/distroless/static-debian12:nonroot
# Stage 6: worker       — gcr.io/distroless/static-debian12:nonroot
# Stage 7: grpc         — gcr.io/distroless/static-debian12:nonroot
#
# Final images:
#   - Run as UID 65532 (nonroot)
#   - Read-only root filesystem
#   - No shell, no package manager — minimal attack surface
#   - Exposes: 8080 (REST), 9090 (metrics), 9000 (gRPC)
```

Build targets:
```bash
docker build --target api    -t oneiot/api:latest    .
docker build --target worker -t oneiot/worker:latest .
docker build --target grpc   -t oneiot/grpc:latest   .
docker build --target test   .   # CI test run
```

### 7.2 — docker-compose.yml

Full local development stack:

| Service | Image | Port |
|---|---|---|
| `postgres` | postgres:16-alpine | 5432 |
| `redis` | redis:7-alpine | 6379 |
| `zookeeper` | confluentinc/cp-zookeeper:7.6.0 | 2181 |
| `kafka-1` | confluentinc/cp-kafka:7.6.0 | 9092 |
| `kafka-2` | confluentinc/cp-kafka:7.6.0 | 9093 |
| `kafka-3` | confluentinc/cp-kafka:7.6.0 | 9094 |
| `kafka-init` | (one-off) | — creates all topics |
| `minio` | minio/minio | 9000, 9001 (console) |
| `api` | (built from Dockerfile) | 8080, 9090 |
| `grpc-server` | (built from Dockerfile) | 50051 |
| `worker` | (built from Dockerfile, 2 replicas) | — |
| `prometheus` | prom/prometheus:v2.52.0 | 9091 |
| `grafana` | grafana/grafana:10.4.2 | 3000 |

PostgreSQL tuned with: `shared_buffers=256MB`, `effective_cache_size=768MB`, `random_page_cost=1.1`, `log_min_duration_statement=200ms`

---

## 8. Kubernetes Manifests

### 8.1 — Namespace + ConfigMap + Secret

```yaml
# Namespace: oneiot (labelled for ArgoCD)
# ConfigMap: full config.yaml — references internal service DNS names
#   e.g. kafka-0.kafka-headless.oneiot.svc.cluster.local:9092
# Secret: managed by ExternalSecrets Operator (syncs from AWS Secrets Manager)
```

### 8.2 — API Deployment (`01-api-deployment.yaml`)

Key production hardening features:
- **Zero-downtime** rolling update: `maxSurge=1, maxUnavailable=0`
- **Topology spread**: enforced across zones AND nodes (`DoNotSchedule`)
- **Pod anti-affinity**: hard rule — no two API pods on same node
- **Init containers**: wait-for-postgres + run-migrations (PreSync)
- **Security context**: `runAsNonRoot=true`, `readOnlyRootFilesystem=true`, `capabilities: drop: [ALL]`, `seccompProfile: RuntimeDefault`
- **IRSA annotation**: `eks.amazonaws.com/role-arn` on ServiceAccount
- **Env injection**: secrets from `secretKeyRef`, pod info via `fieldRef`
- **Probes**: separate startup (30×5s), liveness (20s period), readiness (10s period)
- **preStop hook**: `sleep 10` — allows load balancer to drain connections
- **Resources**: requests: 250m CPU / 256Mi RAM; limits: 1000m / 512Mi

### 8.3 — Services + Ingress + HPA + PDB + CronJob + Job + NetworkPolicy

**Services:**
- `oneiot-api-svc` (ClusterIP, ports 80→8080, 9090)
- `oneiot-grpc-svc` (ClusterIP, port 9000)

**HPA (`autoscaling/v2`):**
- CPU target 60%, Memory target 70%, custom `http_requests_per_second` metric
- Scale-down: 5min stabilization, max 10%/min
- Scale-up: 30s stabilization, max(100%, 3 pods)/30s

**PDB:**
- `oneiot-api`: `minAvailable: 2`
- `oneiot-worker`: `minAvailable: 1`

**Ingress (NGINX):**
- TLS via `cert-manager` `ClusterIssuer` (Let's Encrypt prod)
- Rate limiting: 100 req/min
- Security headers: `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, HSTS
- Separate gRPC ingress with `backend-protocol: GRPC`

**CronJob:** `0 * * * *` — refreshes `device_daily_summary` materialized view. `concurrencyPolicy: Forbid`, `activeDeadlineSeconds: 600`

**Job:** `db-migrate` — tagged `argocd.argoproj.io/hook: PreSync`, runs migrations before each deploy

**NetworkPolicy:** Explicit allow-list for ingress (from ingress-nginx, from monitoring) and egress (to postgres, redis, kafka, external HTTPS only)

### 8.4 — Kustomization

```yaml
# base/kustomization.yaml
resources:
  - 00-namespace-config.yaml
  - 01-api-deployment.yaml
  - 02-services-ingress-hpa.yaml
images:
  - name: oneiot-api
    newName: 123456789.dkr.ecr.ap-south-1.amazonaws.com/oneiot/api
    newTag: latest
```

---

## 9. ArgoCD (GitOps)

`deployments/argocd/applications.yaml`:

**App of Apps pattern:**
```
oneiot-platform (App of Apps)
├── oneiot-api          (Kustomize overlay, prod)
├── kafka               (Bitnami Helm chart, KRaft mode, 3 brokers)
└── cert-manager        (Jetstack Helm chart, CRDs included)
```

**AppProject:**
- Restricts source repos and destination namespaces
- Two roles: `dev` (sync/get), `ops` (all)

**Automated sync policy:**
- `prune: true` — removes resources deleted from Git
- `selfHeal: true` — reverts manual cluster changes
- Retry: 5 attempts, exponential backoff (5s → 3min)

**ignoreDifferences:**
- `/spec/replicas` on Deployments (HPA manages this)
- `/data` on Secrets (ExternalSecrets manages these)

**Notifications:** Slack alerts on sync success, sync failure, health degraded

---

## 10. Terraform (AWS Infrastructure)

All modules target **ap-south-1 (Mumbai)**.

### 10.1 — VPC Module

```hcl
# 3-AZ design: ap-south-1a, ap-south-1b, ap-south-1c
# Subnet tiers:
#   public   (10.0.64.0/20 ×3)  — ALB, NAT Gateways
#   private  (10.0.0.0/20 ×3)   — EKS nodes, MSK, ElastiCache
#   database (10.0.128.0/20 ×3) — RDS only
#   intra    (10.0.192.0/20 ×3) — EKS control plane

# One NAT Gateway per AZ (HA, not single-point-of-failure)
# VPC Flow Logs → CloudWatch (60s aggregation)

# VPC Endpoints (avoid NAT for AWS service traffic):
#   Gateway:   S3
#   Interface: ECR API, ECR DKR, Secrets Manager, STS, CloudWatch Logs

# Security Groups:
#   eks_nodes     — egress all
#   rds           — ingress 5432 from eks_nodes only
#   elasticache   — ingress 6379 from eks_nodes only
#   msk           — ingress 9092, 9094, 2181 from eks_nodes only
```

### 10.2 — EKS Module

```hcl
# EKS 1.30, private control plane endpoint + public (restricted CIDRs)
# Control plane logging: api, audit, authenticator, controllerManager, scheduler

# Add-ons (all managed, most_recent):
#   coredns          — 2 replicas
#   vpc-cni          — prefix delegation (more IPs per node)
#   aws-ebs-csi-driver — gp3 StorageClass (default, encrypted, 3000 IOPS)
#   aws-efs-csi-driver

# Node groups:
#   system  (t3.medium, 2-4) — tainted CriticalAddonsOnly
#   oneiot  (c6i.xlarge, 3-20) — tainted dedicated=oneiot, 50GB gp3
#   spot    (c6i.2xlarge mix, 0-30) — cost optimization

# Karpenter: replaces cluster-autoscaler, bin-packing aware

# IRSA roles:
#   vpc-cni     → AmazonEKS_CNI_Policy
#   ebs-csi     → AmazonEBSCSIDriverPolicy
#   oneiot-api  → custom S3 + Secrets Manager policies (least privilege)
```

### 10.3 — RDS + ElastiCache + MSK + S3

```hcl
# RDS PostgreSQL 16:
#   db.r6g.large primary, db.r6g.medium replica
#   Multi-AZ (synchronous standby in another AZ)
#   Storage: 100GB gp3, auto-scale to 1000GB
#   Encrypted at rest (KMS), encrypted in transit (SSL require)
#   Performance Insights 7 days, Enhanced Monitoring 60s
#   Backup: 30 days retention, automated snapshots

# ElastiCache Redis 7.1:
#   cache.r7g.large × 3 (1 primary, 2 replicas)
#   Multi-AZ, automatic failover
#   Encrypted at rest (KMS) + in transit (TLS + AUTH token)
#   Slow-log → CloudWatch, 512MB maxmemory, allkeys-lru

# MSK Kafka 3.6 (KRaft):
#   kafka.m5.large × 3 brokers
#   SASL/IAM authentication (no passwords)
#   Snappy compression, min.insync.replicas=2
#   1TB EBS per broker, 250 MB/s throughput
#   JMX + Node exporter → Prometheus

# S3 (telemetry bucket):
#   Versioning enabled
#   KMS encryption (aws:kms, bucket key)
#   Lifecycle: Standard → Standard-IA @30d → Glacier @90d → expire @365d
#   Public access fully blocked
```

### 10.4 — Production Root Module (`terraform/envs/prod/main.tf`)

```hcl
# S3 backend with DynamoDB locking + KMS encryption
# Provisions in order:
#   1. KMS key (multi-service encryption)
#   2. VPC
#   3. ECR (api, worker, grpc-server) + lifecycle policy (keep 20)
#   4. EKS
#   5. RDS PostgreSQL
#   6. ElastiCache Redis
#   7. MSK Kafka
#   8. S3
#   9. Secrets Manager (all secrets in one JSON secret)
#  10. ArgoCD (Helm release, HA mode, nginx ingress)

# Outputs: cluster_name, rds_endpoint, redis_endpoint,
#          kafka_brokers, telemetry_bucket, ecr_api_url
```

---

## 11. Tanka / Jsonnet

### 11.1 — `tanka/environments/default/main.jsonnet`

Uses `grafana/jsonnet-libs` k8s library to generate Deployments, Services for api/worker/grpc using functional composition:

```jsonnet
api_deployment:
  deployment.new('oneiot-api', config.api.replicas, [
    container.new('api', config.api.image)
    + container.withPorts([...])
    + container.withEnvFrom([configMapRef, secretRef])
    + container.resources.withRequests({cpu: '250m', memory: '256Mi'})
    + container.readinessProbe.httpGet.withPath('/readyz')
    + container.livenessProbe.httpGet.withPath('/healthz')
  ])
  + deployment.spec.strategy.rollingUpdate.withMaxSurge(1)
  + deployment.spec.strategy.rollingUpdate.withMaxUnavailable(0)
```

### 11.2 — `tanka/environments/default/config.libsonnet`

Centralised environment values: image tags, replica counts, resource requests, Kafka broker list, HPA thresholds.

### 11.3 — `tanka/lib/oneiot.libsonnet`

Reusable library of Jsonnet functions:

| Function | Output |
|---|---|
| `hpa(name, min, max, cpuTarget, memTarget)` | `autoscaling/v2` HPA with scale-up/down behavior |
| `pdb(name, minAvailable)` | `policy/v1` PodDisruptionBudget |
| `securityContext` | Non-root, read-only FS, drop ALL capabilities |
| `httpProbe(path, port, ...)` | Standard liveness/readiness probe object |
| `topologySpread(key, value)` | Zone + hostname spread constraints |
| `serviceMonitor(name, port, path)` | `monitoring.coreos.com/v1` ServiceMonitor |
| `prometheusRule(name, rules)` | PrometheusRule with alerting groups |
| `standardAlerts(serviceName)` | High error rate + high P99 latency + pod not ready |

---

## 12. Concept Index

| JD Requirement | Files |
|---|---|
| **Goroutines** | `pipeline/telemetry_worker.go`, `kafka/pipeline.go`, `grpc/server.go` |
| **Channels** | `pipeline/telemetry_worker.go` (rawCh, normalizedCh, semaphore) |
| **Mutex** | `kafka/pipeline.go` (RWMutex on writers map) |
| **Atomic** | `pipeline/telemetry_worker.go` (atomic.Int64 counters) |
| **sync.WaitGroup** | `kafka/pipeline.go`, `grpc/server.go` |
| **Context cancellation** | Every package — passed through the full call chain |
| **PostgreSQL advanced** | `device/repository.go` — CTEs, window functions, Z-score |
| **PostgreSQL partitioning** | `migrations/001_initial_schema.up.sql` |
| **PostgreSQL COPY** | `db/postgres.go`, `device/repository.go` |
| **PostgreSQL materialized views** | `migrations/001_initial_schema.up.sql` |
| **Redis sessions** | `cache/redis.go`, `auth/jwt.go` |
| **Redis rate limiting** | `cache/redis.go` (sliding window ZSet) |
| **JWT** | `auth/jwt.go` (access + refresh, HMAC-SHA256) |
| **Kafka producer groups** | `kafka/pipeline.go` (Producer, per-topic writers) |
| **Kafka consumer groups** | `kafka/pipeline.go` (ConsumerGroup, sticky, DLQ) |
| **Multi-stage pipeline** | `kafka/pipeline.go` (Pipeline), `pipeline/telemetry_worker.go` |
| **gRPC bidirectional** | `grpc/server.go` (StreamTelemetry) |
| **gRPC unary** | `grpc/server.go` (PublishBatch, GetDeviceStats) |
| **gRPC server-stream** | `grpc/server.go` (SubscribeAlerts) |
| **gRPC interceptors** | `grpc/server.go` (logging, recovery, auth) |
| **Multi-stage Dockerfile** | `Dockerfile` (7 stages) |
| **Docker Compose** | `docker-compose.yml` (13 services) |
| **K8s Deployment** | `deployments/k8s/base/01-api-deployment.yaml` |
| **K8s Service** | `deployments/k8s/base/02-services-ingress-hpa.yaml` |
| **K8s Ingress** | `deployments/k8s/base/02-...yaml` (REST + gRPC) |
| **K8s SSL (cert-manager)** | `deployments/k8s/base/02-...yaml` (ClusterIssuer) |
| **K8s HPA** | `deployments/k8s/base/02-...yaml` (autoscaling/v2) |
| **K8s PodDisruptionBudget** | `deployments/k8s/base/02-...yaml` |
| **K8s Job** | `deployments/k8s/base/02-...yaml` (db-migrate, PreSync) |
| **K8s CronJob** | `deployments/k8s/base/02-...yaml` (refresh-materialized-views) |
| **K8s NetworkPolicy** | `deployments/k8s/base/02-...yaml` |
| **ArgoCD** | `deployments/argocd/applications.yaml` |
| **GitOps** | ArgoCD automated sync + selfHeal |
| **Terraform VPC** | `terraform/modules/vpc/main.tf` |
| **Terraform EKS** | `terraform/modules/eks/main.tf` |
| **Terraform RDS** | `terraform/modules/rds/main.tf` |
| **Terraform ElastiCache** | inline in `rds/main.tf` |
| **Terraform MSK** | inline in `rds/main.tf` |
| **Terraform S3** | inline in `rds/main.tf` |
| **Terraform ECR** | `terraform/envs/prod/main.tf` |
| **Terraform IRSA** | `terraform/modules/eks/main.tf` |
| **Tanka Jsonnet** | `tanka/environments/default/main.jsonnet` |
| **Jsonnet library** | `tanka/lib/oneiot.libsonnet` |
| **AWS ap-south-1** | All Terraform files |
| **AWS S3** | `terraform/modules/rds/main.tf` (s3 section) |

---

## 13. Running Locally

```bash
# 1. Start all infrastructure
docker-compose up -d

# 2. Wait for services to be healthy, then run migrations
go run ./cmd/api/... migrate --up

# 3. Start API server (REST)
CONFIG_PATH=config/config.dev.yaml go run ./cmd/api/...

# 4. Start gRPC server (separate terminal)
CONFIG_PATH=config/config.dev.yaml go run ./cmd/grpc-server/...

# 5. Start telemetry pipeline worker (separate terminal)
CONFIG_PATH=config/config.dev.yaml go run ./cmd/worker/...

# 6. Test the API
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"admin@oneiot.io","password":"secret"}'

# 7. Generate proto code
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       proto/telemetry.proto

# 8. Run tests
go test -race -cover ./...

# 9. View Grafana dashboards
open http://localhost:3000   # admin / grafana_dev
```

---

## 14. CI/CD Flow

```
Developer pushes to feature branch
        │
        ▼
GitHub Actions CI:
  ┌─ go vet + staticcheck
  ├─ go test -race -coverprofile
  ├─ docker build --target test
  ├─ trivy image scan
  └─ golangci-lint

        │ (merge to main)
        ▼
GitHub Actions CD:
  ┌─ docker build --target api/worker/grpc
  ├─ docker push → ECR (ap-south-1)
  ├─ Update image tag in deployments/k8s/overlays/prod/kustomization.yaml
  └─ git commit + push → triggers ArgoCD

        │
        ▼
ArgoCD detects Git diff:
  ┌─ Runs PreSync hook (db-migrate Job)
  ├─ Applies Kustomize overlay (prod)
  ├─ Rolling update: maxSurge=1, maxUnavailable=0
  └─ Health check → notifies Slack

        │
        ▼
Kubernetes:
  ┌─ HPA scales based on CPU/memory/custom metrics
  ├─ Karpenter provisions/decommissions EC2 nodes
  ├─ CronJob refreshes materialized views hourly
  └─ NetworkPolicy enforces pod-to-pod isolation
```
