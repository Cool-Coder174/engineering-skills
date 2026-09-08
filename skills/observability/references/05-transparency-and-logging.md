# Transparency: Designing a System That Can Be Seen

**Sources:** *Release It! Design and Deploy Production-Ready Software* — Michael T. Nygard (Pragmatic Bookshelf,
2007). Ch. 17, Transparency, Sec. 17.1 through Sec. 17.6, p. 265–298. Fig. 17.10, p. 299, rates each technology
against each perspective. The operations database and the review cadences are in `06-the-operations-database.md`.

Nygard defines transparency as the qualities that let operators, developers, and business sponsors understand four
things about a system (Sec. 17, p. 265). Those four things are the historical trends, the present conditions, the
instantaneous state, and the future projections.

**A system without transparency cannot survive long in production** (Sec. 17, p. 266). An administrator cannot tune
a system that hides its work. A developer cannot make it more reliable. A sponsor cannot see whether the system
earns money, so the sponsor stops the funding. The system then drifts into decay and works a little worse after each
release.

**A transparent system matures faster than an opaque one**, because debugging it is much easier (p. 266). It also
trains the people who attend it, in the way a ship engine trains its engineers.

---

## 1. The four perspectives

Different people need different perspectives, and one view does not serve them all (Sec. 17.1, p. 267).

| Perspective | The question it answers | Best technology | Do not use it for |
|---|---|---|---|
| **Historical trending** | What did the system do over past periods? | A database, which Nygard names the OpsDB (p. 267) | The dashboard. It lacks the immediacy of the present (p. 267) |
| **Predictive forecasting** | What will the system do next? | A correlative model built on past data (p. 269) | The dashboard. A projection is sensitive and is not urgent (p. 269) |
| **Present status** | What has the system done? | The dashboard, fed by the OpsDB (p. 272, p. 300) | Deep diagnosis of one event |
| **Instantaneous behavior** | What is the system doing right now? | The monitoring system, thread dumps, stack traces, errors in log files, JMX (p. 274) | A wide audience. Restrict the dangerous channels (p. 274) |

**The fit of each technology** (Fig. 17.10, p. 299). The figure rates each technology on a
four-level scale. The levels are unsuitable, poor fit, workable with effort, and well suited.
Logging and monitoring are both good at immediate behavior. Present status is derivable from log
files alone, and the work is hard. A monitoring system is great at instantaneous behavior, and it
does a good job of present status. Nygard states that monitoring systems have a way to go on the
historical view and the future view. He calls that a gap (p. 299).

**Historical trending holds business metrics and system metrics together** (Sec. 17.1, p. 267). Business metrics
include customers, orders, conversion rate, and revenue. System metrics include free storage, average CPU
utilization, network bandwidth, and errors logged. **Signal that a person needs this view.** The person asks how
many orders arrived yesterday. The person asks which system was the limiting factor at the last
traffic spike. "This day last year" means the same day of the week, not the same day of the month
(p. 268, fn. 2). Do not point a reporting tool at the production transactional database (p. 268).

**A prediction is always built on a model. A bad model gives a bad prediction** (Sec. 17.1, p. 269). Nygard names
the linear projection. A model that is good enough comes from correlations in past data, and it is valid inside a
certain domain of applicability. He cites "yesterday's weather" from Beck and Fowler, which is accurate about 70% of
the time (p. 268). **A release can alter or invalidate the correlations that carry a projection.** Reexamine them
after each release. Wait for an adequate body of new measurements. Attach to every prediction a
reference that names the projections you used. **A question that needs a projection about a projection squares the error. It does
not double it** (p. 269).

**Instantaneous behavior answers the question "What is going on?"** (Sec. 17.1, p. 273). People want it most when an
incident already runs. An anomaly here often, but not always, produces an incorrect status. Nygard gives the chain.
Users receive errors. Users leave. The transactions per hour metric then falls below its nominal range. Catch the
aberrant behavior before the users walk away. **Restrict the dangerous channels.** Not everyone gets access to the
JMX console, because a person there can stop servers or change vital parameters. A thread dump needs privileged
access (p. 274).

