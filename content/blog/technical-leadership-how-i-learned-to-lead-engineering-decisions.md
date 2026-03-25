---
title: "Technical Leadership: How I Learned to Lead Engineering Decisions"
date: 2025-03-22
---

I've been leading teams for 4 years now—first at Phenom (7 engineers), now at Warner Bros. Discovery (4 engineers).

Early on, I made mistakes as a leader. I micromanaged. I made decisions in isolation. I avoided conflict.

This post is about what I learned about leading technical teams.

## The Micromanagement Trap

At Phenom, when I was first promoted to lead, I did this:

```
Task assignment:
  - "Dev A, you work on this feature"
  - "Dev B, you work on that feature"
  - "Dev C, you're on bug fixes this sprint"
```

**The problem**:

My team stopped thinking. They waited for instructions. When I went on vacation, velocity dropped 50%.

**What changed**:

I shifted to outcome-based leadership:

```
Outcome definition:
  - "Team, our goal is to ship the data integration platform by June"
  - "We need to onboard 200 customers, support 500 workflows"
  - "I'm here to unblock you, not direct you"
```

Engineers self-organized. They picked features. They collaborated.

**Velocity increased by 40%**.

**Lesson**: Define the "what" and "why," let the team figure out the "how."

## The Decision Vacuum

Early on, I made technical decisions alone:

```
Leadership:
  - Srivatsav: Choose between Apache Flink and Spark Streaming
  - Srivatsav: 30 minutes
  - Srivatsav: We're using Flink
  - Team: Okay?
```

**The problem**:

Two issues:

1. **Buy-in**: Team didn't own the decision. If they disagreed, they didn't speak up.
2. **Blind spots**: I missed considerations the team would have caught.

Example: I chose Flink for a project. A junior engineer asked "What about Spark Streaming? We have more experience there." I dismissed it. Three months later, we hit a Spark-specific performance issue we could have debugged faster.

**What changed**:

We introduced ADRs (Architectural Decision Records) and required team input:

```markdown
# ADR-007: Streaming Framework Selection

## Context
We need to process 1M events/day with sub-second latency.

## Decision
Use Apache Flink.

## Considered Alternatives
- **Spark Streaming**: Rejected due to higher memory overhead for our workload (Team input from Srivatsav)
- **Akka Streams**: Rejected due to team unfamiliarity and lack of mature state management
- **Flink**: Selected for exactly-once semantics and lower memory footprint (Team consensus)

## Consequences
### Positive
- Lower operational cost (smaller cluster)
- Better exactly-once guarantees
- Team has reference documentation

### Negative
- Learning curve (estimated 2 weeks ramp-up)
- Fewer examples in production at our company
```

The ADR didn't just record the decision—it recorded **why** we rejected alternatives, including team input.

**Lesson**: Document decisions. Require input. Make rationale visible.

## The "Good Enough" Conflict

I struggled with perfectionism early on. I wanted every solution to be optimal.

At Phenom, we were building a low-code platform. The data model was complex. I spent 3 months designing a "perfect" schema.

**The problem**:

While I designed, a competitor launched a simpler solution and signed customers we wanted.

**What I learned**:

### 1. Speed > Perfection

Shipping "good enough" in January beats "perfect" in June.

### 2. Iterate

```
Perfect design (6 months) → v1.0
Good design (2 months) → v1.0 → feedback → v1.1 → v1.2 (same 6 months)
```

The second path gets to v1.2 faster and v1.2 is often better than perfect v1.0 because of real-world feedback.

### 3. Technical Debt is Negotiable

Some shortcuts are okay:

```java
// Technical debt accepted for speed
public void processDataLegacy(Data data) {
    // TODO: Refactor to use new schema (ticket #1234)
    processUsingOldSchema(data);
}
```

Document it, track it, pay it back.

**Lesson**: Perfectionism delays learning. Ship fast, iterate, improve based on feedback.

## Managing Up and Out

I've managed two types of expectations: my team's expectations of me, and management's expectations of the team.

### Managing the Team Up

Early on, I promised things I couldn't deliver:

```
Leadership:
  - Srivatsav: "We'll have the feature done by Friday"
  - Reality: Blocked by dependency, delayed to next Wednesday
  - Result: Trust damage
```

**What changed**:

I became transparent about uncertainty:

```
Leadership:
  - Srivatsav: "I think we can ship Friday, but there's a dependency on team B that I haven't confirmed. I'll check with them today and confirm by EOD."
  - Outcome: Managed expectations, team appreciated honesty
```

**Lesson**: Under-promise, over-deliver. Be clear about uncertainty.

