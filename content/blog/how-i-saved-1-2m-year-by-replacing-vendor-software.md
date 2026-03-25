---
title: "How I Saved $1.2M/Year by Replacing Vendor Software"
date: 2025-03-25
---

Last quarter, my team at Warner Bros. Discovery completed a project that eliminated a $1.2M annual contract. We replaced an external vendor's SOX compliance controls with an in-house solution.

This post explains how we approached it, the technical decisions, and lessons learned about build vs. buy.

## The Situation

Our financial reporting platform was using a third-party SOX compliance tool. The product worked, but we had several problems:

- **Annual cost**: $1.2M in licensing fees
- **Customization limits**: We couldn't adapt to changing compliance requirements
- **Vendor lock-in**: Any enhancement request meant waiting for their roadmap
- **Integration friction**: The tool was designed for generic cases, not our specific needs

My team (4 engineers) was tasked with evaluating alternatives.

## The Decision Matrix

We considered three options:

| Option | Cost | Timeline | Risk | Verdict |
|---------|------|----------|-------|----------|
| Status quo (keep vendor) | $1.2M/year | None | Low | ❌ No change |
| Negotiate with vendor | $1M/year (best case) | 3-6 months | Medium | ❌ Still locked in |
| Build internal | $200K first year, $50K ongoing | 3-4 months | Medium | ✅ Full control |

The analysis was clear: we needed to build. But the real question wasn't cost—it was risk. Could we deliver a reliable solution in 4 months?

## Technical Architecture

Here's what we built:

### Core Components

**1. Rule Engine**
```python
# Simplified representation of our rule engine
class ComplianceRule:
    def __init__(self, name, check_function, severity):
        self.name = name
        self.check = check_function
        self.severity = severity

    def validate(self, transaction):
        result = self.check(transaction)
        if not result.is_valid:
            Alert.send(
                rule=self.name,
                transaction=transaction.id,
                severity=self.severity,
                details=result.errors
            )
        return result
```

The rule engine is pluggable—when compliance requirements change, we add a new rule without touching core code.

**2. Data Validation Pipeline**
- Source: Financial transactions from our data lake (Databricks)
- Processing: PySpark jobs validate against ruleset
- Output: Compliance reports + exception logs
- Storage: Results stored in Postgres for audit trails

**3. Dashboard & Alerts**
- Real-time monitoring of rule execution
- Alerting on critical violations (high severity)
- Historical trend analysis for compliance officers

### Key Design Decisions

**Rule-based vs. Code-based**

We chose a rule-based engine instead of hardcoding logic. Why?

- **Flexibility**: Compliance rules change frequently. Business users can update rules in JSON config without code changes.
- **Auditing**: Every rule has metadata (who created, when, why).
- **Testing**: Each rule can be tested independently.

**Batch vs. Streaming**

Our validation runs in batch (daily), but we architected for streaming:

```scala
// Structured for eventual migration to streaming
trait ComplianceProcessor {
    def processBatch(transactions: Seq[Transaction]): Report
    def processStream(eventStream: KStream[Transaction]): Unit
}
```

This design lets us move to Apache Flink in the future without rewriting business logic.

## The Implementation

**Month 1: Requirements & Prototype**
- Interviewed compliance team to capture all rules
- Built PoC with 5 representative rules
- Validated against historical data (matched vendor results 98%)

**Month 2-3: Full Development**
- Implemented rule engine (Python)
- Built PySpark pipeline (PySpark, Databricks)
- Created dashboard (Streamlit for internal tooling)

**Month 4: Testing & Rollout**
- Load tested against 1M transactions (peak volume)
- Parallel run with vendor tool for 2 weeks
- Signed off by compliance team

## Challenges & Lessons

### Challenge 1: Rule Explosion

We discovered 200+ compliance rules, not the 50 we initially documented.

**Solution**: We categorized rules and grouped by domain:
```
- Financial rules (reconciliation, approval chains)
- Data quality rules (format validation, deduplication)
- Operational rules (audit trails, access control)
```

**Lesson**: Scope estimation for compliance systems is hard. Assume 3x initial rule count.

### Challenge 2: False Positives

Initial runs flagged 8% of transactions as non-compliant. The vendor tool flagged 2%.

**Root cause**: Our rules were too strict. We interpreted "within X days" as "exact X days," not "≤ X days."

**Fix**: Adjusted rule logic and added exception handling.

**Lesson**: Production data reveals edge cases you never think about in design meetings.

### Challenge 3: Performance at Scale

Our first PySpark job took 4 hours to process a day's transactions. The vendor tool did it in 45 minutes.

**Optimization**:
1. Partitioned by business unit (4 partitions, parallel processing)
2. Cached reference data (account metadata, exchange rates) in memory
3. Reduced shuffle operations (broadcast variables instead of joins)

**Result**: 38 minutes. Faster than the vendor.

**Lesson**: Profiling beats guessing. We spent 1 week optimizing until we profiled and found the real bottleneck (shuffle).

## The Outcome

After 4 months, we migrated fully.

### Metrics

| Metric | Before | After | Improvement |
|--------|---------|--------|-------------|
| Annual cost | $1.2M | $50K | 96% reduction |
| Compliance check time | 45 min | 38 min | 16% faster |
| False positive rate | 2% | 2.1% | Comparable |
| Time to add new rule | 4-8 weeks | 2 days | 85% faster |

### Non-Metric Benefits

- **Agility**: We can respond to new compliance requirements in days, not quarters.
- **Customization**: Every rule is tailored to our business context.
- **Ownership**: My team now owns the codebase—no vendor negotiations.

## Takeaways

### 1. Build vs. Buy is a Spectrum, Not a Binary

We didn't build from scratch—we built on top of PySpark and Databricks. The decision was about **where to capture value**:
- Off-the-shelf: Vendor captures value through licensing
- In-house: We capture value through customization and agility

### 2. Start with Parallel Run

The 2-week parallel run saved us. We discovered the false positive issue before full migration, which would have caused chaos.

### 3. Document the "Why"

Every architectural decision document (ADR) includes:
- Context
- Decision
- Consequences

When the compliance team asked "why not use rule X?" six months later, our ADR explained the trade-offs.

### 4. Cost Savings != Value Delivery

Saving $1.2M got management attention, but the real value is speed. We can now iterate on compliance in days.

## What I'd Do Differently

1. **Start with user research earlier**: We interviewed compliance team in month 1, but we should have done it in the decision phase.
2. **Build observability first**: We added monitoring after issues arose. It should have been built-in.
3. **Plan for rule growth**: We designed for 50 rules, not 200. Scalability was a pleasant surprise.

---

This project taught me that replacing a vendor is about more than cost—it's about owning your destiny. When you own the code, you own the response time.

**What's your experience with build vs. buy? I'd love to hear your stories.**
