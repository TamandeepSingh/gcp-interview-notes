# Behavioral Questions - Samsung DevOps Interview

## Question 1: Tell Me About a Critical Production Incident

### The Question
"Tell me about a critical production incident you handled. Walk me through what happened, how you responded, and what you learned. Don't minimize it - we want to hear about REAL problems."

### What Interviewer is Evaluating

✓ How do you handle pressure?
✓ Do you take ownership or blame others?
✓ How deep is your technical understanding?
✓ Can you communicate complex issues clearly?
✓ Do you learn from mistakes?
✓ Did you prevent it from happening again?

### Strong Answer Framework

```
Structure: SITUATION → COMPLICATION → RESOLUTION → LEARNING

SITUATION (30 seconds):
"In March, we had a production incident where our database failover 
failed, and we lost 15 minutes of user data. We support 50 million 
concurrent users across 3 regions, and the primary us-east region's 
database suddenly went down due to a failed disk."

COMPLICATION (60 seconds):
"The issue was multi-layered:
1. Automated failover didn't trigger because our health check was 
   misconfigured - it only checked connection, not data availability
2. When we manually initiated failover to standby, the replica was 
   2 minutes behind due to replication lag we weren't monitoring
3. Some in-flight transactions were lost
4. During the panic, we didn't follow our runbook and made it worse"

MY RESPONSE (90 seconds):
"Here's exactly what I did:
- T+0: Paged the on-call team and escalated to VP (transparency)
- T+2: Immediately halted new writes to prevent data drift
- T+5: Manually promoted read replica to primary
- T+8: Redirected read traffic to secondary using DNS failover
- T+12: Investigated root cause in parallel (didn't rush back online)
- T+15: Made manual data reconciliation to restore lost transactions

I personally led the investigation and was transparent with stakeholders 
every 2 minutes about our status and ETA."

LEARNING (60 seconds):
"What I learned and implemented:

1. Monitoring gap: We weren't monitoring replication lag. I added:
   - Prometheus alert: lag > 5 seconds
   - Dashboard showing real-time replication status
   
2. Health check was wrong: Changed from simple connection test to:
   - Query test (SELECT 1 WITH ROW LOCK)
   - Replication lag check
   - Actually testing the test to ensure it works
   
3. Runbook was stale: Updated it with:
   - Exact decision tree for failover scenarios
   - Added automated failover for clear-cut failures
   - Team drilled the runbook monthly afterward
   
4. Process: We initiated quarterly DR drills where we simulate
   failures under controlled conditions
   
6 months later, we had another database failure. Our new monitoring 
caught it immediately, automated failover worked, no data loss.
That told me the fixes were solid."

KEY PHRASES TO USE:
- "I took ownership that..." (not "we" - show personal involvement)
- "The root cause was..." (technical depth)
- "I learned that..." (growth mindset)
- "We implemented..." (brought team along)
- "Monitoring showed..." (data-driven)
- "Next time..." (prevention mindset)
```

### What NOT to Say

❌ "Everything was working correctly, no changes needed"
- Shows lack of post-incident reflection

❌ "It was definitely the network, not our code"
- Blame-shifting, not confidence-building

❌ "We never heard what the issue was"
- Lack of follow-through

❌ "I wasn't really involved in the fix"
- Shows lack of ownership

❌ "We just waited until someone else fixed it"
- Passivity, not leadership

### Follow-up Questions

**Q1: How did you communicate with customers during the incident?**
- Expected: Specific timeline, transparency, regular updates

**Q2: Did you have to make any tradeoffs? How did you decide?**
- Expected: Tradeoff analysis, business context understanding

**Q3: If it happened again, what would you do differently?**
- Expected: Lessons learned applied, continuous improvement

---

## Question 2: Disagreement with Senior Engineer

### The Question
"Tell me about a time you disagreed with someone more senior than you (your manager, architect, etc.) about a technical decision. How did you handle it?"

### What Interviewer is Evaluating

✓ Can you stand by your convictions with evidence?
✓ Are you respectful to hierarchy while thinking independently?
✓ Can you communicate technical ideas persuasively?
✓ Do you listen to other perspectives?
✓ Can you admit when you're wrong?
✓ How do you handle conflict?

### Strong Answer Framework