---

## 2. Present status, and the dashboard

Present status is not about what the system does now. It is about what the system has done (Sec. 17.1, p. 270). It
covers each piece of hardware, each application server, each application, and each batch job. **The status of a
component is a combination of events and parameters.**

| Term | Definition | The rule |
|---|---|---|
| **Event** | A point-in-time occurrence | Some events are normal or required. A daily inventory feed is required, so its absence must cause an alarm (p. 270) |
| **Abnormal event** | An occurrence of concern | Categorize it low-medium-high, or warn-severe-critical (p. 270) |
| **Parameter** | A continuous metric, or a discrete state | A parameter is nominal while its metric sits inside the acceptable range (p. 271) |
| **Caution range** | A second, tighter range | It warns that the parameter approaches its threshold (p. 271) |

**Transparency matters most at the parameter** (p. 270). An application that reveals more of its internal state
gives parameters that are more accurate and more actionable.

**The nominal rule for a continuous metric.** Nominal is the mean value for the time period plus or minus two
standard deviations (Sec. 17.1, p. 271). For most metrics that traffic drives, the period with the most stable
correlation is the hour of the week. The day of the month means little. In the travel, floral, and sports industries
the most relevant measurement counts backward from a holiday. Nygard states that there is no one right answer for
all organizations (p. 272).

| Color | Condition (Fig. 17.1, p. 273) |
|---|---|
| **Green** | **All** of these are true. All expected events have occurred. No abnormal events have occurred. All metrics are nominal. All states are fully operational |
| **Yellow** | **At least one** of these is true. An expected event has not occurred. A medium-severity abnormal event has occurred. A parameter is above or below nominal. A noncritical state is not fully operational, such as a Circuit Breaker that cut a noncritical feature |
| **Red** | **At least one** of these is true. A **required** event has not occurred. A high-severity abnormal event has occurred. A parameter is far above or far below nominal. A critical state is not at its expected value, such as "accepting requests" being false when it must be true |

**This definition covers a trouble condition that people often overlook, which is too much of a good thing**
(p. 272). Show the dashboard broadly. A projection on a lunchroom wall is not out of the question (p. 272).

**One dashboard, several facets.** An engineer in operations cares first about the component view. A developer wants
the application view. A sponsor wants the view combined to the feature level. The dashboard must know the linkages
between these views (p. 272). An administrator who sees a component outage must then see which business processes it
affects, which helps correct prioritization. **The dashboard must represent a required expected event, whether or
not it occurred, and whether or not it succeeded.** Daily feeds, extracts, and batch jobs are as much a part of the
system as the web servers.

---

## 3. Design for transparency from the start

**Transparency comes from deliberate design and architecture** (Sec. 17.2, p. 275). Nygard states that "adding
transparency" late in development is about as effective as "adding quality". You can perhaps do it, but only at
greater effort and cost than if you had built it in from the beginning.

**Visibility inside one application or one server is not enough. Strictly local visibility leads to strictly local
optimization** (p. 275). Visibility into one application at a time also masks a scaling effect. Section 10 gives
both stories.

**Keep the policy outside the application.** Three decisions belong outside the application
(p. 275). They are which metrics trigger alerts, where to set the thresholds, and how to combine
state variables into one health status. These are policy decisions. They change at a very different
rate than the application code does.

**Watch the coupling.** A monitoring framework intrudes on the internals of a system easily. The monitoring and
reporting systems must be like an exoskeleton built around your system, not woven into it (p. 275). Design to a
small set of standards to avoid that coupling. This is catalog code M-34.

---

## 4. Enabling technologies

**A process on a server is opaque by nature** (Sec. 17.3, p. 276). Unless you run a debugger on it, the process
reveals almost nothing about itself. The first task is to get information out of it.

