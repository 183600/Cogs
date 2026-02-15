注意:README.md和PLAN.md里的代码有可能不是最好的
# effectful-ecosystem — Implementation Plan

## Architecture Principles

1. **Effect definition and interpreters are separate packages**
   - `effectful-db-core`: effect types only (minimal deps)
   - `effectful-db-postgres`: PostgreSQL interpreter
   - `effectful-db-sqlite`: SQLite interpreter
   - `effectful-db-testing`: pure/mock interpreters

2. **Every effect has at least one pure/mock interpreter**

3. **Interpreters are thin wrappers** over existing Haskell libraries

4. **All effects are composable** — no conflicts between interpreters

## Package Dependency Graph

```
effectful-core (from upstream)
    │
    ├── effectful-db-core
    │   ├── effectful-db-postgres  (depends: postgresql-simple)
    │   ├── effectful-db-sqlite    (depends: direct-sqlite)
    │   └── effectful-db-testing
    │
    ├── effectful-http-core
    │   ├── effectful-http-client   (depends: http-client, http-client-tls)
    │   └── effectful-http-testing
    │
    ├── effectful-log-core
    │   ├── effectful-log-stdout
    │   ├── effectful-log-fast-logger (depends: fast-logger)
    │   └── effectful-log-testing
    │
    ├── effectful-config-core
    │   ├── effectful-config-env
    │   ├── effectful-config-yaml  (depends: yaml)
    │   └── effectful-config-testing
    │
    ├── effectful-metrics-core
    │   ├── effectful-metrics-prometheus (depends: prometheus-client)
    │   ├── effectful-metrics-statsd
    │   └── effectful-metrics-testing
    │
    ├── effectful-cache-core
    │   ├── effectful-cache-redis   (depends: hedis)
    │   ├── effectful-cache-lru
    │   └── effectful-cache-testing
    │
    ├── effectful-retry-core
    │
    └── effectful-testing (meta-package)
```

## Module Structure (per effect)

```
effectful-db-core/
  src/
    Effectful/
      Db.hs                    -- Re-export
      Db/
        Effect.hs              -- Effect definition
        Types.hs               -- Query, Row types
        DSL.hs                 -- Convenience functions

effectful-db-postgres/
  src/
    Effectful/
      Db/
        Postgres.hs            -- runDbPostgres, runDbPool
        Postgres/
          Internal.hs          -- Implementation details

effectful-db-testing/
  src/
    Effectful/
      Db/
        Testing.hs             -- runDbPure, runDbRecording
```

## Phase 1: effectful-log (Week 1-2)

Starting with logging because it's the simplest and most universally needed.

### Effect Definition

```haskell
module Effectful.Log.Effect where

import Effectful
import Effectful.Dispatch.Dynamic
import Data.Text (Text)
import Data.Aeson (Value, ToJSON, toJSON)
import Data.Time (UTCTime)

-- | Core logging effect
data Log :: Effect where
  LogMessage :: CallStack -> LogLevel -> Text -> [LogField] -> Log m ()
  WithLogContext :: [LogField] -> m a -> Log m a
  GetLogContext :: Log m [LogField]

type instance DispatchOf Log = 'Dynamic

-- | Log levels
data LogLevel
  = LevelTrace
  | LevelDebug
  | LevelInfo
  | LevelWarn
  | LevelError
  | LevelFatal
  deriving (Show, Eq, Ord, Bounded, Enum)

-- | Structured log field
data LogField = LogField
  { logFieldKey   :: !Text
  , logFieldValue :: !Value
  } deriving (Show, Eq)

-- | Structured log entry (for capture/testing)
data LogEntry = LogEntry
  { leTimestamp :: !UTCTime
  , leLevel     :: !LogLevel
  , leMessage   :: !Text
  , leFields    :: ![LogField]
  , leContext   :: ![LogField]
  , leCallStack :: !CallStack
  } deriving (Show)

-- | Create a log field
(.=) :: ToJSON v => Text -> v -> LogField
k .= v = LogField k (toJSON v)

-- | Convenience functions
logTrace, logDebug, logInfo, logWarn, logError, logFatal
  :: (HasCallStack, Log :> es) => Text -> [LogField] -> Eff es ()
logTrace msg fields = withFrozenCallStack $
  send (LogMessage callStack LevelTrace msg fields)
logDebug msg fields = withFrozenCallStack $
  send (LogMessage callStack LevelDebug msg fields)
logInfo msg fields = withFrozenCallStack $
  send (LogMessage callStack LevelInfo msg fields)
logWarn msg fields = withFrozenCallStack $
  send (LogMessage callStack LevelWarn msg fields)
logError msg fields = withFrozenCallStack $
  send (LogMessage callStack LevelError msg fields)
logFatal msg fields = withFrozenCallStack $
  send (LogMessage callStack LevelFatal msg fields)

-- | Add context fields to all log messages within a scope
withContext :: Log :> es => [LogField] -> Eff es a -> Eff es a
withContext fields = send . WithLogContext fields
```