```
Structure: SITUATION → MY POSITION → SENIOR'S RATIONALE → RESOLUTION

SITUATION (30 seconds):
"I was on a team managing Kubernetes clusters. Our VP of Engineering 
proposed upgrading from GKE 1.20 to 1.26 (6 major versions at once) 
to access new features we needed."

MY POSITION (45 seconds):
"I believed we should upgrade incrementally (1.20→1.22→1.24→1.26) because:

1. Kubernetes API deprecations between versions are significant
   - 1.20-1.22: beta APIs removed
   - Our services used deprecated APIs I identified in our codebase
   
2. My data: I examined our cluster specs and found ~47 resources using 
   apis/v1beta3 which would be removed in 1.24
   
3. Risk: Single 6-version jump = all 47 resources would break
   
I proposed: 3 minor upgrades with 2-week validation between each
Time: roughly the same (3 weeks total - we only had 3 weeks anyway)
Safety: Catch issues immediately after each step"

SENIOR'S RATIONALE (30 seconds):
"Our VP said:
- 'Engineering speed matters more than perfect safety'
- 'We can handle breaking changes as they come'
- 'Support for 1.20 ends next month anyway'

Valid points, but I thought he hadn't considered the specific 
deprecations in our code."

RESOLUTION (60 seconds):
"I did NOT just argue back and forth. I:

1. Requested 15 minutes to prepare a comparison:
   - Listed all deprecated APIs in our clusters
   - Showed which services would break with each version
   - Calculated effort to fix: ~4 engineering-days
   
2. Created a written proposal with:
   - Timeline (3 vs 1 week)
   - Risk assessment (1 vs 3+ failure points)
   - Cost in engineering time (4 days vs 2-3 days in firefighting)
   - Fallback plan for each scenario
   
3. Scheduled a 1:1, shared the data, explained my reasoning
   - Not emotional ('but it's risky!')
   - Technical ('here's what breaks, here's when')
   
4. Was ready to accept his decision
   - Said: 'If you think speed is worth the risk, I'll own the fallout'
   - But presented the data clearly

OUTCOME:
He changed his mind. We did incremental upgrades. Found 2 issues
in 1.22 that required code changes. Would have been chaos in 1.26.

LEARNING:
I learned that:
- Data > opinions (even from senior people)
- Disagreement is healthy if it's respectful and data-driven
- Documentation matters. I had written everything down
- I should have escalated earlier with the evidence"

KEY PHRASES:
- "I had data showing..."
- "I proposed an alternative because..."
- "I respected their perspective, and also..."
- "I owned the outcome"
- "I learned that disagreement is valuable when handled professionally"
```

### What NOT to Say

❌ "I told him he was wrong"
- Disrespectful, not collaborative

❌ "I fought them on it and eventually won"
- Adversarial framing, not team mentality

❌ "I just did what they said even though it was dumb"
- Passive, no technical advocacy

❌ "It blew up in their face and I was right"
- Vindictive, not constructive

❌ "I still think they made the wrong call"
- Shows holding a grudge

---

## Question 3: Learning Something New Under Pressure

### The Question
"Tell me about a time you had to learn something new very quickly to solve a production problem. What was it? How did you approach it?"

### What Interviewer is Evaluating

✓ Can you learn fast?
✓ Do you stay calm under pressure?
✓ Is your learning self-directed?
✓ How do you research and debug?
✓ Do you ask for help when appropriate?

### Strong Answer Framework