| | White-box technology | Black-box technology |
|---|---|---|
| Where it runs | Inside the observed process or system (p. 276) | Outside the process, on externally observable things (p. 276) |
| Who adds it, and when | Developers, during development (p. 276) | Operations, usually after delivery (p. 276) |
| Coupling | Tighter (p. 276) | Loose (p. 276) |
| The example Nygard names | Logging (Sec. 17.4, p. 276) | The monitoring system (Sec. 17.5, p. 283) |

**Even for a black-box tool, developers can do helpful work during development** (p. 276). One uniform error format
is that work (p. 284–285). For the SRE version, read `03-white-box-and-black-box.md`.

---

## 5. The log contract

**A log file is the most reliable and most versatile information vehicle that Nygard names** (Sec. 17.4, p. 276).
Nothing is more loosely coupled, because every framework and tool that exists can scrape a log file (p. 277). A log
file reflects activity inside an application, so it reveals instantaneous behavior. It is also persistent, so you
can examine it to understand status. Status needs digestion, because you must trace state transitions into current
states (p. 276).

### 5.1 Configuration

**Make the location of the log file configurable** (Sec. 17.4, p. 277). An administrator often
wants logs on a different drive than the operating system or the content. Log files are large, they
grow rapidly, and they consume a lot of I/O. If the location is not configurable, the administrator
relocates the files anyway, in a way you might not like. Nygard names the configurable path as
reason number 72 not to write your own logging framework.

### 5.2 Levels

**Aim logging at production operations, not at development or at testing** (Sec. 17.4, p. 278). Administrators and
engineers in operations spend far more time with these files than developers do (p. 277). **Anything logged at ERROR
or SEVERE must require action by operations.** That rule governs the table.

| The event | Level | Reason |
|---|---|---|
| A Circuit Breaker trips to open | ERROR | It must not happen normally, and action is probably required at the other end of the connection (p. 278) |
| The application fails to connect to a database | ERROR | There is a problem with the network or with the database server (p. 278) |
| A user typed a bad credit card number | A warning, if you log it at all | Business logic and user input are not system problems (p. 278) |
| A `NullPointerException` | Decide by the effect | An exception is not automatically an error (p. 278) |
| An interesting state transition | Always log it | The record matters during a postmortem (p. 283) |
| A debug or trace message in production | Never ship it | See the build step below (p. 278) |

**Add a build step that automatically removes any configuration that enables a debug or a trace level** (p. 278). A
debug level in production buries real issues in method traces and trivial checkpoints. One commit while the logging
configuration holds a debug level is all it takes. This is code M-27. **A log file trains its readers.** As people
read or scan the log of a new system, they learn what normal means for it (p. 277). A noisy application teaches its
operators that errors are normal.

### 5.3 The catalog of messages

Operations commonly asks for a list of every log message the system can produce (Sec. 17.4, p. 278). Nygard gives a
method that produces the list.

1. Run the "Externalize Strings" command of the IDE over the code that holds the logging calls.
2. Read the resource bundle. Each entry has a key, such as `FulfillmentClient.2`, and message text.
3. Change the accessor so that it prefixes the key to the text as a message code (p. 279).
4. Send the resource bundle to operations as the catalog of messages (p. 278).
5. Record each code in a run book or a knowledge base (p. 279).

**A message code buys two things** (p. 278–279). It makes communication between operations and development accurate.
Nygard contrasts it with the report "it said something about a fatal system error". It also gives operations a short
string to find in a run book.

### 5.4 A log message is a user interface

**A log file is a human-computer interface, so judge it by human factors** (Sec. 17.4, p. 280). Nygard gives the
reason directly. In a Severity 1 incident, a human who misreads status information can prolong the problem or make
it worse.

| Format property | Write it this way | Nygard's judgement |
|---|---|---|
| Record shape | One line for each record | The JDK two-line default format makes scanning utterly impossible (p. 281) |
| Column layout | Space-padded columns | The columnar format helps a human read the file (p. 281) |
| Severity | An indicator of one character, such as I, W, and A | Once you know the letters, scanning for a warning becomes trivial (p. 281) |
| Message code | A field of its own | It aids automated parsing of the file (p. 281) |

