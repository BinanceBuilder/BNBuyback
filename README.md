# BNBuyback

![BNBuyback](bnbuyback.png)

**Automated On-Chain Buyback Execution Infrastructure for BNB Chain**

---

## Project Overview

BNBuyback is a deterministic buyback execution system designed for protocols operating on BNB Chain. It converts protocol-generated revenue or accumulated fees into programmatic buy pressure through scheduled or condition-based market purchases.

This is infrastructure, not a financial product. BNBuyback provides a transparent, verifiable, and non-custodial mechanism for executing buybacks according to predefined rules. It does not involve discretionary execution, admin override capabilities post-deployment, or off-chain decision logic.

**What BNBuyback is:**
- An execution layer for automated token buybacks
- A stateless, deterministic system for converting revenue to buy pressure
- An on-chain verifiable protocol with no hidden execution paths

**What BNBuyback is not:**
- A token
- A financial instrument
- A guarantee of price performance
- A centralized service

---

## System Architecture

BNBuyback operates as a minimal execution core with the following components:

```
┌─────────────────────┐
│   Revenue Source    │
│ (Protocol Treasury) │
└──────────┬──────────┘
           │
           │ BNB balance accumulation
           │
           ▼
┌─────────────────────┐
│  Execution Core     │
│  - Trigger Logic    │
│  - Safety Checks    │
│  - Execution State  │
└──────────┬──────────┘
           │
           │ Buyback call
           │
           ▼
┌─────────────────────┐
│    DEX Router       │
│  (PancakeSwap, etc) │
└──────────┬──────────┘
           │
           │ Token acquisition
           │
           ▼
┌─────────────────────┐
│   Target Asset      │
│ (Bought tokens)     │
└──────────┬──────────┘
           │
           │ Post-acquisition action
           │
           ▼
┌─────────────────────┐
│  Burn / Treasury /  │
│    Sink Address     │
└─────────────────────┘
```

### Core Components

**Revenue Source**: The origin of BNB used for buybacks. Typically a protocol treasury or fee accumulation contract.

**Execution Core**: Stateless logic that evaluates triggers, enforces constraints, and executes swaps through DEX routers.

**DEX Router Integration**: Interface to on-chain liquidity pools. Executes swaps with slippage protection and minimum output validation.

**Target Asset**: The token being bought back. Post-acquisition, tokens are routed according to configuration (burn, treasury, liquidity sink).

---

## Execution Core

The execution core is a deterministic state machine with the following properties:

- **Stateless Execution**: Each buyback is independent. No persistent state affects future executions beyond configuration parameters.
- **Read-Only Pre-Execution**: All trigger conditions are evaluated using read-only calls to on-chain state.
- **Atomic Execution**: Buyback transactions either complete fully or revert entirely. No partial executions.
- **Non-Discretionary**: Once deployed, execution logic cannot be altered without redeployment.

### Core Functions

```solidity
function executeBuyback(
    uint256 amountBNB,
    uint256 minTokensOut,
    address[] calldata path
) external returns (uint256 tokensBought);

function checkExecutionConditions() external view returns (bool canExecute);

function getNextExecutionTime() external view returns (uint256 timestamp);
```

---

## Buyback Logic

BNBuyback supports two execution modes:

### 1. Fixed-Amount Buybacks

Executes swaps of a predetermined BNB amount at each trigger.

**Configuration:**
```json
{
  "mode": "fixed",
  "amount": "1000000000000000000",  // 1 BNB in wei
  "minOutputPercent": 95              // 5% max slippage
}
```

### 2. Percentage-Based Buybacks

Executes swaps using a percentage of the current BNB balance in the revenue source.

**Configuration:**
```json
{
  "mode": "percentage",
  "percentageOfBalance": 10,           // 10% of balance
  "minOutputPercent": 95,
  "maxAmountPerExecution": "5000000000000000000"  // 5 BNB cap
}
```

### Execution Path

1. **Balance Check**: Query BNB balance of revenue source
2. **Amount Calculation**: Determine execution amount based on mode
3. **Liquidity Validation**: Check DEX pool depth to prevent excessive slippage
4. **Price Validation**: Verify current exchange rate is within acceptable bounds
5. **Execution**: Perform swap through DEX router
6. **Output Validation**: Ensure minimum tokens received
7. **Token Routing**: Transfer acquired tokens to designated sink

---

## Execution Triggers

BNBuyback supports multiple trigger mechanisms:

### Time-Based Triggers

Executes at fixed intervals.

```json
{
  "triggerType": "interval",
  "intervalSeconds": 86400,  // Daily execution
  "startTime": 1704067200    // Unix timestamp
}
```

### Condition-Based Triggers

Executes when on-chain conditions are met.

**Volume Threshold:**
```json
{
  "triggerType": "volumeThreshold",
  "targetPair": "0x...",
  "volumeThreshold": "1000000000000000000000",  // 1000 BNB 24h volume
  "checkIntervalSeconds": 3600
}
```

**Price Threshold:**
```json
{
  "triggerType": "priceThreshold",
  "targetPair": "0x...",
  "priceBelow": "1000000000000000",  // Execute only below price
  "priceAbove": "100000000000000"    // Execute only above price
}
```

