# CreditWatch — Complete Build Specification

> **Status:** MVP architecture specification  
> **Primary language:** Kotlin  
> **Secondary language:** Java where JVM/platform interop is simpler  
> **Primary target:** Desktop (macOS, Windows, Linux)  
> **First cloud provider:** Vast.ai  
> **Core principle:** Small, local-first, read-only, understandable

---

## 1. Product Definition

CreditWatch is a small open-source desktop application that answers four questions:

1. **How much cloud credit do I have left?**
2. **How quickly am I spending it?**
3. **How long can I safely continue at the current rate?**
4. **Am I obviously paying for resources that appear underused?**

The application is deliberately **not** a full FinOps platform, infrastructure orchestrator, monitoring suite, or cloud management console.

The MVP should feel closer to a compact battery monitor for cloud credits than to Grafana.

The user installs CreditWatch on their own computer, connects a supported cloud provider using a read-only or minimum-permission API key, and immediately gets a small configurable dashboard plus native desktop warnings.

The initial provider is Vast.ai.

---

## 2. Product Philosophy

The project should remain governed by these constraints:

- local-first
- open source
- no mandatory account
- no central CreditWatch backend
- credentials remain on the user's machine
- no automatic cloud-resource modification in V1
- no mandatory telemetry agent
- small memory and CPU footprint
- maximum 12 dashboard tiles
- provider-neutral business logic
- explicit distinction between **measured**, **reported**, and **estimated** values
- recommendations must explain why they were produced
- the app must still be useful if the telemetry agent is never installed

A feature that does not materially help answer the four product questions should normally stay out of the MVP.

---

# 3. What We Are Building

## 3.1 Desktop application

The desktop application performs:

- provider authentication
- balance retrieval
- instance retrieval
- burn-rate calculation
- raw runway calculation
- safe-runway calculation
- historical sampling
- threshold evaluation
- burn anomaly detection
- local notifications
- optional telemetry ingestion
- lightweight efficiency analysis
- local persistence
- 3 × 4 configurable dashboard
- system tray/menu-bar operation

## 3.2 Optional instance telemetry agent

A very small agent may run inside a rented instance.

It provides information that the provider API may not expose with enough resolution:

- CPU utilization
- RAM utilization
- GPU utilization
- VRAM utilization
- disk utilization
- network throughput
- heartbeat / last-seen time

The telemetry agent does **not**:

- control the cloud instance
- expose a shell
- install drivers
- alter workloads
- automatically stop machines
- send data to CreditWatch servers

Its only purpose is to produce utilization measurements.

---

# 4. Non-Goals for V1

Do not build these into the initial release:

- AWS support
- Azure support
- GCP support
- RunPod support
- automatic instance shutdown
- automatic resizing
- automatic migration
- automatic load balancing
- Kubernetes support
- Prometheus support
- Grafana support
- team accounts
- enterprise RBAC
- browser dashboard
- iOS application
- AI recommendations
- ML-based forecasting
- invoice reconciliation
- tax accounting
- automatic payments
- credit purchasing
- cloud-resource creation
- cloud-resource deletion
- long-term observability storage
- central SaaS service

These can be future extensions, but they should not shape the MVP architecture more than necessary.

---

# 5. Core User Experience

The default dashboard is a 3-column × 4-row grid.

Maximum visible cards: **12**.

Example:

```text
┌──────────────┬──────────────┬──────────────┐
│ BALANCE      │ BURN RATE    │ SAFE RUNWAY  │
│              │              │              │
│ $24.18       │ $0.61/hr     │ 35h 42m      │
├──────────────┼──────────────┼──────────────┤
│ GPU LOAD     │ GPU MEMORY   │ CPU LOAD     │
│              │              │              │
│ 78%          │ 14/24 GB     │ 32%          │
├──────────────┼──────────────┼──────────────┤
│ RAM          │ NETWORK      │ STORAGE      │
│              │              │              │
│ 12/32 GB     │ 18 Mbps      │ 51%          │
├──────────────┼──────────────┼──────────────┤
│ INSTANCES    │ EFFICIENCY   │ ALERTS       │
│              │              │              │
│ 2 Running    │ 76%          │ 1 Warning    │
└──────────────┴──────────────┴──────────────┘
```

The user may:

- reorder tiles
- hide tiles
- replace one tile with another
- restore the default layout
- select the provider/account represented by a tile where relevant

For V1, do **not** support arbitrary tile dimensions.

All dashboard tiles should have the same size.

This significantly simplifies layout behavior, persistence, responsive sizing, drag-and-drop, and testing.

---

# 6. Recommended Technology Stack

## 6.1 Runtime

Use:

- **JDK 21 LTS**
- Kotlin/JVM
- Java where useful

JDK 21 gives a stable modern baseline while keeping deployment straightforward.

## 6.2 UI

Use:

- **Compose Multiplatform Desktop**

Reasons:

- Kotlin-native development experience
- one desktop UI codebase
- Windows/macOS/Linux support
- tray/menu-bar APIs
- native desktop distribution support
- no Electron/Node dependency

## 6.3 Networking

Use:

- **Ktor Client**

Recommended components:

- `ktor-client-core`
- `ktor-client-cio`
- `ktor-client-content-negotiation`
- `ktor-serialization-kotlinx-json`

Use a **single long-lived `HttpClient` instance** rather than constructing one per request.

Configure:

- request timeout
- connect timeout
- socket timeout
- response validation
- JSON serialization
- sanitized logging in development only
- retry policy only for safe idempotent operations

Never log:

- Authorization headers
- API keys
- secret values
- full error bodies if they could contain credentials

## 6.4 Serialization

Use:

- `kotlinx.serialization`

Provider JSON objects should deserialize into provider-specific DTOs and then be mapped into domain models.

Never let provider DTOs leak into the core business logic.

## 6.5 Persistence

For the MVP, use one of these:

### Preferred simple option

- SQLite through JDBC

Store:

- settings
- layout
- samples
- alerts
- provider account metadata
- telemetry history

### Alternative

If the team strongly prefers typed SQL:

- SQLDelight

Do not introduce an ORM unless it provides clear value.

## 6.6 Coroutines

Use:

- Kotlin Coroutines
- `StateFlow`
- `SharedFlow`

Use structured concurrency.

Avoid manually creating unmanaged threads.

---

# 7. Recommended Project Layout

Start with a multi-module Gradle project.

```text
creditwatch/
│
├── settings.gradle.kts
├── build.gradle.kts
├── gradle.properties
│
├── app-desktop/
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│
├── core-domain/
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│
├── core-engine/
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│
├── provider-api/
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│
├── provider-vast/
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│
├── persistence/
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│
├── notifications/
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│
├── telemetry-api/
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│
├── telemetry-agent/
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│
└── test-fixtures/
    ├── build.gradle.kts
    └── src/main/resources/
```

This is intentionally modular without becoming microservice-like.

---

# 8. Module Responsibilities

## `core-domain`

Contains pure models and value objects.

Examples:

- `Money`
- `MoneyRate`
- `ProviderId`
- `AccountId`
- `InstanceId`
- `CloudInstance`
- `BalanceSnapshot`
- `BurnSnapshot`
- `RunwaySnapshot`
- `TelemetrySnapshot`
- `EfficiencyAssessment`
- `Alert`
- `AlertRule`

It should not depend on:

- Compose
- Ktor
- SQLite
- Vast.ai
- platform notifications

This module should be highly testable.

---

## `provider-api`

Defines the provider contract.

Example:

```kotlin
interface CloudProvider {
    val providerId: ProviderId
    val capabilities: ProviderCapabilities

    suspend fun validateCredentials(): CredentialValidation
    suspend fun getAccountSnapshot(): ProviderAccountSnapshot
    suspend fun getInstances(): List<ProviderInstanceSnapshot>
}
```

The contract should expose what CreditWatch needs rather than mirroring provider REST APIs.

Example capabilities:

```kotlin
data class ProviderCapabilities(
    val providesBalance: Boolean,
    val providesInstancePricing: Boolean,
    val providesStoragePricing: Boolean,
    val providesBandwidthPricing: Boolean,
    val providesNativeTelemetry: Boolean,
    val balanceFreshness: DataFreshness,
    val billingFreshness: DataFreshness
)
```

---

## `provider-vast`

Contains:

- Vast HTTP client
- Vast authentication
- Vast DTOs
- Vast-to-domain mappers
- response validation
- Vast-specific billing interpretation

The rest of CreditWatch should not know how Vast formats its API.

---

## `core-engine`

Contains:

- burn calculator
- runway calculator
- safe-runway calculator
- sample smoothing
- anomaly detection
- efficiency rules
- alert evaluation

It consumes domain values.

It should contain no HTTP code and no UI code.

---

## `persistence`

Contains:

- SQLite connection
- migrations
- repositories
- retention cleanup
- settings storage

---

## `notifications`

Contains:

- notification service abstraction
- operating-system implementations
- notification deduplication
- cooldown logic

---

## `telemetry-api`

Contains the common telemetry schema and ingest interface.

---

## `telemetry-agent`

Contains the optional remote agent.

It should be independently buildable.

---

## `app-desktop`

Contains:

- Compose UI
- view models
- window state
- tray behavior
- dashboard customization
- dependency wiring
- lifecycle management

---

# 9. Domain Model

## 9.1 Money

Never use `Double` for account balances.

Use integer minor units or `BigDecimal`.

Example:

```kotlin
@JvmInline
value class CurrencyCode(val value: String)

data class Money(
    val amount: BigDecimal,
    val currency: CurrencyCode
)
```

For Vast.ai V1, currency will normally be USD.

All arithmetic should define an explicit scale and rounding strategy.

Recommended:

- internal precision: at least 6 decimal places
- display balance: 2 decimal places
- display hourly rate: 2–4 decimals depending on magnitude

---

## 9.2 Money rate

```kotlin
data class MoneyRate(
    val amountPerHour: BigDecimal,
    val currency: CurrencyCode
)
```

Normalize rates internally to **per hour**.

This keeps the engine consistent even if the provider bills per second.

Per-second billing can still be accurately represented:

```text
hourlyRate = perSecondRate × 3600
```

The provider's billing granularity should be retained as metadata.

---

## 9.3 Cloud instance

```kotlin
data class CloudInstance(
    val id: InstanceId,
    val providerId: ProviderId,
    val label: String?,
    val state: InstanceState,
    val gpuModel: String?,
    val gpuCount: Int?,
    val computeRate: MoneyRate?,
    val storageRate: MoneyRate?,
    val estimatedBandwidthRate: MoneyRate?,
    val createdAt: Instant?,
    val rawMetadata: Map<String, String> = emptyMap()
)
```

`rawMetadata` is useful for debugging/provider detail but must not become a substitute for proper typed fields.

---

# 10. Vast.ai Integration

Vast.ai is a strong first provider because its model naturally fits CreditWatch:

- prepaid credits
- per-second compute billing
- continuous storage billing
- bandwidth billing
- user API key
- programmatic instance listing
- visible hourly pricing fields

As of the current Vast documentation, compute is billed per second while the instance is billable, storage is charged continuously while an instance exists, and bandwidth is charged per transferred byte.

This means CreditWatch must **not** assume:

> stopped instance = zero burn

A stopped Vast instance can still incur storage cost.

That distinction is important.

---

# 11. Vast Authentication

Use a user-generated Vast API key.

Authentication should be encapsulated inside `provider-vast`.

The Vast client should send the appropriate bearer authorization header according to the current Vast API specification.

Credential handling rules:

- API key is requested once
- key is immediately persisted in secure local storage
- raw key is never placed in normal SQLite settings tables
- raw key is never included in logs
- UI should display only a masked representation
- removing an account must delete the locally stored credential

Example:

```text
Vast.ai
API key: vast_••••••••••••4d92
Status: Connected
```

---

# 12. Credential Storage

Do not store provider keys as plaintext in SQLite.

Prefer operating-system secure storage.

Recommended abstraction:

```kotlin
interface SecretStore {
    suspend fun put(id: SecretId, value: CharArray)
    suspend fun get(id: SecretId): CharArray?
    suspend fun delete(id: SecretId)
}
```

Platform implementations can use:

- macOS Keychain
- Windows Credential Manager
- Linux Secret Service / compatible keyring

If a portable fallback is ever provided, it must clearly state its security characteristics.

Do not silently downgrade to plaintext.

---

# 13. Provider Polling

Do not continuously hammer the provider API.

Recommended baseline:

```text
balance/account:   every 60 seconds
instances:         every 30–60 seconds
background idle:   every 2–5 minutes
manual refresh:    immediate
```

Polling should be configurable later, but defaults should be conservative.

Use exponential backoff after failures.

Example:

```text
failure 1 -> wait normal interval
failure 2 -> 2 min
failure 3 -> 4 min
failure 4 -> 8 min
cap       -> 15 min
```

Successful calls reset the backoff.

---

# 14. Polling State Machine

Each provider account can be modeled as:

```text
DISCONNECTED
    ↓
CONNECTING
    ↓
SYNCING
    ↓
HEALTHY
```

Errors may transition to:

```text
AUTH_ERROR
RATE_LIMITED
OFFLINE
DEGRADED
```

Do not discard the last valid data immediately.

Instead show:

```text
Balance     $18.42
Updated     7 min ago
Status      Provider unavailable
```

Stale data must be visibly marked.

---

# 15. Burn Rate

## 15.1 Basic burn rate

At its simplest:

```text
totalBurn =
    sum(instance.computeRate)
  + sum(instance.storageRate)
  + estimatedBandwidthRate
```

However, each term has different confidence.

For example:

```text
compute      provider reported
storage      provider reported/derived
bandwidth    estimated or unknown
```

Therefore store confidence/source metadata.

Example:

```kotlin
data class RateComponent(
    val type: CostType,
    val rate: MoneyRate,
    val confidence: Confidence,
    val source: DataSource
)
```

---

## 15.2 Do not hide incomplete burn data

If bandwidth cost cannot be reliably calculated:

Do not claim:

```text
Burn $0.6200/hr
```

as an exact all-in amount.

Prefer:

```text
Base burn $0.62/hr
+ bandwidth usage
```

or:

```text
Estimated burn $0.63/hr
```

with a detail view explaining the estimate.

---

# 16. Raw Runway

Raw runway answers:

> If the current burn remained unchanged, when would the available balance reach zero?

Formula:

```text
rawRunwayHours =
availableBalance / currentHourlyBurn
```

Example:

```text
balance = $30
burn    = $0.75/hr

runway = 40 hours
```

Edge cases:

### Burn = 0

Display:

```text
Runway: —
No active burn detected
```

Do not divide by zero.

### Balance <= 0

Display:

```text
Runway: 0m
Balance depleted
```

### Burn unknown

Display:

```text
Runway: unavailable
Waiting for billing data
```

---