**Nygard states that the two-line format defeats man and machine** (p. 281). `grep` has no idea how to handle a
two-line record. **Modern.** A structured record, such as one JSON object for each line, serves only the machine
half of that contract. Keep the severity field and the message code, and render the record for a person. **Write the
message for the person who reads it during an incident.** A log file must carry information that is clear, accurate,
and actionable (p. 280). Name the actor and the action. Section 10 shows what an unnamed actor costs. These rules
are catalog codes M-29 and M-30.

### 5.5 The two final rules

**A message must include an identifier that traces the steps of one transaction** (Sec. 17.4, p. 283). Use a user
id, a session id, a transaction id, or an arbitrary number that you assign when the request arrives. When you must
read 10,000 lines of a log file after an outage, a string to search for saves a lot of time.

**Log every interesting state transition, even when you plan to send an SNMP trap or a JMX notification** (p. 283).
It costs a few seconds of coding, it leaves the options open downstream, and the record of state transitions matters
during a postmortem.

**Modern.** Propagating one trace identifier across every outbound call is later practice. Nygard also states no
redaction rule for a log line. Name every field that you log, and never format a whole object into a message. The
neighboring rule that he does state is for memory. A core file is a memory dump and holds the passwords, so disable
core dumps on a production application (Sec. 12.2, p. 228). These rules are codes M-31, M-32, and M-33.

---

## 6. Monitoring systems

**Logging helps only while the application runs. A dead process logs no tales, and neither does a hung process**
(Sec. 17.5, p. 283). Some entity outside the process must watch. A monitoring system has three essential components.
They are agents that collect information, a reliable transport mechanism, and the display of that information. An
agent observes operating system statistics, process health, patterns inside a log file, and port listeners. It scans
syslog on UNIX, and the event logs on Windows (p. 283–284).

| The rule | Why it holds |
|---|---|
| **An agent detects only the events that a person told it to look for** | Unanticipated or novel behavior might not be detected (p. 284) |
| **Report every error in a log file with the same format** | Then an unexpected error is still caught. An error in a different format, or one that nobody logs, cannot be caught (p. 284–285) |
| **A monitoring system always needs a heartbeat** | The agent dies with its host. The heartbeat detects a failed agent, or a network failure between the agent and the central system (p. 285) |
| **Keep monitoring traffic off the segments that carry production traffic** | A worm, a denial-of-service attack, or a configuration error then disables monitoring automatically (p. 285) |
| **Keep monitoring traffic off a VLAN that carries public traffic** | That traffic holds hostnames, user names, internal IP addresses, snippets of log files, process names, and process ids (p. 285) |

**Agentless monitoring is essentially a marketing ploy** (p. 285). The observations still need
collaboration by the operating system of the observed host. The gathering therefore consumes
resources there.

**Gap 1. A commercial monitoring system is oriented toward IT, not toward business results** (p. 286). You often
cannot say which server provides one feature, because groups of servers across several tiers serve it. The
monitoring system must know the business features, and it must identify the impact on those features at every system
event.

**Gap 2. A monitoring system reports the system's view of itself, not the view of one user**
(p. 287). Every component can run correctly by itself and still give the end user a bad result.
Nygard says this often happens with blocked threads and with cascading failures. The remedy is the
outside probe, in `03-white-box-and-black-box.md`. This is catalog code M-26.

**The selection of the monitoring system will be done for you** (p. 289). Nygard states that the decision outlasts
corporate commitments to operating systems, programming languages, hardware vendors, and org charts. **Therefore the
monitoring system is part of the environment that you design for.** Design to a small set of standards, official and
de facto, and avoid vendor lock-in.

---

## 7. Standards, de jure and de facto