### Stdout Interpreter

```haskell
module Effectful.Log.Stdout where

import Effectful
import Effectful.Dispatch.Dynamic
import Effectful.Log.Effect
import Data.Time (getCurrentTime)
import Data.IORef

runLogStdout :: IOE :> es => LogLevel -> Eff (Log : es) a -> Eff es a
runLogStdout minLevel = interpret $ \env -> \case
  LogMessage cs level msg fields -> do
    when (level >= minLevel) $ liftIO $ do
      now <- getCurrentTime
      let formatted = formatLogEntry now level msg fields
      putStrLn formatted

  WithLogContext fields action -> do
    -- Thread context through local state
    localSeqUnlift env $ \unlift -> do
      unlift action  -- simplified; real impl threads context

  GetLogContext -> pure []

formatLogEntry :: UTCTime -> LogLevel -> Text -> [LogField] -> String
formatLogEntry time level msg fields =
  unwords
    [ show time
    , "[" ++ show level ++ "]"
    , Text.unpack msg
    , if null fields then ""
      else show (map (\(LogField k v) -> (k, v)) fields)
    ]
```

### Test Interpreter

```haskell
module Effectful.Log.Testing where

import Effectful
import Effectful.Dispatch.Dynamic
import Effectful.Log.Effect
import Data.IORef

-- | Capture all log entries for testing assertions
runLogCapture
  :: IOE :> es
  => IORef [LogEntry]
  -> Eff (Log : es) a -> Eff es a
runLogCapture ref = interpret $ \env -> \case
  LogMessage cs level msg fields -> liftIO $ do
    now <- getCurrentTime
    let entry = LogEntry now level msg fields [] cs
    atomicModifyIORef' ref (\es -> (es ++ [entry], ()))

  WithLogContext _fields action ->
    localSeqUnlift env $ \unlift -> unlift action

  GetLogContext -> pure []

-- | Discard all logs
runLogSilent :: Eff (Log : es) a -> Eff es a
runLogSilent = interpret $ \env -> \case
  LogMessage _ _ _ _ -> pure ()
  WithLogContext _ action ->
    localSeqUnlift env $ \unlift -> unlift action
  GetLogContext -> pure []

-- | Assert that a specific log was emitted
assertLogged :: [LogEntry] -> LogLevel -> Text -> Bool
assertLogged entries level msg =
  any (\e -> leLevel e == level && leMessage e == msg) entries
```

## Phase 2: effectful-db (Week 3-4)

### Effect Definition