# 17. Safe Runway

Raw runway can be overly optimistic.

Safe runway provides a conservative planning value.

For V1, avoid complicated forecasting.

Use a deterministic rule.

Recommended algorithm:

```text
referenceBurn =
max(
    currentBurn,
    movingAverage1h
)

safeBurn =
referenceBurn × safetyMultiplier

safeRunway =
availableBalance / safeBurn
```

Suggested default:

```text
safetyMultiplier = 1.10
```

Meaning a 10% safety buffer.

Example:

```text
balance          $30.00
current burn     $0.60/hr
1h average       $0.67/hr

reference burn   $0.67/hr
safe burn        $0.737/hr

safe runway      ~40h 42m
raw runway       50h
```

The safe-runway algorithm must be visible in settings/help.

Do not present it as a guarantee.

---

# 18. Optional Provider Freshness Margin

Some providers may have delayed cost data.

The provider contract should allow:

```kotlin
data class BillingFreshness(
    val expectedDelay: Duration?,
    val isRealtime: Boolean
)
```

If a provider reports delayed values, the safe-runway engine can optionally subtract a time margin.

Example:

```text
safeRunway =
calculatedSafeRunway - reportingDelayAllowance
```

For Vast.ai V1, use the freshest reliable provider fields available and clearly label any derived values.

---

# 19. Burn History

Persist periodic burn snapshots.

Example record:

```kotlin
data class BurnSnapshot(
    val accountId: AccountId,
    val timestamp: Instant,
    val balance: Money?,
    val computeRate: MoneyRate?,
    val storageRate: MoneyRate?,
    val bandwidthRate: MoneyRate?,
    val totalRate: MoneyRate?,
    val dataQuality: DataQuality
)
```

Recommended sample retention:

- high resolution: 24 hours
- medium resolution: 7 days
- optional aggregate: 30 days

For the MVP, 24–72 hours may be enough.

Keep storage bounded.

---

# 20. Sample Aggregation

Do not store millions of rows.

Simple strategy:

### Last 24 hours
One sample every 60 seconds.

Maximum:

```text
1,440 samples/account/day
```

### Older samples
Downsample to:

```text
5-minute or 15-minute averages
```

A background cleanup job can run once per startup and once per day.

---

# 21. Burn Spike Detection

The goal is not advanced anomaly detection.

A simple deterministic model is sufficient.

Example:

```text
baseline = average burn over previous 30 minutes
current  = latest burn

increasePercent =
(current - baseline) / baseline × 100
```

Trigger only if:

```text
baseline > minimumMeaningfulBurn
AND
increasePercent >= configuredThreshold
AND
condition persists for N samples
```

Suggested default:

```text
threshold = 50%
persistence = 2 samples
```

This avoids warning because of one noisy reading.

---

# 22. Alert System

Alerts should come from an explicit rule engine.

```kotlin
interface AlertRule {
    val id: AlertRuleId

    fun evaluate(context: MonitoringContext): AlertEvaluation
}
```

Rules should be pure wherever possible.

---

# 23. Initial Alert Types

## Balance alerts

Examples:

- below 25%
- below 10%
- below 5%

These require a reference balance.

For prepaid accounts where "original deposited amount" may not be reliable, percentage-based alerts should only be enabled when a meaningful reference amount exists.

Absolute thresholds may be more dependable:

- below $10
- below $5
- below $1

---

## Runway alerts

Recommended defaults:

- 24 hours
- 12 hours
- 6 hours
- 3 hours
- 1 hour
- 30 minutes

The user does not need all enabled.

Suggested first-run defaults:

- 12h
- 6h
- 1h

---

## Burn spike alerts

Example:

```text
Burn increased 82%
$0.41/hr → $0.75/hr
```

---

## Efficiency alerts

Examples:

- likely idle instance
- low GPU utilization
- possibly oversized GPU memory
- telemetry missing

---

# 24. Alert Deduplication

Do not notify the user every polling cycle.

Every alert needs lifecycle state:

```text
NEW
ACTIVE
ACKNOWLEDGED
RESOLVED
```

Example:

If safe runway remains under six hours:

```text
22:00 -> notify
22:01 -> no duplicate
22:02 -> no duplicate
```

Only notify again when:

- a lower threshold is crossed
- the alert resolved and later reoccurred
- the configured repeat cooldown expires

---

# 25. Hysteresis

Without hysteresis, values close to a threshold can cause repeated alert/resolution flipping.

Example threshold:

```text
safe runway < 6h
```

Resolve only when:

```text
safe runway > 6h 30m
```

This margin prevents notification chatter.

---

# 26. Notifications

Desktop notifications should work while the main window is hidden.

Compose Desktop can manage tray behavior, while JVM desktop APIs can provide fallback notification functionality where appropriate.

Design a platform-neutral abstraction:

```kotlin
interface NotificationService {
    suspend fun show(notification: CreditWatchNotification)
}
```

Notification payload:

```kotlin
data class CreditWatchNotification(
    val title: String,
    val body: String,
    val severity: Severity,
    val deduplicationKey: String,
    val action: NotificationAction?
)
```

---

# 27. System Tray / Menu Bar

The application should remain useful while hidden.

Tray/menu-bar behavior:

### Left click
Open compact dashboard.

### Menu
Provide:

```text
Open CreditWatch
Refresh now
Pause monitoring
Settings
Quit
```

Possible tray tooltip:

```text
Vast.ai — $18.42 — 23h safe
```

Do not put sensitive information in the tooltip if the user disables balance visibility.

---

# 28. Dashboard Architecture

Treat each tile as a view over observable state.

```kotlin
interface DashboardTile {
    val type: TileType
}
```

Tile configuration:

```kotlin
data class TilePlacement(
    val position: Int,
    val tileType: TileType,
    val configuration: Map<String, String>
)
```

Positions range:

```text
0..11
```

A simple ordered list is enough.

There is no need for a grid-layout database model.

---

# 29. Initial Tile Types

Recommended V1 tiles:

1. Balance
2. Burn Rate
3. Safe Runway
4. Raw Runway
5. Running Instances
6. GPU Utilization
7. VRAM Utilization
8. CPU Utilization
9. RAM Utilization
10. Network
11. Storage
12. Efficiency
13. Active Alerts
14. Last Sync

The user chooses up to 12.

---

# 30. Tile States

Every tile must support more than just a normal numeric state.

Required states:

```text
LOADING
AVAILABLE
STALE
UNAVAILABLE
ERROR
NOT_CONFIGURED
```

Example:

```text
GPU LOAD
—
Telemetry agent not connected
```

This is better than displaying 0%.

Zero is a valid measurement and must never mean "unknown."

---

# 31. Telemetry Architecture

Telemetry should be optional.

Preferred design:

```text
┌──────────────────────────┐
│ Desktop CreditWatch      │
│                          │
│ Billing Engine           │
│ Alert Engine             │
│ Efficiency Engine        │
└─────────────┬────────────┘
              │
        secure telemetry
              │
┌─────────────▼────────────┐
│ Instance Agent           │
│                          │
│ CPU / RAM / GPU / Disk   │
└──────────────────────────┘
```

---

# 32. Agent Data Contract

Example:

```kotlin
@Serializable
data class TelemetrySnapshot(
    val agentId: String,
    val timestampEpochMs: Long,
    val cpuPercent: Double?,
    val memoryUsedBytes: Long?,
    val memoryTotalBytes: Long?,
    val gpu: List<GpuTelemetry>,
    val diskUsedBytes: Long?,
    val diskTotalBytes: Long?,
    val networkRxBytesPerSecond: Long?,
    val networkTxBytesPerSecond: Long?
)
```