| Standard | Year and origin | The essential concept | The constraint |
|---|---|---|---|
| **SNMP** | 1988, version 1 (p. 289) | "Everything is a variable." There are no commands, only variable assignments (p. 289) | Only three operations exist. Get a variable, set a variable, enumerate variables (p. 298) |
| **CIM** | 1996, from the DMTF (p. 292) | A metaobject protocol and a brokered structure that allow dynamic registration and discovery (p. 292–293) | Technically superior, far less widely implemented. Nygard calls it a future concern (p. 293) |
| **JMX** | 1998, as JSR 3, and part of the JVM since Java 5 (p. 292, p. 296) | An object-oriented view. An MBean is a management proxy registered with an MBeanServer (p. 293) | Little used by application developers, but broadly supported by platform developers (p. 296) |

**SNMP details, and the MIB warning** (Sec. 17.6, p. 289–292). A vendor writes a Management Information Base in
ASN.1, and the monitoring system compiles it. A trap is an asynchronous notification from an agent to a manager.
**Do not write your own MIB without cause.** Nygard states that a MIB for custom software is a
huge undertaking of a very specialized nature. A monitoring administrator is also often reluctant
to install a MIB from an untrusted party (p. 291). For a Java system, write to JMX and use a
JMX-to-SNMP connector (p. 298).

**Attach an MBean to a long-lived component** (sidebar, p. 294). Nygard names the components that serve well.
Resource pools, caches, repositories, and interfaces to external systems.

**Make administration interfaces scriptable** (p. 296). Nygard states that he cannot overstate the value of one. A
useful set of MBeans through JMX is the easiest way to offer scriptable administration for a Java application
(p. 297).

---

## 8. What to expose

**Expose every state variable, every counter, and every metric** (Sec. 17.6, p. 297). Nygard gives two reasons why a
shorter list fails. First, you are likely to guess the key metric wrong. Second, even a correct guess decays,
because code changes and demand patterns change. **Provide universal visibility now, and externalize the policy so
you can defer those decisions** (p. 297).

| Family | The values that carry the diagnosis |
|---|---|
| **Traffic indicators** | Page requests total, page requests, transaction counts, concurrent sessions (p. 297) |
| **Resource pool health** | Enabled state, total resources, resources checked out, high-water mark, resources created, resources destroyed, times checked out, **threads blocked waiting for a resource**, times a thread has blocked waiting (p. 297) |
| **Database connection health** | Number of SQLExceptions thrown, number of queries, average response time to queries (p. 297) |
| **Integration point health** | State of the Circuit Breaker, number of timeouts, number of requests, average response time, number of good responses, **number of network errors, number of protocol errors, number of application errors**, actual IP address of the remote endpoint, current concurrent requests, concurrent request high-water mark (p. 298) |
| **Cache health** | Items in cache, memory used, hit rate, items flushed by the garbage collector, configured upper limit, time spent creating items (p. 298) |

Present status adds thread pools, request channels, business transactions, users, memory, and
garbage collection (Sec. 17.1, p. 270–271). `../SKILL.md`, Table 8, carries the full per-component
list. **Every counter carries an implied time component.** Read each one as if it ends with "in the
last n minutes" or "since the last reset" (p. 298). A medium-sized application can have hundreds of
parameters (p. 271). The system still has to do something other than collect data (p. 297).

---

## 9. The operator question test

**Apply this test to any change that adds a log message, a dashboard panel, or an admin view.** Name a question that
an operator must answer at 03:00. Then check that the operator can answer it without a developer.

| The operator needs | The basis in the book |
|---|---|
| Logging aimed at operations | Administrators and engineers in operations spend far more time with the log files than developers do (Sec. 17.4, p. 277) |
| A list of every message the system can produce | Operations commonly asks for that list, and the resource bundle supplies it (Sec. 17.4, p. 278) |
| A code to find in a run book | A message code is easy for operations to find in a run book or a knowledge base (Sec. 17.4, p. 279) |
| A message that names who must act | An ambiguous message drove six months of wrong action (Sec. 17.4, p. 281–283) |
| A scriptable administration interface | Nygard cannot overstate its value (Sec. 17.6, p. 296) |