```
SITUATION (30 seconds):
"Our company decided to use Helm for deployments. I had used Kubernetes 
but never Helm. A critical deployment broke because of a Helm value 
override issue, and no one else on my team knew Helm. It was 2 PM on 
Friday before a weekend release window."

CHALLENGE (30 seconds):
"The problem was:
- 20 services deployed via Helm, 2 completely broken
- Error: 'Expected map[string]string, got string'
- Deployments rolling back, user-facing services failing
- Helm documentation was complex and unfamiliar
- I had 3 hours to fix it before the release"

MY APPROACH (90 seconds):
"1. ACKNOWLEDGE WHAT I DON'T KNOW (Crucial!)
   - Told my manager: 'I don't know Helm, but I'll learn enough to fix this'
   - Set expectations: 'Give me 30 min to diagnose'
   
2. ISOLATED THE PROBLEM (didn't just guess)
   - kubectl get deployment -o yaml | grep helm
   - Saw recently modified values.yaml
   - Compared working vs broken services
   - Found pattern: broken services had float values in int fields
   
3. TARGETED LEARNING
   - Read: https://helm.sh/docs/chart/values/ (10 minutes, skipped basics)
   - Searched: 'helm values YAML casting'
   - Found: values.yaml doesn't have types like Go does, everything is string
   - Checked: values-override.yaml (the culprit file)
   - Root cause: passing 1000 (int) when needed '1000' (string)
   
4. FIXED IT
   - Updated override: port: '8080' (now string)
   - Did helm template to validate (didn't just reapply)
   - Redeployed
   - Verified pods came up

5. PREVENTED RECURRENCE
   - Documentation: Added comment in values.yaml
   - Linting: Added yamllint rule to catch types
   - Team training: Taught team about Helm's string handling

TIME: 45 minutes total (had 3 hours)"

KEY LEARNING (30 seconds):
"I learned:
- YAML doesn't enforce types - I need to understand my tools deeply
- When under pressure, isolate before panicking
- Learning targeted documentation is faster than tutorials
- Testing changes (helm template to dry-run) prevents 'quick fixes'

3 months later, I led Helm training for the team. What started as 
a crisis became expertise I could share."

KEY PHRASES:
- "I didn't know X, so I..."
- "I isolated the problem before jumping to solution"
- "I targeted my learning: read docs specific to this issue"
- "I validated before deploying"
- "I documented for the team afterward"
```

### What NOT to Say

❌ "I had to Google every command"
- Too reactive, shows no problem-solving

❌ "Someone else told me it was a type issue"
- Doesn't show independent learning

❌ "I just deleted and redeployed from scratch"
- Brute force, not learning

❌ "I still don't understand Helm"
- Didn't commit to learning beyond the crisis

---

## Question 4: Scope Creep & Saying No

### The Question
"Tell me about a time you had to say 'no' to a request from management or a customer. Why did you say no? How did you handle it?"

### What Interviewer is Evaluating

✓ Can you prioritize effectively?
✓ Do you understand business impact?
✓ Can you say no respectfully?
✓ Do you offer alternatives?
✓ Are you a "can-do" team member without being a pushover?

### Strong Answer Framework

```
SITUATION (30 seconds):
"Our SVP of Product asked me to implement major changes to our 
observability stack 3 weeks before a critical compliance audit. 
We were already running the audit preparation at full capacity."

THE REQUEST (30 seconds):
"They wanted to:
- Migrate from Datadog to Prometheus
- Set up federated Prometheus across 3 regions
- Implement new alerting rules
- All within 2 weeks to 'be ready'

It sounded impressive, but timing was terrible."

WHY I SAID NO (60 seconds):
"I didn't just say 'no,' I said 'not now' with reasoning:

1. TIMING CONFLICT
   - Our audit prep was scheduled: 160 hours engineering
   - This asked for: 120 hours (realistic estimate)
   - We only had 300 total hours available that sprint
   - Math: already at 53% capacity
   
2. RISK ANALYSIS
   - Changing observability stack 3 weeks before audit = risk
   - Audit would check our monitoring for compliance
   - New system = untested, likely missing features we need
   - If failed mid-audit: catastrophic customer impact
   
3. DEPENDENCIES
   - Datadog is instrumented in 200 services
   - Migration isn't just infrastructure, affects applications
   - Need coordination and testing

I framed it as 'I want this done RIGHT, but now isn't when':
'This is a great project, but doing it under audit pressure 
sets us up for failure. What if we do it the week AFTER audit?'"

MY ALTERNATIVE (45 seconds):
"I proposed:
1. Keep Datadog for audit (low risk)
2. Meanwhile, do Prometheus planning (documentation, PoC)
3. After audit (5 days): dedicate 2 weeks to clean migration
4. Outcome: better executed migration vs rushed mess

I also explained the business impact:
- Audit failure = compliance violations = $1M+ fines
- Prometheus done right = better long-term savings
- Prometheus done rushed = outages, customer impact

OUTCOME:
SVP saw the logic. We did Prometheus planning while in audit mode.
After audit (which we passed), we did Prometheus migration cleanly 
in 2 weeks. Everyone was happy, and we had time to do it right."

KEY LEARNING (30 seconds):
"I learned saying 'no' is about:
- Timing, not dismissal
- Offering alternatives
- Understanding business impact
- Being confident in analysis
- Framing as 'preventing failure' not 'blocking innovation'"
```