GPU:

```kotlin
@Serializable
data class GpuTelemetry(
    val index: Int,
    val name: String?,
    val utilizationPercent: Double?,
    val memoryUsedBytes: Long?,
    val memoryTotalBytes: Long?
)
```

---

# 33. Agent Sampling

Recommended sampling:

```text
local machine measurement: every 5–10 seconds
desktop transmission:      every 15–30 seconds
```

Do not send every raw OS reading.

The agent may average several readings before sending.

This reduces network use and noise.

---

# 34. Agent GPU Telemetry

For NVIDIA GPUs, the simplest V1 solution is to invoke `nvidia-smi`.

Example query concept:

```text
nvidia-smi \
  --query-gpu=index,name,utilization.gpu,memory.used,memory.total \
  --format=csv,noheader,nounits
```

The Kotlin process:

1. detects whether `nvidia-smi` exists
2. runs the command
3. applies a timeout
4. parses output
5. maps values into telemetry objects

Rules:

- never hang indefinitely waiting for `nvidia-smi`
- do not run it excessively
- tolerate multi-GPU output
- report unsupported GPU telemetry as unavailable
- do not fail the entire agent if GPU telemetry fails

---

# 35. CPU and Memory Telemetry

Prefer JVM/OS-accessible values where reliable.

Possible sources:

- `OperatingSystemMXBean`
- `/proc` on Linux where necessary
- OS-specific JVM calls

Keep platform-specific collectors behind:

```kotlin
interface SystemTelemetryCollector {
    suspend fun collect(): SystemTelemetry
}
```

The agent can initially target Linux because Vast instances are commonly Linux-based.

Desktop support for the monitoring agent is not an MVP requirement.

---

# 36. Telemetry Transport

Keep V1 simple.

Possible designs:

## Option A — desktop pulls over SSH

Pros:

- no listening network service on instance
- reuses existing SSH connectivity
- straightforward security model

Cons:

- repeated SSH overhead
- key configuration complexity

## Option B — tiny HTTPS/WebSocket agent

Pros:

- efficient live telemetry
- cleaner streaming behavior

Cons:

- port exposure
- TLS/authentication complexity
- more attack surface

## Recommended V1

Start with **SSH-based polling** or a user-explicit localhost/tunnel mechanism.

Do not expose an unauthenticated telemetry HTTP port publicly.

The architecture can later support a persistent agent transport.

---

# 37. Agent Pairing

The desktop application must associate telemetry with a provider instance.

Matching methods, in order of preference:

1. explicit instance ID supplied to agent
2. pairing token generated by desktop
3. SSH target selected from known provider instance

Do not attempt unreliable heuristics such as IP-only matching if avoidable.

---

# 38. Efficiency Engine

Efficiency is intentionally advisory.

It should answer:

> Is this resource obviously underused according to simple transparent rules?

It should **not** claim to know whether a workload is unnecessary.

---

# 39. Idle Instance Rule

Suggested first rule:

```text
CPU average < 10%
AND
GPU average < 10%
AND
network below threshold
FOR at least 30 minutes
```

Result:

```text
LIKELY IDLE
```

Not:

```text
WASTED INSTANCE
```

Suggested message:

```text
This instance has shown very low CPU, GPU and network activity
for 42 minutes.

Current cost: $0.62/hr
Potential saving if stopped: up to $14.88/day
```

Remember that on Vast, stopping can leave storage charges.

Therefore savings calculations must distinguish:

```text
compute savings
storage still charged
```

---

# 40. Low GPU Utilization Rule

Example:

```text
GPU average < 20%
for 30 minutes
AND instance is running
```

Suggestion:

```text
GPU utilization has averaged 12% over the last 30 minutes.
Check whether this workload still requires the current GPU.
```

Do not automatically recommend a specific cheaper GPU unless provider data supports a valid comparison.

---

# 41. VRAM Oversizing Rule

Example:

```text
peak VRAM use over 60 min < 35% of total VRAM
```

Suggestion:

```text
Peak GPU memory usage was 7.4 GB of 24 GB over the last hour.
A lower-memory GPU may be sufficient if your workload requirements permit it.
```

This is a hint, not a proof.

Memory requirements can spike later.

---

# 42. Efficiency Score

If an efficiency score is included, it must remain simple and explainable.

For a GPU-focused workload:

```text
GPU utilization      60%
VRAM utilization     20%
CPU utilization      10%
network activity     10%
```

Potential formula:

```text
score =
gpuNormalized  * 0.60 +
vramNormalized * 0.20 +
cpuNormalized  * 0.10 +
networkNormalized * 0.10
```

However, do not overemphasize this score.

The recommendations are more useful than an arbitrary percentage.

A better tile could show:

```text
EFFICIENCY
Good

1 possible optimization
```

rather than pretending `73%` is scientific.

---

# 43. Cost Savings Estimation

For an instance judged likely idle:

```text
potentialComputeSavingPerDay =
computeHourlyRate × 24
```

If stopped instances retain storage billing:

```text
potentialNetSavingPerDay =
computeHourlyRate × 24
```

Do not subtract storage unless destruction is being considered.

If CreditWatch does not support destructive actions, label it:

```text
Potential compute saving
```

not:

```text
Guaranteed saving
```

---

# 44. Data Quality

Each important value should carry quality metadata.

Example:

```kotlin
enum class DataQuality {
    LIVE,
    RECENT,
    STALE,
    ESTIMATED,
    PARTIAL,
    UNKNOWN
}
```

A dashboard tile can then communicate:

```text
$0.61/hr
Estimated
```

or:

```text
$24.18
Updated 35s ago
```

This prevents false precision.

---

# 45. Application State

Central app state can be exposed through flows.

Example:

```kotlin
data class AccountMonitoringState(
    val account: ConnectedAccount,
    val balance: BalanceSnapshot?,
    val instances: List<CloudInstance>,
    val burn: BurnSnapshot?,
    val runway: RunwaySnapshot?,
    val alerts: List<Alert>,
    val syncState: SyncState
)
```

A repository/service owns this state.

Compose observes `StateFlow`.

The UI should not directly call provider APIs.

---

# 46. Service Layer

Recommended services:

```text
ProviderSyncService
BurnEngine
RunwayEngine
AlertEngine
TelemetryService
EfficiencyEngine
NotificationCoordinator
HistoryService
```

Data flow:

```text
Provider API
    ↓
ProviderSyncService
    ↓
Domain snapshots
    ↓
BurnEngine
    ↓
RunwayEngine
    ↓
AlertEngine
    ↓
UI / notifications
```

Telemetry joins separately:

```text
Telemetry Collector
    ↓
TelemetryService
    ↓
EfficiencyEngine
    ↓
UI / efficiency alerts
```

---

# 47. Dependency Direction

Maintain this direction:

```text
app-desktop
    ↓
core-engine
    ↓
core-domain

provider-vast
    ↓
provider-api
    ↓
core-domain

persistence
    ↓
core-domain
```

The core must never depend on Vast.

This is what makes later providers manageable.

---

# 48. Vast Provider Mapping

The Vast adapter should extract only fields CreditWatch needs.

Typical instance information includes:

- instance ID
- actual status
- GPU name
- GPU count
- total hourly price
- storage-related pricing fields where available
- instance label
- connection metadata if telemetry setup needs SSH

Vast's own tooling exposes instance listing and account information through its API/CLI.

