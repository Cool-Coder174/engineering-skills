# A Load Test That Finds Real Faults

**Sources:** *Release It! Design and Deploy Production-Ready Software* — Michael T. Nygard
(Pragmatic Bookshelf, 2007). Ch. 7, p. 147–160, and Sec. 7.3, p. 152–155 in particular.
Supporting sections: Sec. 8.2, p. 164, Sec. 8.6, p. 174, and Sec. 9.3, p. 182. Also Sec. 9.4,
p. 185, Sec. 9.7, p. 194–195, and Sec. 14.1, p. 241–243.
*Site Reliability Engineering: How Google Runs Production Systems* — Beyer, Jones, Petoff,
Murphy (O'Reilly, 2016). Ch. 3, p. 59–60, Ch. 4, p. 67–68, and Ch. 5, p. 75–76. Also Ch. 17,
p. 225–229 and p. 242–245.

A load test produces one number for this skill. The number is the throughput at an acceptable
response time, for a stated workload. `01-defining-capacity.md` defines that number.

A polite script produces a different number. The different number passes the review and fails
the launch. This file gives the test design that resembles real traffic, and it names the
traffic that a script omits.

---

## 1. The case that this file exists for

**War story — Trampled by Your Own Customers (Sec. 7.1, p. 147–148).** A retailer replaced
its whole commerce stack. More than three hundred people built it. The content delivery
network changed its metadata at 9 a.m., and the change reached the world in about eight
minutes. At 9:05 a.m. the site held 10,000 active sessions. At 9:10 a.m. it held more than
50,000. At 9:30 a.m. it held 250,000 sessions, and then the site crashed.

The site had already passed a load-test campaign of three months and more than sixty
application builds (Sec. 7.3, p. 155). The campaign raised capacity ten times, to 12,000
active sessions (Sec. 7.3, p. 155). Marketing accepted 12,000 sessions for a launch in the
slow part of the year (Sec. 7.3, p. 155). The test was long, expensive, and honest. The site
still crashed in thirty minutes.

**The test measured the traffic that the team could imagine.** Every script mimicked a real
user with a real browser (Sec. 7.4, p. 155). Every script moved from one page to a linked
page. Every script accepted cookies. Nygard states the result in one line: the real world can
be rude, crude, and vile (Sec. 7.4, p. 156).

**A passed load test is not evidence unless the test contained noise.** Section 6 lists the
noise. `capacity-defect-catalog.md` records the gap as C-05 and C-06.

**The failure was a capacity failure that became a stability failure.** The sessions consumed
RAM. Session replication then consumed CPU and network bandwidth for every page request
(Sec. 7.4, p. 155). Nygard calls sessions the Achilles heel of every application server
(Sec. 7.4, p. 155).

---

## 2. Why a test that targets QA passes

**War story — building for the test environment (Sec. 7.2, p. 149).** Nygard joined the
project. He found that the teams built everything to pass testing, not to run in production.
Across fifteen applications and more than five hundred integration points, every
configuration file named the integration-testing environment. Some components assumed the QA
topology, which nobody expected to match production. Production would hold firewalls that QA
did not hold. QA ran one instance where production ran a cluster. The barrier to change in
the test environment was high. Most of the team ignored the differences. A change would cost
one or two weeks of build-deploy-test cycles.

**Signal that this failure is present:** a team explains a difference between the environments
instead of removing it. At that moment the test result stops transferring to production.

**Nygard ranks the causes of a failed deployment.** The usual cause is a mismatch in topology,
not a mismatch in configuration values (Sec. 14.1, p. 241). Topology is the number and the
connectivity of the servers and the applications (Sec. 14.1, p. 241–242).

| The gap | What the case study had | The remedy | Source |
|---|---|---|---|
| Configuration target | Every file written for the test environment | Hold the overrides separate from the code, with each varying property in exactly one place | Sec. 7.2, p. 149 and p. 151 |
| Host separation | Several applications on one test host | Keep the applications on separate hosts | Sec. 14.1, p. 242 |
| Instance count | One instance against a production cluster | Run more than one instance. The sensible numbers are zero, one, and many | Sec. 14.1, p. 242 |
| Firewalls | Firewalls in production only | Develop with the firewalls present from the first day | Sec. 14.1, p. 243 |
| Network gear | No load balancer in the test path | Buy the same vendor and product line, though not the top model | Sec. 14.1, p. 243 |
| Data volume | Development-sized data | Run against production-sized data, scrubbed. Write a data generator when no set exists | Sec. 9.7, p. 194–195 |

**A gain measured on a development database can disappear in production.** The production
query plan differs (Sec. 9.7, p. 195). `02-capacity-antipatterns.md` covers the query
defects. `capacity-defect-catalog.md` records the topology gap as C-08.

---

## 3. Which test to run

**A load test and a stress test answer different questions.** Run both. Neither one answers
for the other.

| Test | The question it answers | The signal that calls for it | What it cannot prove | Source |
|---|---|---|---|---|
| Load test | What is the throughput at an acceptable response time? | A launch, a peak season, or a change to a request path | Behavior above the limit | Release It! Sec. 7.3, p. 152 |
| Stress test | Where is the limit, and how does the component fail? | Nobody can name the first component to fail | Behavior at normal load | SRE Ch. 17, p. 228 |
| Performance test | Did this release change the resource cost? | Every release | The position of the limit | SRE Ch. 17, p. 225–226 |
| Smoke test | Does the critical path still work? | Before any more expensive test | Any capacity number | SRE Ch. 17, p. 225 |
| Configuration test | Is production configured as the file states? | After a rollout | Any capacity number | SRE Ch. 17, p. 227–228 |
| Canary | Does the new build change the variance under real traffic? | The load test passed and the release is next | The size of the fault before release | SRE Ch. 17, p. 228–229 |
| Production probe | Do the known-good and known-bad requests still behave? | Continuously, after release | Any capacity number | SRE Ch. 17, p. 242–245 |
| Soak test | Does a resource leak over hours? | A long-lived process holds a cache, a pool, or a session store | Peak behavior | **Modern** |

**SRE gives the reason for the stress test.** Individual components do not degrade gracefully
past a point. They fail catastrophically instead (Ch. 17, p. 228). The stress test asks how full
a database can get before writes fail. It also asks how many queries per second an
application server accepts before requests fail (Ch. 17, p. 228).

**A canary test is not really a test.** SRE calls it structured user acceptance (Ch. 17,
p. 229). It exposes the code to live traffic and it does not always find a new fault (Ch. 17,
p. 229). `reverse-branching` owns the canary progression and the revert.

**A production probe is a set of requests, not a load.** SRE splits the bank into three sets
(Ch. 17, p. 243). The first set holds known-bad requests that must error. The second holds
known-good requests that the team can replay against production. The third holds known-good
requests that it cannot replay. Those probes should never fail (Ch. 17, p. 243).

---

## 4. The workload model

**Traffic analysis gives you variables. Experience assigns importance to them.** Nygard names
the variables: browsing patterns, pages per session, conversion rates, think-time
distributions, connection speeds, and catalog access patterns (Sec. 7.3, p. 152). His team
expected think time, conversion rate, session duration, and catalog access to be the largest
drivers (Sec. 7.3, p. 152).

| Element | The rule | The book's number | Source |
|---|---|---|---|
| Session mix | Model grazers, searchers, and buyers as separate scripts | More than 90 percent view the home page and one product detail page | Sec. 7.3, p. 152 |
| Conversion share | Send a small share of the virtual users through checkout | 4 percent | Sec. 7.3, p. 152 |
| Page depth | Give each script the page count of its own session type | Twelve pages for a checkout, seven or fewer for a scan | Sec. 7.3, p. 152 |
| Think time | Use the measured delay between clicks. Never use zero | Five to ten seconds between page clicks | Sec. 7.3, p. 152 and Sec. 9.3, p. 182 |
| Asynchronous think time | Model the shorter delay that a background request creates | One to three seconds between requests | Sec. 9.3, p. 182 |
| Session duration | Model it, because it sets the live session count | Timeout of about 10 minutes for retail, 5 for a media gateway, up to 20 for travel | Sec. 9.4, p. 185 |
| Catalog access | Spread the requests across the catalog as the analysis shows | A local decision | Sec. 7.3, p. 152 |
| Connection speed | Include the slow client, because it holds server memory longer | A local decision | Sec. 7.3, p. 152 |
| Harshness | Make the mix somewhat harsher than the real traffic | Nygard's team believed its mix was slightly harsher | Sec. 7.3, p. 152 |

**Checkout is the most expensive session, so it needs its own script.** It calls external
integrations for card validation, address normalization, inventory, and availability
(Sec. 7.3, p. 152). A mix that omits checkout understates the cost of every integration
point.

**Set the session timeout before you model the session count.** The rule is one standard
deviation past the average think time (Sec. 9.4, p. 185). The thirty-minute default is
overkill (Sec. 9.4, p. 185). `02-capacity-antipatterns.md` gives the session rules.

---

## 5. Count sessions, not concurrent users

**There is no such thing as a concurrent user (Sec. 7.3, p. 153).** Load-test vendors say
"concurrent users" when they mean bots. Business sponsors say it when they mean sessions.
Unless users connect straight to the database, the concurrent user is a fiction.

**Counting concurrent users judges capacity badly.** Capacity depends on what the users do
(Sec. 7.3, p. 153). A population that only views the front page yields a much higher capacity
than a population that buys.

**A session lasts longer than its user.** The server receives discrete requests and ties them
together with an identifier (Sec. 7.3, p. 153). The server cannot separate a user who will
never click again from a user who has not clicked yet, so it applies a timeout (Sec. 7.3,
p. 153). Counting sessions therefore overestimates the number of users (Sec. 7.3, p. 153).
The book's diagram shows five sessions for two users (Sec. 7.3, p. 154).

**The number of active sessions is one of the most important measurements about a web
system** (Sec. 7.3, p. 154). Report it for every step of the test. In the case study the
session count led the team almost straight to the fault (Sec. 7.4, p. 155).

---

## 6. Noise: the traffic that nobody scripts

**War story — Murder by the Masses (Sec. 7.4, p. 155–157).** Search engines drove about
40 percent of the visits. On the switch day they drove customers to old URLs. The web
servers routed every `.html` request to the application servers, so that the site could track
sessions. Each dead link therefore created a session only to serve a 404 page. Spiders that
keep no cookies created a new session on every page request, and each session stayed resident
for thirty minutes. One search engine created up to ten sessions per second. The team also
found nearly a dozen high-volume page scrapers. One of them sent requests from many small
subnets, and it changed the `User-Agent` string between consecutive requests from one
address. Amateur shopbots requested one product detail page once per minute, three years
after that console was scarce.

| Noise class | What the client does | What it costs | Source |
|---|---|---|---|
| Dead-link referral | Requests an old URL that returns 404 | One session per 404, because the application server renders the page | Sec. 7.4, p. 156 |
| Cookieless spider | Requests pages and returns no cookie | One session per page request, resident until the timeout | Sec. 7.4, p. 156 |
| Page scraper | Rotates subnets and `User-Agent` strings to hide its origin | Sessions at volume, and no cookie handling | Sec. 7.4, p. 156 |
| Shopbot | Requests one URL once per minute from several addresses | One session per request, forever | Sec. 7.4, p. 156 |
| Repeated URL flood | Requests the same URL again and again after the user leaves | Sessions, plus the full cost of the page | Sec. 7.4, p. 157 |
| Asynchronous request with no session id | Sends a background request that carries no identifier | A new wasted session for every request | Sec. 9.3, p. 184 |

**Noise can stop the site, not only reduce its capacity.** Nygard states both outcomes
(Sec. 7.5, p. 157). Add one script for each row of the table before you accept a number.

**Serve the noise without the application server.** The team made a static copy of the home
page for unidentified customers. It also moved 404 handling away from the application
(Sec. 7.6, p. 158–159). The dynamic home page was served five million times a day, was
identical every time, and needed more than 1,000 database transactions to build (Sec. 7.6,
p. 158). `03-capacity-patterns.md` covers the precompute pattern.

**A scraper is a capacity cost here, and a threat elsewhere.** This file measures what it
consumes. `security-engineering` owns the identified abusive client.

---

## 7. Generate the load independently of the response time

**Send the next request on a schedule, not after the previous response — Modern.** Neither
book states this rule. A tool configured with a fixed number of virtual users and no arrival
rate stops sending load exactly when the system slows. The queue never grows, and the report
flatters the system.

**Signal that the test has this defect:** the tool configuration names virtual users and names
no requests per second. The measured throughput tracks the response time in a straight line.

**Fix:** drive the test at a fixed arrival rate. Then raise the rate in steps and record the
response-time distribution at each step. `capacity-defect-catalog.md` records this as C-07.
`data-systems-design` owns the statement of this rule as a requirements principle. This file
owns the test configuration that satisfies it.

---

## 8. Test the failure mode, not the happy path

**War story — the testing gap (Sec. 7.5, p. 157).** Two things were missing. First, the team
tested the application the way it was meant to be used. A script requested a URL, waited for
the response, and then requested another URL that appeared on that response page. No script
requested the same URL 100 times per second without cookies. Nygard adds that the team would
probably have called such a script unrealistic and ignored the crash it produced. Second, the
developers built no safety devices that stop a bad condition. The application continued to
send threads into the danger zone. New request threads collected behind the threads that were
already broken or hung.

**Nygard states the rule directly: do not follow the happy path only** (Sec. 7.5,
p. 157). A load tester who only obeys the rules is like an application tester who only clicks
buttons in the right order.

**Run the test past the target. Signal: the highest step of the plan equals the target.**
Follow these five steps.

1. Raise the arrival rate in steps until one component fails.
2. Record the throughput at the knee, where the correlation with the driving variable stops.
3. Record the first component that failed, and the error that it returned.
4. Measure the recovery time from full saturation, with the load removed.
5. Set the operating limit below the knee, and record the margin.

**Recovery time is a separate number from the limit.** The overloaded site needed nearly an
hour to serve pages again (Sec. 7.6, p. 158). That number is why the team throttled the site
before saturation rather than after it.

**A missing safety limit turns the test into a crash.** Nygard's general rule has two parts.
Place safety limits on everything, and protect the request-handling threads (Sec. 8.6,
p. 174).
`../SKILL.md`, Section 7 carries the limits and the book's numbers. `self-healing-apis` owns
Circuit Breaker and the other stability patterns that act at the limit.

---

## 9. The loop that makes a load test useful

**War story — the load generators lied twice (Sec. 7.3, p. 154).** One evening, the engineer
from the load-test vendor watched the Windows machines in the load farm. They were
downloading and installing software. Somebody was hacking the farm while the team used it
to generate load. On another occasion the results showed a bandwidth ceiling. An AT&T
engineer had noticed that one subnet used too much bandwidth, and had capped the link that
generated 80 percent of the load. Neither ceiling belonged to the system under test.

**Check the generators before you believe a ceiling.** A ceiling that does not move when you
add application servers is a candidate generator fault.

**The value of a load test comes from the cycle time, not from the report.** An unattended
test leaves about three or four days between runs (Sec. 7.3, p. 152). Nygard's team put the
test manager, a vendor engineer, an architect, a DBA, and Nygard himself on one call, twelve
hours a day (Sec. 7.3, p. 152 and p. 154). The team then made a configuration change within a
single day, and a code change within two or three days. It made a few architecture changes
within a week (Sec. 7.3, p. 154–155). The first run reached only 1,200 concurrent users before
the site stopped, and every application server needed a restart (Sec. 7.3, p. 154). The team
needed a twenty-fold gain (Sec. 7.3, p. 154).

**Run the test on every release. Signal: any release at all.** Nygard requires continuous
capacity monitoring, because each release can change scalability and performance, and because
demand changes the workload (Sec. 8.6, p. 174). SRE assigns the same job to the performance
test (Ch. 17, p. 225–226). That test catches a program whose memory grows from 8 GB to 32 GB.
It also catches a response that grows from 10 ms to 50 ms and then to 100 ms.

**Automate the run.** A test that a person repeats by hand every release is toil (Ch. 5,
p. 75–76). SRE defines toil as manual, repetitive, automatable, tactical, without enduring
value, and growing with the service (Ch. 5, p. 75–76).

**A test result can gate a release.** SRE halts releases when the error budget is spent, and
invests the time in testing instead (Ch. 3, p. 60). Use the same rule for a measured capacity
regression.

---

## 10. Read the result correctly

**Report percentiles, not a mean.** SRE states that the 99th and 99.9th percentiles show a
plausible worst case and the 50th shows the typical case (Ch. 4, p. 68). Do not assume that
the mean equals the median, and do not assume a normal distribution without a check (Ch. 4,
p. 68–69). `capacity-defect-catalog.md` records the mean as C-04.

**The knee is the result.** The knee is the point where the correlation between the two
variables stops (Sec. 8.2, p. 164). Demand keeps rising there while service falls (Sec. 8.2,
p. 164). A rapid falloff at the knee means a constraint is already reached (Sec. 8.2, p. 164).
`01-defining-capacity.md` gives the procedure that names the constraint.

**Report these five numbers for every run.**

1. The workload mix, including the noise scripts.
2. The arrival rate at each step, and the response-time percentiles at each step.
3. The throughput at the knee, and the component that reached its limit there.
4. The live session count at the knee.
5. The recovery time from saturation.

**A number with no workload and no response-time bound is not a capacity number.** Both
variables are required (Sec. 8.1, p. 162). `capacity-defect-catalog.md` records this as C-03.

---

## 11. The mitigations cost money, so price them

**War story — the aftermath (Sec. 7.6, p. 158–160).** The content delivery network added a
gateway page in one day. It redirected a browser that handled no cookies. It throttled the
share of new sessions that reached the real home page. It also blocked named IP addresses. An
engineer watched the session counts at all times for three weeks, and by the third week the
throttle stayed at 100 percent all day. The sessions held whole shopping carts and up to
2,000 search results, so the team disabled session failover. A customer in checkout on a lost
instance then returned to the cart page. Nygard names the theme: nothing is as permanent as a
temporary fix, and most of these fixes stayed for the next year or two.

**Every emergency mitigation needs an expiry date and an owner.** The case study priced the
cost (Sec. 7.6, p. 160). It was lost orders, broken checkout, abandoned personalization,
doubled hardware, and a year of remediation instead of revenue features.
`capacity-defect-catalog.md` records this as C-40. `reverse-branching` owns the removal.

**The losses were avoidable.** Two years later the same site carried more than four times the
load on fewer servers, from software improvement alone (Sec. 7.6, p. 160).

---

## 12. The defect codes that this file supports

| Code | Short name | Where this file covers it |
|---|---|---|
| C-05 | The load script follows the happy path only | Sections 1 and 8 |
| C-06 | The test contains no noise traffic | Section 6 |
| C-07 | The load generator waits for the response — **Modern** | Section 7 |
| C-08 | The test environment does not match the production topology | Section 2 |
| C-42 | The load generators themselves are never verified | Section 9 |
| C-09 | No test past the limit | Section 8 |
| C-03 | A capacity number with no workload and no response-time bound | Section 10 |
| C-04 | Capacity read from a mean | Section 10 |
| C-17 | Every request creates a session | Sections 5 and 6 |
| C-40 | A temporary mitigation with no expiry | Section 11 |

Read `capacity-defect-catalog.md` for the signature, the consequence, and the fix of each
code.

---

## 13. What this file does not own

| Question | Owner |
|---|---|
| Which resource is the constraint, and what is the knee | `01-defining-capacity.md` |
| Why the per-request cost is high | `02-capacity-antipatterns.md` |
| How to remove the cost that the test found | `03-capacity-patterns.md` |
| How the measured number becomes a plan | `05-intent-based-capacity-planning.md` |
| How traffic is distributed, and which utilization signal to read | `06-load-balancing-and-utilization.md` |
| What the system does when it is over the limit | `self-healing-apis` |
| Canary progression, revert, and the release that caused a regression | `reverse-branching` |
| Whether the design stays correct under concurrency | `data-systems-design` |

**Proportionality.** A change with no new call, no new query, and no new per-request work
needs no load test. State the reason and stop. `../SKILL.md`, Section 12 gives the rule.