### What NOT to Say

❌ "My manager was unrealistic"
- Blaming, not problem-solving

❌ "I just said 'we can't do that'"
- No alternative offered

❌ "I escalated it to my skip-level and got overruled"
- Should have handled it with data

❌ "It blew up later and proved me right"
- Vindictive framing

---

## Question 5: Mentoring or Teaching

### The Question
"Tell me about a time you helped someone else grow their technical skills. How did you approach mentoring?"

### What Interviewer is Evaluating

✓ Can you elevate your team?
✓ Are you patient and clear?
✓ Do you take initiative for growth?
✓ Can you teach without condescension?
✓ Are you an investment in company culture?

### Strong Answer Framework

```
SITUATION (30 seconds):
"I had a junior engineer on my team who was solid in application 
development but had almost no DevOps/infrastructure experience. 
They wanted to level up but said, 'Infrastructure seems magical 
and I'll never understand it.'"

MY APPROACH (90 seconds):
"I designed a structured mentoring plan:

Week 1: Fundamentals (1 hour/day)
- Docker: showed how containers work (hands-on with Dockerfiles)
- Not lectures. I built a container together, they built one
- They deployed their container to local Kubernetes

Week 2-3: Kubernetes Deep Dive (2 hours/day)
- kubectl commands (why they matter, not just syntax)
- Real problems: 'Your pod won't start, help us debug' (guided)
- We worked through actual tickets together

Week 4: Autonomous Tasks
- I gave them a real production issue: 'Pod is using too much memory'
- I watched, offered hints, let them lead the investigation
- They found it was a memory leak in a library

Key Mentor Strategies I Used:
1. NOT telling, showing
   - 'Watch how I use strace to debug this'
   - 'Now you try with this similar problem'
   
2. Meeting them where they are
   - Explained containers using analogy: 'Like VMs, but lighter'
   - They were a developer, so framed through application lens
   
3. Debugging with them, not for them
   - Gave hints: 'What would ps aux show?'
   - Let them run commands
   - Asked 'What does that output mean?'
   
4. Celebrating wins
   - When they fixed the memory leak: 'You just solved a production issue.
     This is what DevOps is.'
   - Reinforced that they CAN understand this domain

5. Honest about unknowns
   - When they asked advanced questions I didn't know: 'Great question.
     Let's research together.'
   - Modeled that it's OK not to know everything

OUTCOME:
After 4 weeks:
- They owned small infrastructure tasks
- Their confidence was completely different
- They later led Kubernetes training for the team
- 8 months later, tech lead asked if they'd consider DevOps focus

COMPANY IMPACT:
- Reduced single points of knowledge
- Junior got career growth
- Team capacity increased"

KEY PHRASES:
- "I recognized they wanted to learn, so I..."
- "I met them where they were"
- "I gave them real problems to solve, with guidance"
- "I didn't just tell them, I let them discover"
- "A few months later, they were mentoring others"
```

### What NOT to Say

❌ "I went too fast and they couldn't keep up"
- Shows poor mentoring skill

❌ "I figured they'd learn on their own"
- Passive, not invested

❌ "I gave them books to read"
- Books alone aren't mentoring

❌ "They still don't get it"
- Blaming them, not examining your approach

---

## Question 6: Failure - What Did You Learn?

### The Question
"Tell me about a time you failed at something. What happened? How did you respond?"

### What Interviewer is Evaluating

✓ Can you be vulnerable?
✓ Do you learn from failure?
✓ Are you honest about mistakes?
✓ Do you take responsibility?
✓ What's your growth mindset like?

### Strong Answer Framework