```haskell
module Effectful.Db.Effect where

import Effectful
import Effectful.Dispatch.Dynamic
import Data.Int (Int64)

-- | SQL query representation (backend-agnostic)
data SqlQuery = SqlQuery
  { sqlText   :: !Text
  , sqlParams :: ![SqlValue]
  } deriving (Show)

data SqlValue
  = SqlText !Text
  | SqlInt !Int64
  | SqlDouble !Double
  | SqlBool !Bool
  | SqlNull
  | SqlBlob !ByteString
  deriving (Show, Eq)

-- | Core database effect
data Db :: Effect where
  DbQuery       :: FromRow a => SqlQuery -> Db m [a]
  DbQueryOne    :: FromRow a => SqlQuery -> Db m (Maybe a)
  DbExecute     :: SqlQuery -> Db m Int64
  DbExecuteMany :: SqlQuery -> [[SqlValue]] -> Db m Int64
  DbWithTransaction :: m a -> Db m a

type instance DispatchOf Db = 'Dynamic

-- | Type class for deserializing rows
class FromRow a where
  fromRow :: [SqlValue] -> Either String a

class ToRow a where
  toRow :: a -> [SqlValue]

-- | Convenience DSL
query :: (Db :> es, FromRow a) => SqlQuery -> Eff es [a]
query = send . DbQuery

queryOne :: (Db :> es, FromRow a) => SqlQuery -> Eff es (Maybe a)
queryOne = send . DbQueryOne

execute :: Db :> es => SqlQuery -> Eff es Int64
execute = send . DbExecute

withTransaction :: Db :> es => Eff es a -> Eff es a
withTransaction action = send (DbWithTransaction action)

-- | Template for SQL queries
sql :: QuasiQuoter  -- future: type-safe SQL quasi-quoter
sql = undefined
```

### PostgreSQL Interpreter

```haskell
module Effectful.Db.Postgres where

import Effectful
import Effectful.Dispatch.Dynamic
import Effectful.Db.Effect
import qualified Database.PostgreSQL.Simple as PG
import Data.Pool (Pool, withResource)

-- | Run with a single connection
runDbPostgres :: IOE :> es => PG.Connection -> Eff (Db : es) a -> Eff es a
runDbPostgres conn = interpret $ \env -> \case
  DbQuery q -> liftIO $ do
    PG.query conn (toPgQuery q) (toPgParams q)

  DbQueryOne q -> liftIO $ do
    rows <- PG.query conn (toPgQuery q) (toPgParams q)
    pure (listToMaybe rows)

  DbExecute q -> liftIO $ do
    PG.execute conn (toPgQuery q) (toPgParams q)

  DbExecuteMany q params -> liftIO $ do
    PG.executeMany conn (toPgQuery q) (map toPgParams' params)

  DbWithTransaction action ->
    localSeqUnlift env $ \unlift -> liftIO $
      PG.withTransaction conn (unlift action)

-- | Run with a connection pool
runDbPool :: IOE :> es => Pool PG.Connection -> Eff (Db : es) a -> Eff es a
runDbPool pool action = liftIO $
  withResource pool $ \conn ->
    runEff . runDbPostgres conn $ action  -- simplified

-- Conversion helpers
toPgQuery :: SqlQuery -> PG.Query
toPgQuery = PG.Query . encodeUtf8 . sqlText

toPgParams :: SqlQuery -> [PG.Action]
toPgParams q = map toPgAction (sqlParams q)

toPgAction :: SqlValue -> PG.Action
toPgAction (SqlText t)   = PG.toField t
toPgAction (SqlInt i)    = PG.toField i
toPgAction (SqlDouble d) = PG.toField d
toPgAction (SqlBool b)   = PG.toField b
toPgAction SqlNull       = PG.toField PG.Null
toPgAction (SqlBlob bs)  = PG.toField (PG.Binary bs)
```

### Pure Test Interpreter

```haskell
module Effectful.Db.Testing where

import Effectful
import Effectful.Dispatch.Dynamic
import Effectful.Db.Effect
import Data.Map.Strict (Map)
import Data.IORef

-- | Pure interpreter backed by an in-memory map
-- Maps query text → list of result rows
runDbPure
  :: IOE :> es
  => IORef (Map Text [[SqlValue]])
  -> Eff (Db : es) a
  -> Eff es a
runDbPure storeRef = interpret $ \env -> \case
  DbQuery q -> liftIO $ do
    store <- readIORef storeRef
    case Map.lookup (sqlText q) store of
      Just rows -> case traverse fromRow rows of
        Right results -> pure results
        Left err      -> error $ "fromRow failed: " ++ err
      Nothing -> pure []

  DbQueryOne q -> liftIO $ do
    store <- readIORef storeRef
    case Map.lookup (sqlText q) store of
      Just (row:_) -> case fromRow row of
        Right result -> pure (Just result)
        Left err     -> error $ "fromRow failed: " ++ err
      _ -> pure Nothing

  DbExecute _q -> pure 1

  DbExecuteMany _q params -> pure (fromIntegral (length params))

  DbWithTransaction action ->
    localSeqUnlift env $ \unlift -> unlift action

-- | Recording interpreter: wraps a real interpreter and captures queries
runDbRecording
  :: (Db :> es, IOE :> es)
  => IORef [SqlQuery]
  -> Eff es a
  -> Eff es a
runDbRecording ref action = do
  -- This is a middleware-style interpreter
  -- In practice, implemented via interpose
  undefined
```