**Liquidity Depth:**
```json
{
  "triggerType": "liquidityDepth",
  "minLiquidityBNB": "50000000000000000000",  // 50 BNB minimum liquidity
  "checkIntervalSeconds": 3600
}
```

---

## Safety & Constraints

BNBuyback implements multiple safety mechanisms:

### Slippage Protection

Maximum acceptable slippage per execution. Transaction reverts if output is below threshold.

```solidity
uint256 minTokensOut = (expectedTokens * minOutputPercent) / 100;
require(tokensReceived >= minTokensOut, "Slippage exceeded");
```

### Execution Caps

Maximum BNB amount per single execution. Prevents oversized swaps in low-liquidity conditions.

```json
{
  "maxExecutionAmount": "10000000000000000000",  // 10 BNB max per tx
  "maxDailyAmount": "50000000000000000000"       // 50 BNB max per day
}
```

### Circuit Breakers

Automatic suspension if conditions indicate market manipulation or extreme volatility.

**Price Deviation Circuit Breaker:**
```json
{
  "maxPriceDeviationPercent": 20,  // Halt if price moves >20% from oracle
  "cooldownPeriod": 3600           // 1 hour cooldown after trigger
}
```

**Failed Execution Circuit Breaker:**
```json
{
  "maxConsecutiveFailures": 3,     // Halt after 3 failed executions
  "cooldownPeriod": 7200           // 2 hour cooldown
}
```

### Liquidity Guards

Minimum liquidity requirements before execution.

```solidity
function checkLiquiditySufficient(address pair, uint256 amount) internal view returns (bool) {
    (uint112 reserve0, uint112 reserve1,) = IPancakePair(pair).getReserves();
    uint256 bnbReserve = address(token0) == WBNB ? reserve0 : reserve1;
    return bnbReserve >= amount * minLiquidityMultiplier;
}
```

---

## Configuration Parameters

BNBuyback configuration is set during deployment. Post-deployment modification requires redeployment.

### Core Parameters

```solidity
struct BuybackConfig {
    address revenueSource;           // Source of BNB for buybacks
    address targetToken;             // Token to buy
    address router;                  // DEX router address
    address tokenSink;               // Destination for bought tokens
    
    ExecutionMode mode;              // FIXED or PERCENTAGE
    uint256 executionAmount;         // Amount per execution (if FIXED)
    uint256 percentageOfBalance;     // Percentage (if PERCENTAGE)
    
    TriggerType trigger;             // Trigger mechanism
    uint256 triggerParam;            // Trigger-specific parameter
    
    uint256 minOutputPercent;        // Slippage tolerance
    uint256 maxExecutionAmount;      // Per-transaction cap
    uint256 maxDailyAmount;          // Daily cap
    
    bool useCircuitBreaker;          // Enable circuit breakers
    uint256 maxPriceDeviation;       // Price deviation threshold
    uint256 minLiquidityMultiplier;  // Liquidity requirement multiplier
}
```

### Example Configuration

```solidity
BuybackConfig({
    revenueSource: 0x1234...,
    targetToken: 0x5678...,
    router: 0xabcd...,
    tokenSink: 0xef01...,
    
    mode: ExecutionMode.PERCENTAGE,
    executionAmount: 0,
    percentageOfBalance: 15,  // 15% of balance
    
    trigger: TriggerType.INTERVAL,
    triggerParam: 86400,  // Daily
    
    minOutputPercent: 95,  // 5% slippage tolerance
    maxExecutionAmount: 5 ether,
    maxDailyAmount: 20 ether,
    
    useCircuitBreaker: true,
    maxPriceDeviation: 15,  // 15% max deviation
    minLiquidityMultiplier: 20  // 20x liquidity requirement
})
```

---

## Transparency & On-Chain Verifiability

All BNBuyback executions are fully transparent and verifiable on-chain.

### Event Logging

Every execution emits detailed events:

```solidity
event BuybackExecuted(
    uint256 indexed executionId,
    uint256 bnbAmount,
    uint256 tokensReceived,
    uint256 pricePerToken,
    address executor,
    uint256 timestamp
);

event ExecutionFailed(
    uint256 indexed executionId,
    string reason,
    uint256 timestamp
);

event CircuitBreakerTriggered(
    string reason,
    uint256 cooldownUntil,
    uint256 timestamp
);
```

### State Queries

Read-only functions for external verification:

```solidity
function getTotalBuybackAmount() external view returns (uint256);
function getTotalTokensAcquired() external view returns (uint256);
function getExecutionCount() external view returns (uint256);
function getLastExecutionTime() external view returns (uint256);
function getCircuitBreakerStatus() external view returns (bool active, uint256 cooldownEnd);
```

### Audit Trail

All transactions are traceable via:
- Transaction hash
- Block number
- Execution parameters
- Input/output amounts
- Gas consumption

No off-chain components. No trusted oracles required for core execution (optional for advanced triggers).

---

## Example Execution Flow

### Scenario: Daily Percentage-Based Buyback