**A sponsor asks for status, not for instantaneous behavior** (Sec. 17.1, p. 274). A sponsor needs to know whether
revenue tracks to plan, and whether the last campaign raised the conversion rate. A status dashboard, historical
trends, and future projections serve those questions. SNMP traps and thread dumps do not (p. 275). Nygard also names
the embarrassment that operations fears. A director of marketing asks why one server runs a higher CPU than the
others, and nobody has read the chart all day.

---

## 10. War stories from this chapter

**The nightly update that changed nothing (Sec. 17.2, p. 275).** A retailer ran a major project to make items appear
on the site faster. The nightly update finished at 5 or 6 a.m., and the business needed it closer to midnight. The
project optimized the batch jobs that fed content to the site, and it met its goal, because those jobs then finished
two hours earlier. Items still did not appear until a long-running parallel process finished at 5 or 6 a.m.

**The caches that invalidated each other (Sec. 17.2, p. 275).** Every time the system displayed
an item, it also updated that item by accident. That update sent a cache invalidation notice to
every other server. Each application server knocked items out of the caches of all the other
servers. Per-server visibility hid the effect. As soon as the statistics of all the caches appeared
on one page, the problem was obvious. Without that view the team would have added many servers, and
each new server would have made the problem worse.

**Voodoo operations, and the message that named no actor (Sec. 17.4, p. 281–283).** Nygard watched an administrator
receive a page and immediately start a database failover. The message that triggered it was one he had written
himself. It was a debug message. It said that an encrypted channel to an outside vendor had served enough
data that the key would soon be vulnerable. The application reset that channel itself right after
it emitted the message. The message had nothing to do with the database. Six months earlier it had
happened to be the last thing logged before a Sybase server went down. That server needed a vendor
patch. There was no causal connection, only a temporal connection. Three things then produced
weekly database failovers during peak hours for six months. They were that temporal connection, the
ambiguous wording "Reset required", and a debug message left enabled and forgotten.

**Why that superstition formed (Sec. 17.4, sidebar, p. 281).** Nygard cites Shermer on why a person over-reports a
pattern. An early human who missed a real leopard was less likely to pass on genes than one who ran
from a harmless bush. A cheap false positive and a fatal false negative therefore favor
superstition. Treat a remediation whose only evidence is time order as unproven. This is code
M-28.

**Three Mile Island (Sec. 17.4, p. 280).** Operators misread the meaning of the coolant pressure and temperature
values, and they took exactly the wrong action at every turn. Nygard cites Chiles, *Inviting Disaster*, p. 49–63, to
establish that a log file is a human interface. Most of our systems will not vent radioactive steam, but they will
take thousands of dollars.

**Batch jobs that failed invisibly for 33 days (Sec. 17.1, p. 272).** Nygard states that a startling number of
business-level issues trace to batch jobs that failed invisibly for 33 days straight. The dashboard must show the
absence of a required event, not only its failure.

---

## 11. What this reference gives to a review

| You are about to | Check these |
|---|---|
| Add a log message | The level in Section 5.2. The message code. The trace identifier. The actor named in the text. The fields that must not appear |
| Add a dashboard panel | The color rule in Section 2. The facet that each audience needs. The required events, present by their absence |
| Add an admin view | The restriction on the dangerous channels (p. 274). Whether the interface is scriptable (p. 296) |
| Add a threshold | Nothing here. The threshold belongs outside the application. Read `06-the-operations-database.md` |

**Related catalog codes.** M-26 through M-34 in `observability-gap-catalog.md`. **Related references.**
`03-white-box-and-black-box.md` for the outside view. `04-time-series-and-rules.md` for the metric and the rule that
reads it. `06-the-operations-database.md` for the expectation and the review cadence. `../SKILL.md`, Section 8,
carries the log contract in short form.