## Phase 3: effectful-http (Week 5)

### Effect & Interpreters

```haskell
module Effectful.Http.Effect where

data Http :: Effect where
  HttpRequest :: HttpRequest -> Http m HttpResponse
  -- Streaming omitted for v1

type instance DispatchOf Http = 'Dynamic

data HttpRequest = HttpRequest
  { reqMethod  :: !Method
  , reqUrl     :: !Text
  , reqHeaders :: ![(HeaderName, ByteString)]
  , reqBody    :: !(Maybe RequestBody)
  , reqTimeout :: !(Maybe Int)  -- microseconds
  } deriving (Show)

data HttpResponse = HttpResponse
  { respStatus  :: !Int
  , respHeaders :: ![(HeaderName, ByteString)]
  , respBody    :: !LByteString
  } deriving (Show)

data Method = GET | POST | PUT | DELETE | PATCH | HEAD | OPTIONS
  deriving (Show, Eq)

-- Convenience
httpGet :: Http :> es => Text -> Eff es HttpResponse
httpGet url = send $ HttpRequest (HttpRequest GET url [] Nothing Nothing)

httpPostJSON :: (Http :> es, ToJSON a) => Text -> a -> Eff es HttpResponse
httpPostJSON url body = send $ HttpRequest HttpRequest
  { reqMethod = POST
  , reqUrl = url
  , reqHeaders = [("Content-Type", "application/json")]
  , reqBody = Just (RequestBodyLBS (encode body))
  , reqTimeout = Nothing
  }

httpGetJSON :: (Http :> es, FromJSON a) => Text -> Eff es a
httpGetJSON url = do
  resp <- httpGet url
  case eitherDecode (respBody resp) of
    Right a  -> pure a
    Left err -> throwIO (HttpDecodeError err)
```

```haskell
module Effectful.Http.Client where

import qualified Network.HTTP.Client as HC

runHttpManager :: IOE :> es => HC.Manager -> Eff (Http : es) a -> Eff es a
runHttpManager mgr = interpret $ \_ -> \case
  HttpRequest req -> liftIO $ do
    hcReq <- toHttpClientRequest req
    resp <- HC.httpLbs hcReq mgr
    pure (fromHttpClientResponse resp)

module Effectful.Http.Testing where

-- Pattern-matching mock
runHttpMock
  :: IOE :> es
  => (HttpRequest -> IO HttpResponse)
  -> Eff (Http : es) a -> Eff es a
runHttpMock handler = interpret $ \_ -> \case
  HttpRequest req -> liftIO $ handler req

-- Sequential responses
runHttpPure :: IOE :> es => [HttpResponse] -> Eff (Http : es) a -> Eff es a
runHttpPure responses = do
  ref <- liftIO $ newIORef responses
  interpret (\_ -> \case
    HttpRequest _ -> liftIO $ atomicModifyIORef' ref $ \case
      []     -> error "No more mock responses"
      (r:rs) -> (rs, r)
    ) action
```

## Phase 4: effectful-config (Week 6)

```haskell
module Effectful.Config.Effect where

data Config :: Effect where
  LookupConfig :: Text -> Config m (Maybe Text)
  RequireConfig :: Text -> Config m Text

type instance DispatchOf Config = 'Dynamic

-- Type-safe helpers via FromJSON
getConfig :: (Config :> es, FromJSON a) => Text -> Eff es a
getConfig key = do
  raw <- send (RequireConfig key)
  case eitherDecodeStrict (encodeUtf8 raw) of
    Right a  -> pure a
    Left err -> throwIO (ConfigParseError key err)

getConfigDefault :: (Config :> es, FromJSON a) => Text -> a -> Eff es a
getConfigDefault key def = do
  mRaw <- send (LookupConfig key)
  case mRaw of
    Nothing  -> pure def
    Just raw -> case eitherDecodeStrict (encodeUtf8 raw) of
      Right a  -> pure a
      Left err -> throwIO (ConfigParseError key err)
```