Use the current Vast OpenAPI specification as the source of truth when implementing actual field names.

Do not rely on copied examples indefinitely.

---

# 49. API Evolution Strategy

Provider APIs change.

Therefore:

- keep raw DTOs isolated
- write fixture-based parser tests
- ignore unknown JSON fields
- fail gracefully when optional fields disappear
- log schema mismatch without logging secrets
- include provider adapter version in diagnostics

Fixture example:

```text
test-fixtures/
└── vast/
    ├── user-current.json
    ├── instances-running.json
    ├── instances-stopped.json
    ├── unauthorized.json
    └── rate-limited.json
```

---

# 50. Error Handling

The user should receive useful states, not stack traces.

Map errors into categories:

```kotlin
sealed interface ProviderError {
    data object Unauthorized : ProviderError
    data object RateLimited : ProviderError
    data object NetworkUnavailable : ProviderError
    data object Timeout : ProviderError
    data class InvalidResponse(val reason: String) : ProviderError
    data class ServerError(val statusCode: Int) : ProviderError
}
```

UI examples:

```text
Vast.ai authentication failed.
Check your API key.
```

```text
Vast.ai is temporarily unavailable.
Showing data from 4 minutes ago.
```

---

# 51. Rate Limiting

If HTTP 429 occurs:

- do not retry immediately in a loop
- honor `Retry-After` if supplied
- otherwise apply backoff
- show a degraded sync indicator
- continue displaying last known values

---

# 52. Offline Behavior

When internet connectivity disappears:

CreditWatch should:

- retain last state
- mark values stale
- stop firing new balance/runway alerts based purely on aging values
- optionally show one connectivity alert
- resume automatically when connectivity returns

Do not calculate a new countdown every second from stale burn/balance indefinitely as if it were current provider truth.

---

# 53. Time Handling

Use:

- `java.time.Instant` internally
- UTC for storage
- system local timezone for display

Never store local formatted timestamps as source data.

---

# 54. Local Database Schema

A simple initial schema:

## `connected_accounts`

```text
id
provider_id
display_name
secret_reference
created_at
last_successful_sync_at
```

## `burn_samples`

```text
id
account_id
timestamp
balance_amount
currency
compute_per_hour
storage_per_hour
bandwidth_per_hour
total_per_hour
data_quality
```

## `telemetry_samples`

```text
id
instance_id
timestamp
cpu_percent
ram_percent
gpu_percent
vram_percent
disk_percent
network_rx_bps
network_tx_bps
```

## `alerts`

```text
id
account_id
instance_id nullable
rule_id
severity
state
first_triggered_at
last_triggered_at
resolved_at nullable
payload_json
```

## `settings`

```text
key
value
```

## `dashboard_layout`

```text
position
tile_type
configuration_json
```

---

# 55. Database Migrations

Every schema change must be migration-driven.

Example:

```text
001_initial.sql
002_add_data_quality.sql
003_add_alert_state.sql
```

Never silently recreate the database on upgrade.

---

# 56. Settings

Initial settings:

### General

- launch at login
- minimize to tray
- refresh interval
- show balance in tray
- start minimized

### Alerts

- safe-runway thresholds
- balance thresholds
- burn spike threshold
- idle-instance detection
- notification cooldown

### Safe runway

- safety multiplier
- moving-average window

### Telemetry

- enable/disable
- polling interval
- idle-duration threshold

---

# 57. Sensible Defaults

Recommended defaults:

```text
Provider refresh        60 sec
Instance refresh        60 sec
Telemetry refresh       30 sec
Safe multiplier         1.10
Burn baseline           30 min
Burn spike warning      +50%
Idle threshold          30 min
Low GPU threshold       <10–15%
```

Defaults should prioritize not annoying the user.

---

# 58. UI Screens

Keep screen count small.

## Screen 1 — Dashboard

Primary 3 × 4 grid.

## Screen 2 — Accounts / Providers

Add/remove Vast account.

## Screen 3 — Alerts

Recent and active warnings.

## Screen 4 — Settings

Monitoring and layout settings.

## Optional detail sheet

Clicking a tile opens a small detail view rather than navigating to a huge analytics screen.

---

# 59. First-Run Flow

```text
Welcome
   ↓
Add provider
   ↓
Select Vast.ai
   ↓
Paste API key
   ↓
Validate
   ↓
Initial sync
   ↓
Dashboard
```

After validation:

```text
Connected

Balance: $30.00
Instances: 2
Estimated burn: $0.62/hr
Safe runway: 43h
```

Do not force the telemetry agent during onboarding.

Later:

```text
Want instance utilization?
Set up optional telemetry
```

---

# 60. Compact Window Behavior

The application should default to a small utility-window size.

Example starting dimensions:

```text
~720 × 600 px
```

Exact size should be determined after prototyping.

Important behavior:

- no excessive navigation chrome
- no sidebar unless it genuinely improves usability
- cards remain readable
- window may be resizable within reasonable minimum bounds
- compact tray popup may show only 3–6 key values

---

# 61. Notification Examples

## Safe runway

```text
CreditWatch — Vast.ai

Safe runway is below 6 hours.

Balance: $4.31
Burn: $0.76/hr
Safe runway: 5h 09m
```

## Burn spike

```text
CreditWatch — Burn increased

Vast.ai burn increased 84%.

Before: $0.41/hr
Now: $0.75/hr
```

## Likely idle instance

```text
CreditWatch — Possible idle instance

RTX 4090 instance #18421 has averaged:
GPU 6%
CPU 8%

for 41 minutes.

Compute cost: $0.62/hr
```

---

# 62. Notification Severity

Suggested levels:

```text
INFO
WARNING
CRITICAL
```

Examples:

### INFO
Possible efficiency improvement.

### WARNING
Safe runway under 6 hours.

### CRITICAL
Safe runway under 1 hour.

Do not overuse CRITICAL.

---

# 63. Logging

Use structured logs.

Recommended levels:

- ERROR
- WARN
- INFO
- DEBUG

Production logs should include:

- sync start/end
- provider error category
- alert state transitions
- database migration events
- agent connectivity state

Production logs must not include:

- API key
- bearer token
- SSH private key
- full secrets
- unredacted auth headers

---

# 64. Diagnostics Export

A useful future-safe feature is:

```text
Export diagnostics
```

It may include:

- app version
- OS
- Java version
- provider adapter version
- recent redacted logs
- sync error categories
- database schema version

It must exclude secrets.

This will be extremely useful for open-source bug reports.

---

# 65. Security Principles

## Minimum privilege

CreditWatch should only request permissions it needs.

For MVP:

- read account/balance information
- read instances
- read pricing/status

Avoid write operations.

If Vast offers scoped API keys, document the minimum required scope.

---

## No cloud-control API in V1

The provider interface should not initially expose:

```text
stopInstance()
destroyInstance()
createInstance()
changeBid()
```

This prevents accidental product creep and reduces the blast radius of a compromised local process.

---

# 66. Threat Model

Primary risks:

1. stolen provider API key
2. plaintext secret storage
3. secret leakage through logs
4. malicious telemetry connection
5. supply-chain dependency compromise
6. untrusted provider JSON
7. exposed telemetry agent
8. malicious local process reading app memory

Mitigations:

- OS secret store
- sanitized logs
- minimal API permissions
- TLS/SSH for telemetry
- strict JSON size limits
- dependency scanning
- signed releases
- no remote command execution through agent
- no central server holding keys

---

# 67. HTTP Safety

Provider client should:

