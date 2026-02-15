# effectful-ecosystem

Production-ready effect interpreters for the
[effectful](https://github.com/haskell-effectful/effectful) effect system.

**Stop reinventing the wheel.** This library provides battle-tested,
composable effects for common application concerns.

## The Problem

Every Haskell application using effect systems ends up writing the same
boilerplate effects: database access, HTTP clients, logging, configuration,
metrics. This fragments the ecosystem and prevents library interop.

`effectful-ecosystem` provides a **standard set of effects** with
**multiple interchangeable interpreters**.

## Package Structure

```
effectful-ecosystem/
├── effectful-db/          -- Database effects
├── effectful-http/        -- HTTP client effects
├── effectful-log/         -- Structured logging
├── effectful-config/      -- Configuration management
├── effectful-metrics/     -- Metrics & observability
├── effectful-cache/       -- Caching effects
├── effectful-retry/       -- Retry with backoff
├── effectful-auth/        -- Authentication effects
└── effectful-testing/     -- Test interpreters for all effects
```

## Quick Start

```haskell
{-# LANGUAGE TypeFamilies #-}
import Effectful
import Effectful.Db
import Effectful.Http
import Effectful.Log

-- Define your business logic using effects
getUser
  :: (Db :> es, Http :> es, Log :> es)
  => UserId -> Eff es User
getUser uid = do
  logInfo "Fetching user" ["userId" .= uid]

  -- Try cache/DB first
  mUser <- dbQuery (selectUser uid)
  case mUser of
    Just user -> pure user
    Nothing -> do
      logWarn "User not in DB, trying external API" []
      resp <- httpGet (userApiUrl uid)
      let user = parseUser (responseBody resp)
      dbExecute (insertUser user)
      pure user

-- Wire up with real interpreters
main :: IO ()
main = do
  pool <- createPool defaultPostgresConfig
  manager <- newTlsManager

  result <- runEff
    . runDbPool pool
    . runHttpManager manager
    . runLogStdout LevelInfo
    $ getUser (UserId 42)

  print result

-- Wire up with test interpreters
test :: IO ()
test = do
  result <- runEff
    . runDbPure (Map.singleton 42 testUser)
    . runHttpPure [mockResponse 200 userJson]
    . runLogSilent
    $ getUser (UserId 42)

  assertEqual result testUser
```

## effectful-db

### Effect Definition

```haskell
data Db :: Effect where
  DbQuery      :: (FromRow a) => Query -> Db m [a]
  DbQueryOne   :: (FromRow a) => Query -> Db m (Maybe a)
  DbExecute    :: Query -> Db m Int64
  DbTransaction :: m a -> Db m a

type instance DispatchOf Db = 'Dynamic
```

### Interpreters

```haskell
-- PostgreSQL via postgresql-simple
runDbPostgres :: Connection -> Eff (Db : es) a -> Eff es a

-- PostgreSQL connection pool
runDbPool :: Pool Connection -> Eff (Db : es) a -> Eff es a

-- SQLite via direct-sqlite
runDbSqlite :: SQLiteConnection -> Eff (Db : es) a -> Eff es a

-- Pure interpreter for testing
runDbPure :: Map Query [row] -> Eff (Db : es) a -> Eff es a

-- Recording interpreter (captures all queries)
runDbRecording :: IORef [Query] -> Connection -> Eff (Db : es) a -> Eff es a
```

## effectful-http

### Effect Definition

```haskell
data Http :: Effect where
  HttpRequest :: Request -> Http m Response
  HttpStream  :: Request -> (ConduitT ByteString Void m a) -> Http m a

type instance DispatchOf Http = 'Dynamic
```

### Convenience functions

```haskell
httpGet    :: Http :> es => Url -> Eff es Response
httpPost   :: Http :> es => Url -> RequestBody -> Eff es Response
httpPut    :: Http :> es => Url -> RequestBody -> Eff es Response
httpDelete :: Http :> es => Url -> Eff es Response

httpGetJSON :: (Http :> es, FromJSON a) => Url -> Eff es a
httpPostJSON :: (Http :> es, ToJSON req, FromJSON resp)
             => Url -> req -> Eff es resp
```

### Interpreters

```haskell
-- Real HTTP via http-client
runHttpManager :: Manager -> Eff (Http : es) a -> Eff es a

-- Mock responses for testing
runHttpPure :: [Response] -> Eff (Http : es) a -> Eff es a

-- Pattern-matching mock
runHttpMock :: (Request -> Response) -> Eff (Http : es) a -> Eff es a

-- Recording (capture all requests)
runHttpRecording :: IORef [Request] -> Manager -> Eff (Http : es) a -> Eff es a
```

## effectful-log

### Effect Definition

```haskell
data Log :: Effect where
  LogMessage :: LogLevel -> Text -> [LogField] -> Log m ()
  LogScope   :: Text -> m a -> Log m a

type instance DispatchOf Log = 'Dynamic

data LogLevel = LevelDebug | LevelInfo | LevelWarn | LevelError
  deriving (Show, Eq, Ord)

data LogField = LogField !Text !LogValue

(.=) :: ToJSON v => Text -> v -> LogField

logDebug, logInfo, logWarn, logError
  :: Log :> es => Text -> [LogField] -> Eff es ()

withLogScope :: Log :> es => Text -> Eff es a -> Eff es a
withLogScope = send . LogScope
```

### Interpreters

```haskell
-- stdout with formatting
runLogStdout :: LogLevel -> Eff (Log : es) a -> Eff es a

-- JSON structured logging to handle
runLogJson :: Handle -> LogLevel -> Eff (Log : es) a -> Eff es a

-- To fast-logger backend
runLogFastLogger :: TimedFastLogger -> LogLevel -> Eff (Log : es) a -> Eff es a

-- Silent (testing)
runLogSilent :: Eff (Log : es) a -> Eff es a

-- Capture logs for assertion (testing)
runLogCapture :: IORef [LogEntry] -> Eff (Log : es) a -> Eff es a
```

## effectful-config

### Effect Definition

```haskell
data Config :: Effect where
  GetConfig :: (FromJSON a, Typeable a) => ConfigKey -> Config m a
  GetConfigOpt :: (FromJSON a, Typeable a) => ConfigKey -> Config m (Maybe a)

type instance DispatchOf Config = 'Dynamic

newtype ConfigKey = ConfigKey Text

-- | Type-safe configuration declarations
data AppConfig = AppConfig
  { dbHost   :: Text
  , dbPort   :: Int
  , logLevel :: LogLevel
  } deriving (Generic, FromJSON)
```

### Interpreters

```haskell
-- From environment variables
runConfigEnv :: Eff (Config : es) a -> Eff es a

-- From YAML/JSON file
runConfigFile :: FilePath -> Eff (Config : es) a -> Eff es a

-- From pure map (testing)
runConfigPure :: Map ConfigKey Value -> Eff (Config : es) a -> Eff es a

-- Layered: file + env override
runConfigLayered :: FilePath -> Eff (Config : es) a -> Eff es a
```

## effectful-metrics

### Effect Definition

```haskell
data Metrics :: Effect where
  IncrCounter   :: CounterName -> Int -> Metrics m ()
  SetGauge      :: GaugeName -> Double -> Metrics m ()
  ObserveHisto  :: HistogramName -> Double -> Metrics m ()
  TimeAction    :: HistogramName -> m a -> Metrics m a

type instance DispatchOf Metrics = 'Dynamic
```

### Interpreters

```haskell
-- Prometheus
runMetricsPrometheus :: PrometheusRegistry -> Eff (Metrics : es) a -> Eff es a

-- StatsD
runMetricsStatsD :: StatsDConfig -> Eff (Metrics : es) a -> Eff es a

-- In-memory (testing)
runMetricsPure :: IORef MetricsStore -> Eff (Metrics : es) a -> Eff es a

-- No-op
runMetricsNoop :: Eff (Metrics : es) a -> Eff es a
```

## effectful-cache

```haskell
data Cache :: Effect where
  CacheLookup :: (Typeable a) => CacheKey -> Cache m (Maybe a)
  CacheInsert :: (Typeable a) => CacheKey -> a -> TTL -> Cache m ()
  CacheDelete :: CacheKey -> Cache m ()
  CacheWith   :: (Typeable a) => CacheKey -> TTL -> m a -> Cache m a

type instance DispatchOf Cache = 'Dynamic

-- Interpreters
runCacheRedis  :: RedisConnection -> Eff (Cache : es) a -> Eff es a
runCacheLRU    :: Int -> Eff (Cache : es) a -> Eff es a  -- in-memory LRU
runCacheNoop   :: Eff (Cache : es) a -> Eff es a          -- no caching
```

## effectful-retry

```haskell
data Retry :: Effect where
  WithRetry :: RetryPolicy -> m a -> Retry m a

type instance DispatchOf Retry = 'Dynamic

data RetryPolicy = RetryPolicy
  { rpMaxAttempts    :: !Int
  , rpBaseDelay      :: !Duration
  , rpBackoffStrategy :: !BackoffStrategy
  , rpRetryOn        :: SomeException -> Bool
  }

data BackoffStrategy
  = ConstantBackoff
  | ExponentialBackoff { maxDelay :: !Duration }
  | FullJitter { maxDelay :: !Duration }

exponentialBackoff :: Int -> Duration -> RetryPolicy
withRetry :: Retry :> es => RetryPolicy -> Eff es a -> Eff es a
```

## effectful-testing

```haskell
-- Convenient test runner combining all pure interpreters
runTestEff
  :: TestConfig
  -> Eff '[Db, Http, Log, Config, Metrics, Cache, IOE] a
  -> IO (a, TestCaptures)

data TestCaptures = TestCaptures
  { capturedQueries  :: [Query]
  , capturedRequests :: [Request]
  , capturedLogs     :: [LogEntry]
  , capturedMetrics  :: MetricsStore
  }
```

## Installation

```cabal
-- Pick what you need
build-depends:
    effectful-db      >= 0.1
  , effectful-http    >= 0.1
  , effectful-log     >= 0.1
  , effectful-config  >= 0.1
  , effectful-metrics >= 0.1
  , effectful-cache   >= 0.1
  , effectful-retry   >= 0.1
  , effectful-testing >= 0.1

-- Or get everything
build-depends: effectful-ecosystem >= 0.1
```