### Managing the Management Out

I've been in conversations where engineering needs were presented as "blocking product":

```
Engineering:
  - "This API change will take 3 months. We need to refactor the data layer."
  - Product: "Can't you just add a wrapper? We're launching next quarter."
  - Engineering: "...I'll try"
```

This sets up failure. The wrapper becomes technical debt. The team resents the pressure.

**What changed**:

I started presenting options with trade-offs:

```
Engineering:
  - "Here are two options:
  1. Wrapper API (2 weeks, but will need refactoring later)
  2. Refactor data layer (3 months, but sustainable)
  3. Hybrid: Wrapper now, refactored data layer in parallel (6 weeks total, but we can launch in 2 weeks)

  My recommendation: Option 3
  Reason: Meets timeline, avoids tech debt, builds right foundation"

  Product: "I see the trade-offs. Let's go with option 3."
```

**Lesson**: Present choices, not problems. Make trade-offs explicit. Get product involved in technical decisions.

## The Growth Mindset

I've mentored 3 junior engineers formally, and many informally. Here's what works:

### 1. Set Context, Not Tasks

Instead of "fix bug #1234," explain:

```
Mentoring:
  - "We have a bug in the payment processing flow. Customers report duplicate charges when they retry.
  - The payment gateway is idempotent, so the issue is in our retry logic.
  - Look at PaymentService.java, line 156-210.
  - Let me know if you need help tracing through the logs."
```

**Lesson**: Context enables autonomy. Tasks produce dependency.

### 2. Code Reviews as Teaching, Not Policing

I've seen two types of code reviews:

```
Bad code review:
  - "Change line 42. Why did you use ArrayList? Use List."
  - "This method name is confusing. Rename it."
  - Result: Engineer is defensive, doesn't ask questions

Good code review:
  - "Great work on the idempotency here. That was a hard problem.
  - "One question: What happens if the database transaction rolls back here? We might need a compensating action."
  - "For next time, consider splitting this method—it's getting long."
  - Result: Engineer is open to feedback, learns from patterns
```

**Lesson**: Praise what works. Ask questions before critiquing. Frame feedback as teaching, not policing.

### 3. Public Wins

I used to keep praise private:

```
Private praise:
  - "Dev A, you did great work on that feature. I'll mention it in your performance review."
  - Problem: No team recognition. Dev B doesn't see what good looks like.
```

Now I'm public:

```
Public praise:
  - Slack: "@team Great work launching the data integration platform! Special shoutout to Dev A for the idempotency fix—that was tricky. Also Dev B for the performance optimization, we shaved 30% off latency."
```

**Lesson**: Praise publicly, correct privately. The whole team learns from wins.

## The Hardest Lesson: Letting Go

This is the one I'm still working on.

When I was at Phenom, I led a team of 7 engineers. Two of them were exceptional. They took on responsibility. They owned features end-to-end.

After I was promoted and moved to Warner Bros., I missed working with them.

But I also realized: I needed to let go.

- I can't micromanage from another company
- They're ready to lead their own teams
- My role changed from "doer" to "mentor"

**Lesson**: Success of your team is your success. When they outgrow you, that's a victory, not a loss.

## Framework I Use Now

When making technical leadership decisions:

```
1. Define the problem (clear, measurable)
2. Gather team input (diverse perspectives)
3. Evaluate alternatives (pros/cons)
4. Decide with rationale (document it)
5. Communicate decision (why, what, when)
6. Monitor outcomes (was it right?)
7. Iterate (change if needed)
```

This isn't a straight line—it's a loop. Decision → Feedback → Iteration.

## What I'd Tell New Engineering Leaders

1. **Your job is to enable, not execute**
   - The best code you write today is obsolete. The best team you build lasts.

2. **Lead by example**
   - When I say "let's ship quality," I ship quality.
   - When I say "let's document," I document.
   - Hypocrisy is visible instantly.

3. **Protect your team**
   - Fight unrealistic deadlines
   - Challenge bad decisions (even from leadership)
   - Celebrate wins publicly

4. **Invest in people**
   - Mentorship isn't "nice to have"—it's your core responsibility
   - A great engineer who leaves is a failure of leadership

5. **Be wrong confidently**
   - You will be wrong. Be transparent about it.
   - "I was wrong about X, here's what I learned" earns more respect than pretending to be right.

## Conclusion

Technical leadership is about making space for engineers to do their best work. It's not about being the best coder—it's about being the best multiplier.

When your team succeeds, you succeed. When your team fails, you failed.

---

**What leadership lessons have you learned? What would you add to this framework?**