- require HTTPS
- limit response body size
- define connection/read timeouts
- validate expected status codes
- avoid arbitrary redirects where credentials could leak
- never attach Authorization headers to a different host
- sanitize provider error text before logging

---

# 68. Telemetry Security

If using an agent network service:

Require:

- authentication token
- encrypted transport
- replay-resistant request design if practical
- narrow API surface
- no command execution endpoint
- no arbitrary file access

Preferred endpoints would be something like:

```text
GET /health
GET /telemetry
```

Nothing more for V1.

---

# 69. Testing Strategy

Testing should focus heavily on calculations and provider parsing.

---

# 70. Unit Tests

Required unit-test areas:

## Money

- addition
- rounding
- currency mismatch protection

## Burn calculator

- running instance
- stopped instance with storage
- missing rate
- multiple instances

## Runway

- normal case
- zero burn
- negative balance
- unknown balance
- very small burn

## Safe runway

- current burn > average
- average > current burn
- multiplier
- missing history

## Alert rules

- threshold crossing
- hysteresis
- duplicate suppression
- resolution

## Efficiency

- idle detection
- one high-utilization signal prevents idle
- insufficient telemetry history
- missing GPU

---

# 71. Provider Contract Tests

Using saved Vast JSON fixtures:

- valid account
- valid running instances
- stopped instance
- missing optional fields
- HTTP 401
- HTTP 429
- malformed response
- extra unknown fields

The provider mapper must be deterministic.

---

# 72. Integration Tests

Use a fake HTTP server.

Do not make normal automated tests depend on real Vast accounts.

Tests should verify:

```text
fake provider response
    ↓
provider adapter
    ↓
domain model
    ↓
burn engine
    ↓
runway engine
    ↓
alert
```

---

# 73. UI Tests

Test:

- all 12 tiles render
- unavailable state
- stale state
- long GPU names
- tiny balance
- huge balance
- 0 instances
- drag/reorder layout
- saved layout restoration

---

# 74. Manual Acceptance Tests

Before release:

### Authentication

- invalid API key
- revoked API key
- valid API key

### Networking

- no internet
- provider timeout
- provider rate limit
- internet restored

### Billing

- one running instance
- multiple running instances
- stopped instance
- zero balance
- zero active compute burn

### Notifications

- safe runway threshold
- duplicate suppression
- alert resolution

### Telemetry

- no `nvidia-smi`
- one GPU
- multiple GPUs
- agent disconnected

---

# 75. Performance Budget

CreditWatch should remain lightweight.

Suggested targets, not hard guarantees:

```text
idle CPU           near 0%
active polling     brief spikes only
desktop memory     preferably <150–200 MB
database           tens of MB, not GB
network            tiny periodic API requests
```

Compose Desktop includes a JVM/runtime footprint, so do not optimize obsessively before measuring.

The important part is avoiding continuous busy loops and large telemetry retention.

---

# 76. Scheduling

Use coroutine-based scheduling.

Example concept:

```kotlin
while (currentCoroutineContext().isActive) {
    sync()
    delay(refreshInterval)
}
```

However:

- lifecycle must own the coroutine scope
- exceptions must not silently terminate monitoring
- retry/backoff state should be explicit
- app shutdown must cancel jobs cleanly

---

# 77. Concurrency Rules

Avoid overlapping provider syncs.

If a 60-second sync takes 75 seconds:

Do not start another sync at second 60.

Use:

```text
previous completes
    ↓
wait interval
    ↓
next begins
```

or a mutex.

Manual refresh may request an immediate refresh but should coalesce with one already running.

---

# 78. Snapshot Consistency

Balance and instance data may come from separate API requests.

Therefore a burn/runway calculation should know the timestamps of its inputs.

Example:

```text
balance timestamp   22:01:05
instances timestamp 22:01:09
calculated          22:01:09
```

If input age difference exceeds a configured tolerance, mark calculation quality as partial or estimated.

---

# 79. Safe Calculation Pipeline

Recommended flow:

```text
1. Fetch account
2. Fetch instances
3. Normalize provider data
4. Validate timestamps
5. Calculate burn
6. Store burn sample
7. Calculate moving averages
8. Calculate raw runway
9. Calculate safe runway
10. Evaluate alert rules
11. Evaluate efficiency rules
12. Update UI state atomically
13. Send newly triggered notifications
```

This ordering keeps UI values internally consistent.

---

# 80. Atomic UI Updates

Avoid a dashboard momentarily showing:

```text
new balance
old burn
old runway
```

Create a combined state object and publish it after a monitoring cycle completes.

---

# 81. Startup Behavior

On startup:

```text
1. initialize logging
2. open database
3. run migrations
4. load settings
5. load last persisted snapshot
6. render UI immediately
7. load credentials
8. begin provider sync
9. begin telemetry polling if configured
10. evaluate state after fresh data arrives
```

Showing the last snapshot makes startup feel instant.

Mark it:

```text
Last updated 2h ago
Refreshing…
```

---

# 82. Shutdown Behavior

On quit:

- cancel monitoring jobs
- stop telemetry polling
- flush pending database writes
- close Ktor client
- close SQLite connection
- remove tray resources
- erase transient secret buffers where practical

---

# 83. Packaging

Use Compose Desktop native distribution tooling.

Produce platform-native artifacts:

### macOS

- `.dmg` and/or `.pkg`
- sign
- notarize for normal distribution

### Windows

- `.msi` or `.exe`
- code signing recommended

### Linux

- `.deb`
- optionally RPM later

Each OS artifact generally needs to be built on the target OS because native packaging includes platform-specific components.

Use CI matrix builds later.

---

# 84. Release Channels

Start simple:

```text
0.1.x  MVP
0.2.x  hardening
0.3.x  telemetry improvements
1.0    stable Vast-first release
```

Do not call the app 1.0 until:

- calculations are stable
- credentials are secured
- migrations work
- notifications are reliable
- update/install process is understood

---

# 85. Version Display

Expose:

```text
CreditWatch 0.1.0
Provider Vast adapter 0.1.0
Database schema 3
```

Useful for support.

---

# 86. Open-Source Repository Structure

At repository root:

```text
README.md
ARCHITECTURE.md
SECURITY.md
CONTRIBUTING.md
LICENSE
CODE_OF_CONDUCT.md
CHANGELOG.md
```

Recommended license depends on your project goals.

Common choices:

- Apache-2.0
- MIT

Apache-2.0 is attractive if explicit patent terms matter.

---

# 87. `SECURITY.md`

State clearly:

- how to report vulnerabilities
- what data CreditWatch stores
- where credentials live
- that provider API keys should be minimum privilege
- that secrets must never be included in GitHub issues
- supported versions

---

# 88. Provider Plugin Strategy

Do not build a dynamic plugin loader in V1.

Use compile-time modules.

Example:

```text
provider-api
provider-vast
```

Later:

```text
provider-runpod
provider-aws
provider-gcp
```

A dynamic third-party plugin system creates classloading, signing, security, compatibility, and distribution complexity that the MVP does not need.

---

# 89. Provider Normalization

Each adapter should map provider-specific semantics into these concepts:

```text
account balance
instance state
compute rate
storage rate
bandwidth rate/usage
billing freshness
provider capabilities
```

If a provider cannot supply one concept, return it as unavailable.

Never fake support.

---

# 90. Multiple Providers Later

The core should naturally allow:

```text
Vast.ai
$18.42
$0.72/hr
23h safe

RunPod
$8.21
$0.31/hr
24h safe
```