**Initial State:**
- Revenue source balance: 10 BNB
- Configuration: 10% of balance, daily execution
- Last execution: 24 hours ago
- Target token: PROJECT_TOKEN

**Execution Sequence:**

```
1. Trigger Check
   ├─ Current time: 1704153600
   ├─ Last execution: 1704067200
   ├─ Interval: 86400 seconds
   └─ Status: TRIGGERED

2. Pre-Execution Validation
   ├─ Balance check: 10 BNB available
   ├─ Amount calculation: 10 BNB * 10% = 1 BNB
   ├─ Max amount check: 1 BNB < 5 BNB cap ✓
   ├─ Liquidity check: Pool has 50 BNB ✓
   ├─ Circuit breaker: INACTIVE ✓
   └─ Status: APPROVED

3. Swap Execution
   ├─ Input: 1 BNB
   ├─ Path: [WBNB, PROJECT_TOKEN]
   ├─ Expected output: 100,000 PROJECT_TOKEN
   ├─ Min output: 95,000 PROJECT_TOKEN (5% slippage)
   └─ Actual output: 98,500 PROJECT_TOKEN

4. Post-Execution
   ├─ Tokens received: 98,500 PROJECT_TOKEN
   ├─ Transfer to sink: 0xef01...
   ├─ Event emitted: BuybackExecuted(...)
   └─ Status: SUCCESS

5. State Update
   ├─ Total bought: +98,500 PROJECT_TOKEN
   ├─ Execution count: +1
   ├─ Last execution time: 1704153600
   └─ Next execution: 1704240000
```

**Transaction Output:**
```
BuybackExecuted(
    executionId: 42,
    bnbAmount: 1000000000000000000,
    tokensReceived: 98500000000000000000000,
    pricePerToken: 10152284263959390,
    executor: 0x9876...,
    timestamp: 1704153600
)
```

---

## Risks & Limitations

BNBuyback is infrastructure. It executes deterministically according to configuration. Users must understand the following:

### Execution Risks

**Slippage**: Large executions in low-liquidity pools may experience significant slippage despite protections.

**MEV Exposure**: On-chain execution is visible before confirmation. Front-running or sandwich attacks are possible.

**Gas Costs**: Execution requires gas. Failed transactions still consume gas.

**Oracle Dependency**: If using price-based triggers, accuracy depends on oracle reliability.

### Market Risks

**Price Impact**: Buybacks create buy pressure but do not guarantee sustained price increases.

**Liquidity Fragmentation**: Execution across multiple DEXs may be necessary for optimal pricing but increases complexity.

**Market Conditions**: Extreme volatility may trigger circuit breakers, preventing execution.

### Technical Risks

**Smart Contract Risk**: Bugs or vulnerabilities in the execution core or DEX routers.

**Network Congestion**: High gas prices or network issues may delay or prevent execution.

**Configuration Error**: Incorrect parameters at deployment cannot be changed without redeployment.

### Regulatory Considerations

**Jurisdiction-Dependent**: Automated buybacks may have regulatory implications depending on token classification and jurisdiction.

**No Financial Advice**: This system does not provide investment recommendations or guarantees.

---

## Non-Custodial Design

BNBuyback does not custody funds. Key properties:

### No Admin Keys

Once deployed with configuration, the execution core:
- Cannot redirect funds to alternate addresses
- Cannot modify execution parameters
- Cannot prevent execution when conditions are met
- Cannot extract BNB or tokens from the system

### Transparent Ownership

```solidity
// Revenue source maintains custody
function executeBuyback() external {
    uint256 amount = calculateAmount();
    
    // Pull funds only during execution
    IERC20(BNB).transferFrom(revenueSource, address(this), amount);
    
    // Execute swap
    uint256 tokensReceived = executeSwap(amount);
    
    // Transfer immediately to sink
    IERC20(targetToken).transfer(tokenSink, tokensReceived);
}
```

Funds are pulled from source, swapped, and transferred to sink in a single atomic transaction. No funds remain in the execution core.

### Sink Configuration

Token sink can be:
- Burn address (0x000...000)
- Protocol treasury
- Liquidity lock contract
- Any address specified at deployment

Sink address is immutable post-deployment.

---

## License & Disclaimer

### License

BNBuyback is released under the MIT License.

### Disclaimer

THIS SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

**Not Financial Advice**: BNBuyback is infrastructure. It does not constitute financial, investment, trading, or legal advice. Users are responsible for understanding applicable laws and regulations in their jurisdiction.

**No Guarantees**: Buyback execution does not guarantee token price appreciation, liquidity, or any financial outcome.

**Audit Status**: This project should be independently audited before production use. Use at your own risk.

**Smart Contract Risk**: Interaction with blockchain systems carries inherent risk including but not limited to loss of funds due to bugs, exploits, or network issues.

---

## Technical Support

For technical inquiries, implementation questions, or bug reports, please open an issue in this repository.

For security vulnerabilities, please contact the maintainers directly.

---

**BNBuyback**: Deterministic buyback execution infrastructure for BNB Chain.

Repository maintained by the BNBuyback development team.