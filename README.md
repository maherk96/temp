# Test Coverage Platform - FX Trading Example

## 🎯 Where Do Requirements Come From?

### The Hierarchy (FX Trading Platform)

```
Coverage Spec: "FX Trading Platform Q1 2026"
  │
  ├── Pillar: Trading Operations
  │   ├── Capability: Order Execution
  │   │   ├── TRD-001: Execute market order (HIGH risk)
  │   │   ├── TRD-002: Execute limit order (HIGH risk)
  │   │   └── TRD-003: Cancel pending order (MEDIUM risk)
  │   │
  │   └── Capability: Position Management
  │       ├── POS-001: Real-time P&L calculation (HIGH risk)
  │       └── POS-002: Margin requirements (HIGH risk)
  │
  ├── Pillar: Risk & Compliance
  │   ├── Capability: Pre-Trade Risk Controls
  │   │   ├── RISK-001: Position limit enforcement (HIGH risk)
  │   │   └── RISK-002: Margin check before trade (HIGH risk)
  │   │
  │   └── Capability: Regulatory Compliance
  │       ├── REG-001: MiFID II transaction reporting (HIGH risk)
  │       └── REG-002: Best execution policy (HIGH risk)
  │
  └── Pillar: Market Data & Pricing
      └── Capability: Real-Time Pricing
          ├── PRICE-001: EUR/USD price feed (HIGH risk)
          └── PRICE-002: Stale price detection (HIGH risk)
```

### Sources of Requirements

1. **Business Requirements**
   - Product: "Traders need to execute EUR/USD spot trades"
   - Becomes: TRD-001, TRD-002

2. **Regulatory Mandates**
   - MiFID II: "Report all transactions within 60 seconds"
   - Becomes: REG-001 (HIGH risk, must automate)

3. **Risk Management**
   - Risk Team: "Must validate margin before every trade"
   - Becomes: RISK-002 (HIGH risk, requires manual + automated)

4. **Incident Response**
   - Production Issue: "Trader executed beyond position limit"
   - Becomes: RISK-001 (HIGH risk, needs coverage)

---

## 👥 Who Uses This Platform?

### 1. **QA Lead** 🎯
**Creates coverage strategy from business/regulatory requirements**

**Workflow**:
```
1. Receives Q1 release requirements:
   - New: GBP/USD trading pair
   - Enhanced: Pre-trade risk controls
   - Regulatory: MiFID II compliance

2. Creates Coverage Spec: "FX Platform Q1 2026"

3. Translates to Coverage Items:
   Product Req → TRD-004: "Execute GBP/USD spot"
   Risk Req   → RISK-003: "Enhanced margin calculation"
   Compliance → REG-003: "MiFID II timestamp accuracy"

4. Sets policies:
   - HIGH-risk items: Require both automated + manual
   - Regulatory items: Require manual approval
```

---

### 2. **Automation Engineer** 🤖
**Links automated tests to coverage items**

**Example**:
```java
@Test
@Tag("covers:TRD-001")  // Market order execution
@Tag("covers:RISK-002") // Margin validation
public void testMarketOrderWithMarginCheck() {
    // 1. Setup: Account with $10K margin
    // 2. Action: Place EUR/USD market order for $50K
    // 3. Verify: Order rejected (insufficient margin)
}
```

**Result**: When this test runs in CI/CD, coverage for TRD-001 and RISK-002 automatically tracked.

---

### 3. **Manual Tester** 🧪
**Validates critical/regulatory scenarios automation can't cover**

**Example Session**:
```
Manual Test Session: "Pre-Trade Risk Controls - Manual Validation"
Linked to: Q1 2026 Spec

RISK-001: Position limit enforcement
├── Test 1: Soft limit warning → PASS
│   Note: "Warning displayed correctly at 80% of limit"
├── Test 2: Hard limit rejection → PASS
│   Note: "Trade blocked at 100% limit, error message clear"
└── Test 3: Multi-currency limit → FAIL
    Note: "Bug found: EUR+GBP positions not aggregated"
    Screenshot attached

Status: SUBMITTED → Awaiting QA Lead approval
```

---

### 4. **Trading Desk / Business Users** 💼
**Care about operational readiness**