## Phase 5: effectful-metrics & effectful-cache & effectful-retry (Week 7-8)

Similar patterns as above. Each follows:
1. Minimal effect type in `-core`
2. Real interpreter wrapping proven library
3. Pure/mock interpreter for testing

## Phase 6: effectful-testing meta-package (Week 9)

```haskell
module Effectful.Testing where

-- | Run a full application stack with all test interpreters
data TestEnv = TestEnv
  { testDbStore   :: IORef (Map Text [[SqlValue]])
  , testHttpMock  :: HttpRequest -> IO HttpResponse
  , testLogs      :: IORef [LogEntry]
  , testMetrics   :: IORef MetricsStore
  , testConfig    :: Map Text Text
  }

defaultTestEnv :: IO TestEnv

runTestStack
  :: TestEnv
  -> Eff '[Db, Http, Log, Config, Metrics, IOE] a
  -> IO (a, TestCaptures)
runTestStack env action = do
  captures <- newTestCaptures
  result <- runEff
    . runDbPure (testDbStore env)
    . runHttpMock (testHttpMock env)
    . runLogCapture (testLogs env)
    . runConfigPure (testConfig env)
    . runMetricsPure (testMetrics env)
    $ action
  caps <- readTestCaptures captures
  pure (result, caps)
```

## Testing Strategy

### Per-effect tests

```haskell
-- Each interpreter gets tested against a common test suite

class DbInterpreterTest where
  -- These tests run against ANY Db interpreter
  test_queryReturnsInserted :: Eff (Db : es) Property
  test_transactionRollback :: Eff (Db : es) Property
  test_emptyQueryReturnsEmpty :: Eff (Db : es) Property

-- Run against pure interpreter
pureTests :: TestTree
pureTests = withDbPure $ runDbTests

-- Run against real postgres (integration)
integrationTests :: TestTree
integrationTests = withDbPostgres testConnStr $ runDbTests
```

### Interop tests

```haskell
-- Test that effects compose correctly
test_dbAndLogCompose :: IO ()
test_dbAndLogCompose = do
  logs <- newIORef []
  db   <- newIORef Map.empty
  result <- runEff
    . runLogCapture logs
    . runDbPure db
    $ do
        logInfo "Starting query" []
        users <- query (SqlQuery "SELECT * FROM users" [])
        logInfo "Got users" ["count" .= length users]
        pure users
  -- Verify logs captured correctly alongside DB operations
  capturedLogs <- readIORef logs
  assertEqual 2 (length capturedLogs)
```

## Milestones

| Week | Deliverable |
|------|-------------|
| 1-2  | effectful-log: effect + stdout + testing interpreters |
| 3-4  | effectful-db: effect + postgres + testing interpreters |
| 5    | effectful-http: effect + http-client + testing |
| 6    | effectful-config: effect + env + yaml + testing |
| 7    | effectful-metrics: effect + prometheus + testing |
| 8    | effectful-cache + effectful-retry |
| 9    | effectful-testing meta-package |
| 10   | Documentation, examples, Hackage release |

## Design Decisions

### Why effectful and not polysemy/cleff?

- `effectful` has the best **performance** (close to plain IO)
- Maintained by active team
- Growing adoption in industry
- Compatible with existing ecosystem via `IOE`

### Why separate packages?

- Minimize dependency footprint
- Users only pay for what they use
- Backend-specific deps (postgresql-simple, hedis) are isolated

### Why dynamic dispatch?

- All effects use `'Dynamic` dispatch for maximum flexibility
- Allows runtime interpreter swapping (e.g., test vs production)
- Performance is still excellent with effectful's optimizations

### Error handling strategy

- Effects throw domain-specific exceptions
- Each effect module defines its own exception hierarchy
- `effectful-core`'s `Error` effect can be used alongside

```haskell
data DbError
  = DbConnectionError Text
  | DbQueryError Text SqlQuery
  | DbConstraintViolation Text
  deriving (Show)
instance Exception DbError
```