But do not implement this UI until Vast is solid.

---

# 91. Multiple Accounts Later

Design account IDs now, even if the first release supports one Vast account.

This prevents painful schema changes.

```kotlin
@JvmInline
value class AccountId(val value: UUID)
```

---

# 92. Accuracy Rules

CreditWatch must never claim more accuracy than it has.

Display conventions:

### Exact-ish provider-reported balance

```text
$18.42
```

### Derived rate

```text
~$0.63/hr
```

### Estimated runway

```text
~27h 14m
```

### Stale

```text
$18.42
Updated 9m ago
```

---

# 93. Do Not Confuse Price With Actual Burn

An instance's advertised/rental hourly price is not necessarily the complete account burn.

Account burn may include:

- compute
- storage
- bandwidth
- reserved/prepaid behavior
- provider-specific adjustments

The UI should distinguish:

```text
Instance compute rate
```

from:

```text
Estimated account burn
```

---

# 94. Vast-Specific Storage Warning

Because Vast documentation states that storage continues billing while an instance exists even when stopped, any efficiency recommendation must account for this.

Incorrect:

```text
Stop this instance and save $0.72/hr total.
```

Better:

```text
Stopping may remove the compute portion of this burn.
Storage charges can continue while the instance exists.
```

CreditWatch should display storage separately whenever the API provides sufficient data.

---

# 95. Recommended MVP Burn Model for Vast

Start with:

```text
known instance hourly cost components
+
known storage cost components
+
bandwidth only when reliably measurable
```

Store the calculation breakdown.

Example detail:

```text
Estimated burn

Compute        $0.540/hr
Storage        $0.031/hr
Bandwidth      not projected
─────────────────────────
Known burn     $0.571/hr
```

This is more honest than silently inventing bandwidth.

---

# 96. Alert Context

Every warning should capture the data that caused it.

Example:

```kotlin
data class AlertEvidence(
    val values: Map<String, String>,
    val observedAt: Instant
)
```

This allows the alert screen to explain:

```text
Triggered because:
Safe runway reached 5h 58m at 22:14.
```

---

# 97. Efficiency Evidence

Same concept:

```text
GPU average       6.2%
CPU average       8.4%
network average   0.8 Mbps
window            41 min
```

Users should be able to understand the recommendation without trusting a black box.

---

# 98. First Implementation Milestones

## Milestone 0 — Skeleton

Goal:

Application launches.

Build:

- Gradle modules
- JDK 21 toolchain
- Compose desktop window
- basic navigation
- test setup

No API calls yet.

---

## Milestone 1 — Vast Connection

Goal:

Connect a Vast account.

Build:

- Ktor client
- secret-store abstraction
- API-key form
- validation request
- provider DTOs
- provider error mapping

Done when:

```text
User pastes key
→ CreditWatch says Connected
```

---

## Milestone 2 — First Three Cards

Goal:

Produce the first useful application.

Cards:

```text
BALANCE
BURN RATE
SAFE RUNWAY
```

Build:

- account snapshot
- instance listing
- normalized rates
- burn engine
- runway engine
- safe-runway engine

This is the real MVP core.

Do not add telemetry before these three numbers are trustworthy.

---

## Milestone 3 — Background Monitoring

Build:

- polling scheduler
- local history
- stale-data handling
- backoff
- offline behavior

---

## Milestone 4 — Alerts

Build:

- alert rule interface
- runway rules
- burn spike
- deduplication
- hysteresis
- native notifications

---

## Milestone 5 — Full 3 × 4 Dashboard

Build:

- tile model
- tile registry
- drag/reorder
- persistence
- reset layout

---

## Milestone 6 — Tray Mode

Build:

- tray/menu-bar icon
- hide/show
- quick values
- start minimized
- quit behavior

---

## Milestone 7 — Telemetry Prototype

Build:

- Linux telemetry collector
- `nvidia-smi` parser
- CPU/RAM
- local snapshot output
- pairing concept

Do not expose public network service yet.

---

## Milestone 8 — Efficiency Rules

Build:

- idle rule
- low-GPU rule
- VRAM oversizing hint
- potential compute saving

---

## Milestone 9 — Packaging

Build:

- macOS artifact
- Windows artifact
- Linux artifact
- CI
- signed/notarized releases where practical

---

# 99. Suggested Development Order Inside JetBrains

Create the project with:

```text
Name:       creditwatch
Language:   Kotlin
Build:      Gradle
DSL:        Kotlin
JDK:        21
```

Then work in this order:

```text
1. core-domain
2. provider-api
3. provider-vast
4. core-engine
5. app-desktop
6. persistence
7. notifications
8. telemetry
```

Do not start by polishing the dashboard.

Prove data correctness first.

---

# 100. The First Vertical Slice

The first end-to-end slice should be only:

```text
Vast API key
    ↓
validate account
    ↓
retrieve balance
    ↓
retrieve instances
    ↓
derive current known burn
    ↓
calculate safe runway
    ↓
render 3 cards
```

If this works reliably, the project has a foundation.

If this does not work reliably, telemetry and customization are distractions.

---

# 101. Suggested Core Interfaces

## Provider

```kotlin
interface CloudProvider {
    val id: ProviderId
    val capabilities: ProviderCapabilities

    suspend fun validateCredentials(): CredentialValidation
    suspend fun getAccountSnapshot(): ProviderAccountSnapshot
    suspend fun getInstances(): List<ProviderInstanceSnapshot>
}
```

## Burn

```kotlin
interface BurnCalculator {
    fun calculate(
        account: ProviderAccountSnapshot,
        instances: List<ProviderInstanceSnapshot>
    ): BurnSnapshot
}
```

## Runway

```kotlin
interface RunwayCalculator {
    fun raw(
        balance: Money,
        burn: MoneyRate
    ): Duration?

    fun safe(
        balance: Money,
        currentBurn: MoneyRate,
        movingAverage: MoneyRate?,
        safetyMultiplier: BigDecimal
    ): Duration?
}
```

## Alerts

```kotlin
interface AlertRule {
    fun evaluate(context: MonitoringContext): AlertEvaluation
}
```

## Telemetry

```kotlin
interface TelemetrySource {
    suspend fun snapshot(): TelemetrySnapshot
}
```

---

# 102. Suggested ViewModel Pattern

Use unidirectional state.

```text
Repository/services
      ↓
ViewModel StateFlow
      ↓
Compose
      ↓
User intent
      ↓
ViewModel
```

UI should emit intents such as:

```text
Refresh
OpenSettings
MoveTile
RemoveTile
AddTile
AcknowledgeAlert
```

Business calculations stay outside composables.

---

# 103. Do Not Put Logic in Tiles

Bad:

```kotlin
@Composable
fun RunwayTile() {
    val runway = balance / burn
}
```

Correct:

```kotlin
@Composable
fun RunwayTile(state: RunwayTileState)
```

Calculation belongs in `core-engine`.

---

# 104. Data Refresh Indicator

Every dashboard should expose sync freshness unobtrusively.

Example footer:

```text
Vast.ai • Updated 28s ago
```

If stale:

```text
Vast.ai • Last update 8m ago • Offline
```

---

# 105. Privacy

CreditWatch should not send analytics by default.

Recommended V1:

```text
No telemetry about the user.
No crash uploads without explicit opt-in.
No provider credentials leave the machine.
```

If crash reporting is introduced later, it should be opt-in or clearly disclosed and rigorously scrub secrets.

---

# 106. Configuration Export

A future-safe feature:

```text
Export settings
```