**Use Case**:
```
Trading Desk Manager asks: "Can we launch GBP/USD trading?"

Checks Coverage Report:
  GBP/USD Trading
  ├── TRD-004: Execute GBP/USD spot
  │   ✅ COVERED (automated: 5 days ago, manual: 3 days ago)
  ├── PRICE-003: GBP/USD price feed
  │   ⚠️ COVERED (automated only, no manual validation)
  └── RISK-004: GBP/USD position limits
      ✅ COVERED (both validated)

Decision: "Launch, but flag PRICE-003 for manual testing next week"
```

---

### 5. **Compliance Officer** 📋
**Ensures regulatory requirements are tested**

**Dashboard View**:
```
MiFID II Compliance Coverage
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Coverage: 95% (19/20 items)

⚠️ Requires Attention:
• REG-001: Transaction reporting
  - Automated: ✅ Passing
  - Manual: ⚠️ Expired (45 days ago)
  - Policy: HIGH-risk regulatory item requires recent manual validation

Action: Schedule manual compliance testing before audit
```

---

### 6. **Head of QA / Engineering Manager** 📊
**Makes release decisions**

**Release Checklist**:
```
Q1 2026 Release Readiness
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Overall Coverage: 97% (58/60 items)

HIGH-Risk Items:
  ✅ 15/15 Trading operations
  ✅ 8/8 Risk controls
  ⚠️ 7/9 Regulatory compliance
  
Policy Warnings: 2
  ⚠️ REG-001: Manual validation expired
  ⚠️ RISK-002: Security penetration test pending

Decision: 
  ✅ Trading features: APPROVED
  ⚠️ Compliance: HOLD until manual testing complete
```

---

## 🔄 Real-World Workflow

### Sprint: New Currency Pair Launch (GBP/USD)

**Week 1: Requirements → Coverage**
```
Product Manager:
└── "Launch GBP/USD trading by end of Q1"

Compliance:
└── "Must meet MiFID II requirements"

QA Lead:
└── Creates Coverage Items:
    ├── TRD-004: Execute GBP/USD trades
    ├── PRICE-003: GBP/USD pricing
    ├── RISK-004: GBP/USD position limits
    └── REG-003: GBP/USD reporting
```

**Week 2-3: Testing**
```
Automation Engineer:
├── Writes E2E tests for GBP/USD
├── Tags with @Tag("covers:TRD-004")
└── Tests run on every commit

Manual Tester:
├── Tests edge cases:
│   - Network failure during trade
│   - Price spike handling
│   - Off-hours trading
└── Records results in manual session
```

**Week 4: Sign-off**
```
Coverage Report:
  ✅ All 4 items covered
  ✅ No policy warnings
  ✅ Both automated + manual evidence

Head of QA:
└── "Approved for production"

Trading Desk:
└── Launches GBP/USD trading
```

---

## 📊 Quick Reference

### Requirements Sources (FX Trading)
- **Product**: New trading features, UX improvements
- **Risk**: Position limits, margin rules, circuit breakers
- **Compliance**: MiFID II, Dodd-Frank, EMIR reporting
- **Operations**: Order routing, execution quality
- **Technology**: Performance SLAs, disaster recovery

### Users & Their Focus
| User | Primary Goal | Key Metric |
|------|--------------|------------|
| QA Lead | Define what needs testing | Coverage Items created |
| Automation Engineer | Automate critical paths | Test execution rate |
| Manual Tester | Validate edge cases | Sessions approved |
| Trading Desk | Operational confidence | Feature readiness |
| Compliance | Regulatory adherence | Regulatory coverage % |
| Head of QA | Release decisions | Overall coverage + warnings |

### Value Proposition
- **Before**: "Are we ready to launch GBP/USD?" → Gut feel, email chains, spreadsheets
- **After**: "Are we ready to launch GBP/USD?" → Click button, see 97% covered with 2 warnings, make data-driven decision

---

## 🎯 Bottom Line

**What**: Platform links business/regulatory requirements → test evidence

**Who**: QA defines requirements, Engineers test, Testers validate, Managers decide

**Why**: Confidence in release readiness + regulatory compliance + audit trail

**Example**: 
- Requirement: "MiFID II transaction reporting"
- Becomes: REG-001 (HIGH risk, requires automation + manual approval)
- Evidence: 500 automated tests daily + manual compliance test quarterly
- Result: Audit-ready documentation + release confidence