```
SITUATION (30 seconds):
"I designed a caching layer for a service expecting 100 req/sec but 
not stress-tested it. On peak traffic day, it actually got 500 req/sec, 
and the cache strategy completely failed. Service started returning 
stale data and latency doubled."

WHAT WENT WRONG (45 seconds):
"Root causes (all my fault):
1. I designed without testing at scale
2. Didn't model cache hit rate properly
3. Used simple TTL cache when needs were more complex
4. Deployed without gradual rollout or killswitch
5. Made assumptions: 'It'll probably work'

The honest truth: I was overconfident. I thought I was a better 
engineer than I was. I didn't ask for code review. I didn't design 
conservatively."

MY RESPONSE (60 seconds):
"Immediately:
- Acknowledged the issue to my team (didn't hide it)
- Turned off the cache feature (prevented worse damage)
- Service recovered to 150ms latency (acceptable)

Investigation:
- Calculated actual vs assumed cache performance
- Found hit rate was 40%, not 80% I expected
- TTL strategy wasn't suitable for this workload
- Load test from laptop ≠ production load

Fix:
- Redesigned cache with:
  - LRU eviction (not just TTL)
  - In-memory monitoring of hit rate
  - Gradual rollout (5% → 10% → 50%)
  - Detailed load testing before full deployment
  
Re-deployment:
- Smaller batches this time
- Feature flag to disable immediately if needed
- Rollout took 2 weeks instead of 1 day (slower = safer)"

WHAT I LEARNED (45 seconds):
"1. Always test at realistic scale
   - Laptop testing ≠ production
   - Use production-like data volumes
   
2. Design for failure
   - What breaks if this goes wrong?
   - Add killswitches
   - Gradual rollouts
   
3. Ask for help
   - I should have had a code review
   - Second opinion catches blind spots
   
4. Monitor what you build
   - Real-time metrics for cache hit rate
   - Alerts for performance degradation
   
5. Be humble
   - I'm not as smart as I think
   - Sometimes the simple solution is best

FOLLOW-UP:
I gave a team talk on 'Things I Got Wrong' because others had same blind spots.
I built a caching patterns guide for the team.
Years later, I mentor people on this exact issue."

KEY PHRASES:
- 'Looking back, I should have...'
- 'I was overconfident in...'
- 'The root cause was my assumption that...'
- 'I learned I need to...'
- 'I took responsibility for...'
```

### What NOT to Say

❌ "It wasn't really my fault, the requirements changed"
- Blame-shifting

❌ "It all worked out in the end"
- Misses the learning

❌ "I haven't really failed at anything"
- Unrealistic, shows unwillingness to be vulnerable

❌ "Other people should have caught this"
- Deflecting responsibility

---

## Additional Behavioral Topics to Prepare

### 1. Initiative & Ownership
**Question**: "Give an example of when you went above and beyond."
**Key Points**:
- Saw a problem nobody asked you to fix
- Solved it anyway without being asked
- Measurable impact
- Didn't just complain, you built solution

### 2. Handling Pressure
**Question**: "Tell me about your most stressful situation at work."
**Key Points**:
- How you stay calm
- Clear prioritization
- Communication
- Outcome
- Reflection on process

### 3. Technical Growth
**Question**: "How do you stay current with new technologies?"
**Key Points**:
- Specific examples (e.g., "I learned Rust" not "I read blogs")
- Chose learning based on business need
- Applied the learning
- Shared knowledge with team

### 4. Cross-functional Collaboration
**Question**: "Tell me about working with a difficult person or team."
**Key Points**:
- Specific, not vague
- Your perspective AND their perspective
- How you resolved it
- What you learned about communication

### 5. Handling Feedback
**Question**: "When did someone give you critical feedback? How did you take it?"
**Key Points**:
- Specific feedback (not just 'my manager said I could improve')
- Initial reaction (be honest if you were defensive)
- How you addressed it
- Outcome
- Growth from feedback

---

## General Tips for Behavioral Answers

✅ **BE SPECIFIC**
- Use actual examples with details
- Avoid generic stories that could be anyone
- Include numbers, dates, specific problems

✅ **SHOW OWNERSHIP**
- Use "I" not "we" when describing your actions
- "I discovered", "I implemented", "I learned"
- "We" when shared team success

✅ **INCLUDE DATA**
- "Reduced latency by 40%"
- "Saved $5M per quarter"
- "Resolved in 2 hours vs previous 4 hours"

✅ **DEMONSTRATE GROWTH**
- What was your mistake/learning?
- How did you grow?
- What do you do differently now?

✅ **SHOW BUSINESS THINKING**
- Connect technical to business impact
- "This prevented $X in downtime"
- "This improved team velocity"

✅ **END ON POSITIVE**
- Resolution or lesson learned
- Not ending mid-problem

❌ **AVOID**
- "I don't remember"
- Generic stories not specific to you
- Blaming others
- Showing no learning
- Technical jargon when simple language works
- Stories where you're just the helper (show YOUR contribution)