Export:

- dashboard layout
- alert thresholds
- application preferences

Do not export secrets.

---

# 107. Update Mechanism

For the earliest releases, manual updates are acceptable.

Do not block the MVP on an automatic updater.

Later:

- signed update metadata
- GitHub Releases
- update notification
- optional auto-update

Security-sensitive updates should eventually be easy to install.

---

# 108. GitHub CI

Suggested jobs:

```text
build
unit-test
integration-test
lint
dependency-scan
package-macos
package-windows
package-linux
```

Package jobs can initially run only on tagged releases.

---

# 109. Coding Standards

Use:

- explicit immutable data models
- small functions
- sealed types for state/error categories
- constructor injection
- no global mutable singletons
- coroutines rather than raw threads
- typed IDs where reasonable

Avoid:

- giant `AppManager`
- huge God ViewModel
- provider logic in UI
- database code in provider adapter
- JSON maps throughout core logic

---

# 110. Dependency Injection

Do not add a large DI framework unless needed.

For this size project, manual constructor injection is fine.

Example:

```kotlin
val provider = VastProvider(httpClient, secretStore)
val repository = MonitoringRepository(provider, database)
val engine = MonitoringEngine(...)
val viewModel = DashboardViewModel(repository, engine)
```

This is explicit and easy to debug.

---

# 111. Naming

Keep names literal.

Good:

```text
BurnCalculator
RunwayCalculator
SafeRunwayPolicy
ProviderSyncService
VastProvider
TelemetrySnapshot
IdleInstanceRule
```

Avoid abstract enterprise naming such as:

```text
ResourceOptimizationIntelligenceOrchestrator
```

The project is meant to remain small.

---

# 112. UI Design Principles

- numbers first
- minimal decoration
- readable at a glance
- one primary value per card
- small secondary explanation
- color must not be the only status signal
- dark and light mode eventually
- keyboard accessible
- avoid animations that distract from monitoring

---

# 113. Card Content Pattern

Each card:

```text
TITLE

PRIMARY VALUE

secondary detail
status/freshness
```

Example:

```text
SAFE RUNWAY

5h 42m

Raw: 7h 09m
Updated 22s ago
```

---

# 114. Alert Detail Pattern

Each alert should show:

```text
What happened?
Why?
When?
What does it cost?
What can the user consider doing?
```

Example:

```text
Possible idle instance

RTX 4090 #18421 has averaged:
GPU 6%
CPU 8%
Network 0.4 Mbps

for 41 minutes.

Compute price: $0.62/hr
Potential compute saving: $14.88/day

Consider stopping the instance if this activity is not expected.
Storage costs may continue while it exists.
```

---

# 115. Accessibility

At minimum:

- text labels for icons
- keyboard navigation
- adequate contrast
- do not rely solely on red/green
- scalable fonts
- meaningful accessibility descriptions

---

# 116. Internationalization

Do not build full localization in V1, but avoid making it impossible.

Keep UI strings centralized.

Use locale-aware formatting for:

- currency
- numbers
- date/time

Internally keep USD/provider currency unchanged.

---

# 117. Metrics the Project Itself Should Care About

Do not add remote analytics.

During development/manual testing, evaluate:

- API requests per hour
- average sync duration
- memory use
- startup time
- alert false positives
- safe-runway calculation correctness
- percentage of sessions requiring telemetry agent
- provider error rate

These can be tested locally.

---

# 118. Definition of Done for V0.1

V0.1 is complete when a user can:

1. install CreditWatch
2. connect Vast.ai
3. see current balance
4. see known current burn
5. see raw runway
6. see safe runway
7. receive a low-runway notification
8. minimize the app to tray
9. restart the app and retain settings
10. remove their Vast credentials cleanly

Telemetry does **not** need to be in V0.1.

---

# 119. Definition of Done for V0.2

Add:

- full 3 × 4 customization
- burn spike warning
- telemetry prototype
- GPU/CPU/RAM cards
- likely-idle warning
- basic efficiency explanation

---

# 120. Future Expansion Rules

A new cloud provider should only need to implement:

```text
authentication
account snapshot
instance snapshot
pricing normalization
capability declaration
```

The following must remain reusable:

```text
burn engine
runway engine
safe runway
alert engine
dashboard
notification system
history
efficiency engine
```

If adding a provider requires rewriting those systems, the provider abstraction is wrong.

---

# 121. Important Vast.ai Implementation Notes

Current Vast documentation and tooling indicate:

- Vast is a marketplace with variable prices.
- GPU compute is billed per second while billable.
- storage is billed while the instance exists, including when stopped.
- bandwidth is charged by data transferred.
- users operate with prepaid credits/account balance.
- account information and instance data are programmatically accessible.
- current Vast tooling exposes an instances API and user/account API.
- the current Vast OpenAPI repository should be treated as authoritative when implementing exact endpoint/field bindings.

Do not freeze undocumented assumptions into the domain layer.

---

# 122. Current Technical References

These references are useful when implementing the project.

## Vast.ai documentation

Official documentation repository:

https://github.com/vast-ai/docs

Pricing overview:

https://github.com/vast-ai/docs/blob/main/guides/pricing.mdx

Instance pricing:

https://github.com/vast-ai/docs/blob/main/guides/instances/pricing.mdx

Official Vast CLI repository/docs:

https://github.com/vast-ai/vast-cli

Vast console:

https://console.vast.ai/

## Ktor

Client configuration:

https://ktor.io/docs/client-create-and-configure.html

Client engines:

https://ktor.io/docs/client-engines.html

## Java desktop tray

SystemTray API:

https://docs.oracle.com/en/java/javase/21/docs/api/java.desktop/java/awt/SystemTray.html

## Compose Multiplatform

JetBrains Compose Multiplatform documentation:

https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-multiplatform.html

---

# 123. Final Architecture Summary

The entire product should reduce to this:

```text
                         ┌─────────────────────┐
                         │  CREDITWATCH UI     │
                         │  3 × 4 dashboard    │
                         └─────────┬───────────┘
                                   │
                         ┌─────────▼───────────┐
                         │ Monitoring State    │
                         └─────────┬───────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
      ┌───────▼────────┐  ┌────────▼────────┐  ┌───────▼────────┐
      │ Billing Engine │  │ Alert Engine    │  │ Efficiency     │
      │                │  │                 │  │ Engine         │
      │ Burn           │  │ Runway          │  │                │
      │ Raw runway     │  │ Burn spikes     │  │ Idle hints     │
      │ Safe runway    │  │ Notifications   │  │ Utilization    │
      └───────▲────────┘  └─────────────────┘  └───────▲────────┘
              │                                         │
      ┌───────┴─────────┐                       ┌───────┴────────┐
      │ Provider API    │                       │ Telemetry      │
      │                 │                       │ Optional       │
      │ Vast.ai V1      │                       │ Instance agent │
      └─────────────────┘                       └────────────────┘
```

The provider side answers:

```text
What am I paying?
How much is left?
```

The telemetry side answers:

```text
What am I actually using?
```

The engine combines those answers into:

```text
How fast am I burning money?
How long is left?
Is my current usage obviously inefficient?
```

That is CreditWatch.

---

# 124. The Rule That Should Keep the Project Healthy

Before adding any feature, ask:

> Does this help the user understand balance, burn, runway, or obvious resource efficiency?

If the answer is no, it probably does not belong in the core application.

The strength of CreditWatch should be that a user can install it, connect a provider, glance at twelve small tiles, and immediately understand whether their cloud credits are safe.

