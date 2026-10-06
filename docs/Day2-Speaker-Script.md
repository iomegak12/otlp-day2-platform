# OpenTelemetry Day 2 — Speaker Script

*How to use this script.* Read the plain paragraphs aloud; they are written to be spoken. Lines in *[italic brackets]* are cues for you, not for the room. Each section has a time marker so you can see whether you are ahead or behind. The hands-on steps live in the runbook; when the script says *[Switch to the runbook]*, do the steps there, then come back here. When you are short of time, skip any paragraph marked *[Optional]*.

---

## Opening: From Pilot to Platform (10 minutes)

### 0:00 — Where we left off

Good morning, everyone, and welcome back. Yesterday was about the source of telemetry. We wrote OTLP by hand and read it field by field. We switched on zero-code instrumentation in Java and Python, added the business view with custom spans and metrics, fixed a metric that was quietly exploding in cost, followed one trade across HTTP, Kafka and a thread pool, and gave every service consistent names.

And remember the last thing I said yesterday: through all of Day 1, we never changed a backend. Everything happened before the data reached Jaeger, Prometheus and Loki. Today we move into the middle of that journey. Today is about the pipeline, and about everything that has to be true before a whole company can depend on it.

### 0:02 — Today's theme: from pilot to platform

The theme for today is "from pilot to platform".

A pilot proves that something works. A platform is something other teams depend on, at three in the morning, during the busiest trading hour of the year, under audit. Those are very different standards. A pilot can forget about memory limits, compliance, cost and alerting. A platform cannot.

So today we are going to take TradeNova's pilot and harden it, one real problem at a time.

### 0:03 — The TradeNova Phase 2 story

*[Tell this as a story. Keep the business part light.]*

Here is where TradeNova stands. Yesterday's pilot went well. Leadership saw a single trade followed from click to database, and they liked it. So they made a decision: OpenTelemetry becomes the company-wide observability platform. One central pipeline that every team sends to.

And immediately, the pilot's blind spots became your problem. The pilot only looked at a few web services. Now the platform has to cover scheduled jobs, plain servers, a cloud fleet that scales itself up and down, alerting that wakes the right people, and the compliance rules every financial firm lives under. You are the platform team. Everything today lands on your desk.

Meet today's cast. There are four of them, plus the platform itself.

`trade-api` is Java 21 with Spring Boot, instrumented with the OpenTelemetry Java agent. It receives trades.

`portfolio-service` is Python with Flask, using zero-code instrumentation. It calls trade-api.

`eod-reconciliation` is a Python batch job. "EOD" means end of day: it reconciles the day's trades, the way a real brokerage does every night. In the lab it runs every ninety seconds so we don't have to wait until midnight, and it fails at random, like real batch jobs do.

The market-data fleet is a group of servers that scale up and down. They are simulated AWS EC2 instances, and each one runs a node exporter.

And the platform host — the machine running all of this — has its own node exporter too.

### 0:05 — The architecture, spoken

*[Put the architecture diagram on screen. Walk it from left to right. Point as you speak.]*

Let me walk you through the lab from left to right, following the data.

On the left are the applications. trade-api and portfolio-service send OTLP — traces, metrics and logs — to a Collector we call the **agent**. In production, the agent runs next to the applications: one per host, or a sidecar, or a DaemonSet in Kubernetes. Its job is everything that should happen close to the source: adding labels, protecting memory, masking sensitive data, and dropping noise.

From the agent, traces go to **Tempo**, Grafana's trace store. Metrics go to **Prometheus**, but not by scraping. The agent *pushes* them using Prometheus remote write. Prometheus has its remote-write receiver switched on, so it accepts samples over HTTP like any other time-series database.

Logs start out going straight from the agent to **Loki**. Later today they will take a detour through **Kafka**. Kafka is a durable buffer. Behind it sits a second Collector, which we call the **gateway**. The gateway reads from Kafka and delivers to Loki. You'll see why that matters in scenario five.

Now the non-OpenTelemetry side, because a real platform is never pure. Prometheus also *scrapes*: the node exporter on the platform host; the market-data fleet, which it finds by asking a **fake AWS EC2 API** that answers exactly like real AWS; a **Pushgateway**, where short-lived batch jobs leave their results; and both Collectors' own metrics, so we can watch the pipeline itself.

Prometheus evaluates alert rules and sends firing alerts to **Alertmanager**. Alertmanager turns them into emails. Those land in **Mailpit**, a local fake mail server with a web inbox, so we can read every alert without spamming anyone.

And on top of everything sits **Grafana**, connected to Prometheus, Loki and Tempo, with links between them.

### 0:08 — How each scenario runs

Today has nine scenarios, and each one follows the same rhythm.

First, I tell you the problem TradeNova reports. Then I explain how the thing really works underneath — the mechanics, not just the name of the feature. Then we switch to the runbook and **show the problem**, live. You'll see it broken. Then a short bridge, and we **apply the fix**, and look at the result together.

That order is deliberate. You'll see it broken before you see it fixed. If you've only seen the fix, you don't really know what it protects you from. And when this breaks in production, the broken version is what you'll be looking at.

*[Ask the room:]* Show of hands: who has been paged for a problem in the monitoring system itself? *[Then say:]* That's today's topic. A platform needs observing too.

---

## Scenario 1: Unlabelled Data (20 minutes)

### 0:00 — The problem TradeNova reports

The first complaint came the day the platform opened to more teams. Data is arriving, but nobody can tell where it came from. There's no environment on it, so test traffic and production traffic look the same. There's no team, so when a metric explodes, nobody knows whom to call. There's no region, and the finance department wants a cost centre on everything so it can charge teams for what they send. Every team has been asked to add these labels themselves. Some did, some didn't, and the ones who did all spelled them differently.

### 0:02 — How it works: resource attributes, once more

Let me remind you of yesterday's distinction, because today it becomes operational.

Span attributes describe *one operation*: this HTTP request, this route, this status code. Resource attributes describe *who produced the data*: this service, on this host, in this environment. Every span, every metric data point and every log record from one process shares the same resource.

Inside OTLP, the resource sits at the top. A message is a list of resources, each with its scopes, each with its spans or metrics or logs. So when the Collector changes a resource attribute, it changes it once, and that one change applies to everything underneath it. That is cheap, and it is why environment, team and region belong on the resource, not on every span.

Now, why do this in the Collector instead of asking every team? Three reasons. First, consistency: one config file decides the spelling, for everyone. Second, no redeploys: if finance changes the cost centre, you change one Collector config, not fifty applications. Third, the applications stay simple. A developer should not need to know the cost centre to ship a service.

### 0:05 — How it works: three processors

We'll use three processors in the agent, and each one does a different job.

The first is `resource_detection`. It asks the environment about itself. We use the `system` detector, which reads the operating system and adds `host.name` and `os.type`. There are detectors for EC2, Azure, Google Cloud, Docker and Kubernetes as well.

Here's a mechanic that matters. Detection runs once, when the Collector starts, and it describes the machine *the Collector* is running on. For an agent that sits on the same host as the applications, that's exactly right. For a central gateway it would be wrong: every span would be stamped with the gateway's host name, not the application's. So resource detection belongs in agents, not gateways.

The second is the `resource` processor. It applies fixed values with an action. There are a few actions, and the difference between them is subtle, so listen carefully.

`insert` adds the attribute only if it isn't already there. If the application set a value, the application wins.

`update` changes the attribute only if it *is* already there. If it's missing, nothing happens.

`upsert` does both: it overwrites the value if it exists, and adds it if it doesn't. The platform wins.

So in our lab we upsert `deployment.environment.name` to "training" and `cloud.region` to "ap-south-1". Those are facts about where the data is, and the platform should enforce them. But we *insert* `tradenova.cost_center` with "CC-4410". That's a default. If a team has its own cost centre, we respect it.

*[Pause. Let that difference land.]*

The third is the `transform` processor, and this is where the real power is. It runs statements written in OTTL, the OpenTelemetry Transformation Language. OTTL lets you read and write any field of the data, with conditions.

In our lab, the statements say, in plain words: set the resource attribute `tradenova.team` to "trading-core" *where* the resource attribute `service.name` is "trade-api". And set it to "wealth-apps" where the service name is "portfolio-service". The `resource` processor can't do that. It applies the same value to everyone. OTTL adds the `where` clause.

### 0:09 — How it works: OTTL contexts

One more idea you need for OTTL, because it comes back in scenarios three and four: contexts.

A context is the level of the data a statement works on. There's a `resource` context, a `scope` context, a `span` context, a `spanevent` context, a `log` context, a `metric` context and a `datapoint` context. A statement in the span context runs once for every span. A statement in the resource context runs once for every resource.

So the context decides two things: which fields you can reach, and how many times the statement runs. If you only need resource fields, use the resource context; it runs far fewer times, so it's cheaper. In recent Collector versions, the Collector usually works out the context from the paths you use. Older configs state it explicitly. You'll see both styles in the wild.

### 0:11 — Show the problem

*[Switch to the runbook: Scenario 1, Show the problem]*

*[While the runbook steps run, say:]* Look at `target_info` in Prometheus. Remember from yesterday: that's where resource attributes end up. Today it has the service name and a few SDK details. Nothing about environment, team or region. Now open Loki and look at the labels: the same story.

### 0:14 — Bridge

So: three missing labels, three different jobs. Detect what the machine can tell us. Enforce what the platform knows. And decide conditionally what only a rule can know. One processor each.

### 0:15 — Apply the fix

*[Switch to the runbook: Scenario 1, Apply the fix]*

### 0:17 — What to point out in the result

*[Point at each one on screen.]*

Look at `target_info` again. It now has `cloud_region`, `deployment_environment_name`, `tradenova_team` and `tradenova_cost_center`. The dots became underscores, the same rule we saw yesterday.

Look at Loki. `cloud_region` and `deployment_environment_name` appear as *labels*. But `tradenova_team` does not appear as a label. Here's why. When Loki receives OTLP, it promotes only a fixed list of well-known resource attributes into index labels. Everything else goes into what Loki calls structured metadata. You can still filter on it, but it isn't part of the index. That protects Loki from high-cardinality labels. You can change that list in Loki's config, but think hard before you do.

And in Tempo, open any trace and look at the resource attributes. They're all there, on every span.

Not a single application was redeployed.

*[Watch out: the agent log will show deprecation warnings if you use old component names. In Collector version 0.161, several components were renamed: `resourcedetection` became `resource_detection`, `otlphttp` became `otlp_http`, the `otlp` exporter became `otlp_grpc`, `prometheusremotewrite` became `prometheus_remote_write`, and `metricstransform` became `metrics_transform`. The old names still work, but they log warnings. Participants who copy older blog posts will hit this.]*

### 0:18 — Datadog lens

**Datadog lens.** In Datadog you get this from two places. Host-level tags come from `DD_TAGS` on the agent and from the cloud integration, which reads EC2 tags and attaches them to everything from that host. Service-level tags come from unified service tagging. The resource processor is the equivalent of `DD_TAGS`: one place, on the agent, that stamps everything passing through. And the difference is that with the Collector, you can make the tag conditional, with OTTL, which the Datadog agent can't do.

### 0:19 — Watch out and ask the room

*[Watch out:]* Upsert is powerful, and that makes it dangerous. If you upsert `deployment.environment.name` in a shared gateway, every team's data gets the same environment, including production data sent there by mistake. Upsert things you are sure about, close to where you are sure about them.

*[Ask the room:]* For the team label, we used a rule on service name. What happens when a new service appears that isn't in our rule?

*[Take answers. Then say:]* It simply gets no team. Nothing fails. That's why many companies add a last statement that sets the team to "unassigned" where the team is still missing. Then you can count unassigned telemetry on a dashboard and chase it, instead of not noticing.

---

## Scenario 2: Market-Open Spike (25 minutes)

### 0:00 — The problem TradeNova reports

The second complaint came from the market open. At nine-fifteen every morning, traffic jumps. On one bad morning, Tempo slowed down at the same time. The agent Collector, which runs in a container with a limit of 320 mebibytes, was killed by the kernel. When it restarted, it had lost everything it was holding. And for a few minutes, applications couldn't send anything at all. The pipeline that was supposed to explain the incident became part of the incident.

### 0:02 — How it works: where memory goes in a Collector

Let's understand where the memory actually goes.

Data comes in through a receiver, which decodes the request — protobuf bytes become Go objects in memory. The data flows through the processors to the exporter, which has a sending queue. By default, that queue holds a thousand *requests*, not a thousand spans. When the backend is slow or down, the queue fills, and each request in it is real memory. Meanwhile, new data keeps arriving and being decoded.

Now add Go's garbage collector. By default, Go runs a collection when the heap has grown to about double what was live after the last collection. Go does not know about your container limit. So if two hundred megabytes are live, Go may happily let the heap grow towards four hundred before it cleans up. In a 320 mebibyte container, the kernel kills the process first. That's an out-of-memory kill. No warning, no graceful shutdown, and everything in the queue is gone.

### 0:05 — How it works: the three layers of the fix

The fix has three layers, and in our testing all three were needed.

**Layer one: the `memory_limiter` processor, placed first.** It checks memory use on a timer — we use every second. It has two thresholds. The hard limit is a percentage of the container memory — we use seventy-five percent, which is 240 mebibytes. The spike limit is a percentage too — twenty-five percent, or 80 mebibytes. The soft limit is the hard limit minus the spike limit: 160 mebibytes.

Above the soft limit, the memory limiter starts refusing data. It returns a *retryable* error to the receiver, and the receiver passes that back to the sender. In OTLP terms, the sender is told: "not now, try again later". A well-behaved SDK or upstream Collector backs off and retries, and it holds the data in its own buffer meanwhile. Above the hard limit, the limiter also forces a garbage collection.

Why must it be first? Because it can only refuse data it sees. If it sits after other processors, memory is already spent on that work before the refusal.

**Layer two: the `batch` processor, placed last.** It collects spans and sends them in larger requests. We use a batch size of 2048 and a timeout of two seconds. Remember, the sending queue counts requests. A thousand requests of a handful of spans each is very little; a thousand requests of two thousand spans each is vastly more. And fewer, bigger requests are cheaper for the backend.

Why last? Because everything before it — filtering, transforming — should happen on the stream first, and the batch should be built from the final data.

**Layer three: `GOMEMLIMIT`.** This is an environment variable for the Go runtime, not a Collector setting. It tells the garbage collector: "try hard to keep total memory below this value". As memory approaches the limit, Go collects more often. We set it to 220 mebibytes — below the container's 320, leaving room for memory Go doesn't manage directly.

### 0:10 — The finding from testing

*[Slow down here. This is the most valuable part of the scenario.]*

I want to tell you what happened when we tested this, because it surprised us.

In the state before this fix — the enrichment processors only, no memory protection, no batching — we paused Tempo and flooded the agent. Memory climbed to about 1.7 gigabytes. In a 320 mebibyte container, that's an OOM kill. Tens of thousands of requests were rejected, because the sending queue of a thousand requests filled up.

Then we added the memory limiter alone. And memory *still* spiked to about 1.7 gigabytes.

Why? Two reasons. First, the receiver decodes each request *before* the memory limiter gets to look at it. Under a flood, many requests are being decoded at once, so memory grows between the limiter's checks. Second, Go's garbage collector still didn't know about the container limit, so garbage piled up even when the limiter was refusing.

Then we added `GOMEMLIMIT`. Peak memory stayed at about 256 megabytes. The agent survived. It refused about 3.8 million spans during the flood, and when Tempo came back, it delivered its backlog.

So the lesson is: the memory limiter is necessary but not sufficient. It decides *what* to refuse. `GOMEMLIMIT` makes Go actually give the memory back.

### 0:12 — Show the problem

*[Switch to the runbook: Scenario 2, Show the problem]*

*[While it runs, say:]* Watch the Collector's own metrics while this happens. `otelcol_process_memory_rss` is the memory the process really uses. `otelcol_exporter_queue_size` is how full the sending queue is. Note: in this Collector version, these self-metrics have no "underscore total" suffix, so don't add one in your queries.

*[When the agent is killed or the queue is full, say:]* There it is. The queue hits its ceiling, and memory keeps climbing until the container is gone.

### 0:15 — Bridge

What we want instead is a Collector that says "no" early and stays alive. Losing some data on purpose, in a controlled way, is much better than losing all of it by accident.

### 0:16 — Apply the fix

*[Switch to the runbook: Scenario 2, Apply the fix]*

### 0:18 — What to point out in the result

*[Point at each metric as you name it.]*

Memory now rises and then flattens. It doesn't keep climbing.

`otelcol_receiver_refused_spans` is going up. That's the memory limiter saying "not now". `otelcol_receiver_accepted_spans` keeps moving too, so data is still flowing — just less of it.

When Tempo comes back, watch `otelcol_exporter_queue_size` drain and `otelcol_exporter_sent_spans` jump. That's the backlog being delivered.

And this is the key point: the loss is *bounded* and *visible*. We know exactly how many spans were refused, because there's a counter for it. When a Collector crashes, there's no counter. You just have a gap, and you don't know how big it is.

### 0:20 — Processor order and gateways

So the rule for processor order is: memory limiter first, batch last, everything else in between.

One more mechanic for later in the week. When a gateway Collector runs a memory limiter and refuses data, the agents sending to it get that retryable error. The agents' exporters then retry, and their own queues absorb the pressure. So pressure travels backwards, from the backend, through the gateway, to the agents, and eventually to the SDKs. That's called backpressure, and it's healthy. It's the system telling everyone to slow down, rather than one component quietly dying.

### 0:21 — Datadog lens

**Datadog lens.** With the Datadog agent, most of you never tuned this, because the vendor tuned it for you. The agent has its own internal limits and drops payloads when it is overloaded, and you see that in the agent's own telemetry. With the Collector, it's your job. That's the price of control. The good news is that the Collector exposes every counter you need to see it happening.

### 0:22 — Watch out and ask the room

*[Watch out:]* Set the memory limiter as a percentage of the container limit, or as an absolute value, but make sure it matches the real limit. If the container limit changes and the Collector config doesn't, the limiter protects nothing.

*[Watch out:]* Senders that don't retry will lose refused data. The load generator we used doesn't retry, which is why we saw millions of refused spans. Real OpenTelemetry SDKs do retry, but their buffers are bounded too.

*[Watch out:]* Alert on `otelcol_receiver_refused_spans` going up, and on the queue size staying near its capacity. Those are your early warnings.

*[Ask the room:]* We set `GOMEMLIMIT` to 220 mebibytes in a 320 mebibyte container. Why not set it to 320?

*[Take answers. Then say:]* Because `GOMEMLIMIT` is a soft target for the Go runtime, not a hard wall. Some memory isn't counted, and Go can overshoot briefly under load. If the soft target equals the hard wall, the kernel kills you during the overshoot. You leave headroom.

---

## Scenario 3: PII Compliance (20 minutes)

### 0:00 — The problem TradeNova reports

The third problem came from compliance, not from engineering. An internal audit sampled a few hours of logs from the new platform. They found account numbers, customer email addresses, and full sixteen-digit card numbers. They also found spans carrying the customer's email and account number as attributes. In a financial firm, that's not a cleanup task. That's a finding with a deadline. And the fixes in the application code will take months, because several teams own those services.

### 0:02 — How it works: why mask at the agent

The first question is *where* to remove sensitive data. The answer is: as early as possible, and before it crosses a boundary.

Think about the journey. Data leaves the application, goes to the agent, then across the network to a gateway, then into Kafka, then into a backend — which might be a vendor's SaaS in another country. Every hop is another copy, another system to secure, and another place the auditors will ask about. If you mask in the backend, the sensitive data has already travelled the whole way and been stored, at least briefly. If you mask at the agent, it never leaves the host.

There's also a practical reason: backend rules are per backend, and two sets of rules drift apart. Rules in the agent apply to every destination at once.

### 0:04 — How it works: OTTL functions for masking

We'll use the `transform` processor again, with a second set of statements we call `transform/pii`. The part after the slash is just a name, so you can have several transform processors with different jobs.

For logs, we work on the log body with `replace_pattern`. It takes a regular expression and a replacement. In plain words: in the log body, replace anything that looks like an email address with "angle-bracket email-redacted". Replace any sixteen-digit number with "card-redacted". And replace "ACC dash" followed by digits with "ACC dash" and eight stars. We keep the "ACC" prefix on purpose: people can still see that an account was involved, without seeing which.

For spans, we work on attributes. For the email attribute, `tradenova.customer.email`, we use `delete_key`. There is no good reason for an email on a span, so it's simply removed. For `tradenova.account.number` we use `replace_pattern` again, so the attribute stays, but masked.

A few more OTTL functions. `replace_all_patterns` applies a pattern across *all* attributes of a map — useful when you don't know which attribute a team used. `keep_keys` is the opposite of delete: list what you allow, and the rest is removed. And `SHA256` hashes a value into a pseudonym. The same account number always gives the same hash, so you can still join and count without storing the number.

*[Watch out, say it clearly:]* Hashing a short, predictable value is not the same as hiding it. An eight-digit account number has only a hundred million possibilities. Anyone can hash all of them in minutes and build a lookup table. If you hash for privacy, add a secret salt before hashing, and keep that salt out of the config repository.

### 0:08 — How it works: the redaction processor and regex escaping

There's another tool: the `redaction` processor. It works the other way round. You give it an *allow-list* of attribute keys, and it removes every attribute not on the list. You can also give it patterns for values to block. The transform approach is a deny-list: "remove what we know is bad". The redaction approach is an allow-list: "keep only what we know is fine". An allow-list is safer, because new attributes are removed by default. But it's more work to run, because every new legitimate attribute needs to be added.

Now a mechanic that will bite you in the lab. In OTTL, regular expressions are written inside quoted strings. A backslash in a string is an escape character. So a regex like "backslash d" for a digit has to be written with *two* backslashes in the config. One backslash is eaten by the string, and the second one reaches the regex engine. If you write only one, the pattern silently matches something different, or nothing at all, and your masking does nothing.

*[Pause.]*

And the order of statements matters. Statements run top to bottom. If one pattern is broader than another, run the specific one first, or the broad one will eat the text before the specific one sees it.

### 0:11 — Show the problem

*[Switch to the runbook: Scenario 3, Show the problem]*

*[While it runs, say:]* Search Loki for "ACC". Look at the funding check lines and the full card numbers. Then open a trade-api trace in Tempo and look at the span attributes. The email is sitting right there.

### 0:13 — Bridge

So we have three kinds of sensitive data in two signals. The goal is not to stop logging. The goal is that the log line still tells the story, without identifying the customer.

### 0:14 — Apply the fix

*[Switch to the runbook: Scenario 3, Apply the fix]*

### 0:16 — What to point out in the result

*[Read these two lines from Loki aloud.]*

"Trade accepted for account ACC-star-star-star-star-star-star-star-star, customer email-redacted." And: "Funding check passed for card card-redacted." The log still tells you what happened. It just doesn't tell you who.

Open a new trace in Tempo. The email attribute is gone, and the account number is masked. Look closely at the HTTP attributes too: the URL path, the full URL and the query string also carried account numbers and e-mails — `url.path`, `url.full`, `url.query`. Instrumentation records URLs automatically, so a masking rule that only covers the attributes you added yourself is not enough. Ours masks those too, and the access-log lines, where the e-mail is URL-encoded as percent-four-zero instead of an at sign.

Now one honest point. Go to Prometheus and look at the trades metric. It still has an account label, with real account numbers in it. We haven't touched metrics yet. That is fixed in the next scenario, for a different reason — cost — and it fixes the compliance problem at the same time.

### 0:17 — Datadog lens

**Datadog lens.** Datadog gives you three places for this. The agent has log processing rules with `mask_sequences`, which is regex masking on the host, very much like what we just did. The APM side has obfuscation and tag replacement rules. And the Sensitive Data Scanner runs in Datadog's backend, after ingestion. That scanner is good, but by the time it runs, the data has already left your network. For a regulated firm, the agent-side options — Datadog's or the Collector's — are the ones that keep the data on your side of the boundary.

### 0:18 — Watch out and ask the room

*[Watch out:]* Masking is pattern-based. It catches what your patterns describe, and nothing else. A card number with spaces in it, an email in a different format, or a customer name — none of those are caught by our rules. The real fix is not to log personal data in the first place. Treat Collector masking as a safety net, not as the solution.

*[Watch out:]* `replace_pattern` on the log body works when the body is a string. If a team sends structured logs, where the body is a map, you need to target the fields inside the map instead.

*[Watch out:]* Test your regexes against real samples before production. A regex that is too broad will mask trade IDs or timestamps, and break the investigations you're trying to support. A regex that is too narrow gives you false comfort.

*[Ask the room:]* Should we mask at the agent *and* in the backend?

*[Take answers. Then say:]* Many regulated firms do both. The agent is the main control. The backend scanner is the detective control: it tells you when something slipped past the agent, so you can add a rule. Defence in depth.

---

## Scenario 4: Observability Cost (20 minutes)

### 0:00 — The problem TradeNova reports

The fourth complaint came from finance, a month after the platform opened. The bill was much higher than planned. When the platform team looked, a lot of what they were paying for was noise. The load balancer checks both services' health endpoints every second, and every check becomes a trace. Every request writes DEBUG logs. And the trades metric carries the account ID as a label. With five thousand accounts, that's thousands of time series for one metric.

### 0:02 — How it works: where cost comes from

Let's be precise about what you pay for, because each signal costs differently.

For metrics, the unit of cost is the *series*: every unique combination of metric name and label values. A series costs memory in Prometheus and money in Datadog, whether it gets one sample a day or one per second.

For traces, the unit of cost is the *span*. Every span is stored, indexed and kept for the retention period.

For logs, the unit of cost is *bytes*: ingested, indexed and stored.

And the cheapest data is the data you never send. Filtering in the agent saves money on the network, in Kafka, and in every backend after it.

### 0:04 — How it works: the filter processor

The `filter` processor drops data that matches a condition. We'll call ours `filter/noise`. Its conditions are written in OTTL, the same language as before.

We drop three things. Spans where `url.path` equals "/health". Log records where the severity number is below `SEVERITY_NUMBER_INFO` — that means DEBUG and TRACE. And log lines whose body matches "GET /health", the access-log lines for the same health checks.

Why the severity *number*? OpenTelemetry gives every log record a number from one to twenty-four: TRACE is one to four, DEBUG five to eight, INFO nine to twelve, and so on. Severity text differs between languages; the number is standard. So "below INFO" works the same for Java and Python.

One detail: the filter drops a span, not a trace. A health check is a one-span trace, so the whole trace goes. Drop a span from the middle of a real trace and you leave a hole.

### 0:07 — How it works: aggregation in the Collector

Now the metric. We use the `metrics_transform` processor — `metrics_transform/cost` — with an operation called `aggregate_labels`. We tell it to keep only `symbol` and `side`, and to sum. Every series that has the same symbol and side gets added together into one.

Why does this give the right answer? Because of temporality, which we covered yesterday. The SDK exports this counter as *cumulative*: each export contains the running total of *every* series. So in one export, all the per-account series for one symbol and side arrive together, and adding them up gives the true total for that symbol and side. Next export, same again.

*[Pause. This is subtle.]*

That only works because everything arrives in the same export. If the data were delta, or if series for the same group arrived in different requests, summing within one batch would give the wrong answer. Know your temporality before you aggregate in the Collector.

And remember yesterday: we fixed a similar problem with an SDK view. A view is still the better fix, because the SDK never builds the expensive series at all. The Collector is the fix when you can't change the application quickly — which, for a platform team serving fifty teams, is most of the time.

### 0:10 — Show the problem

*[Switch to the runbook: Scenario 4, Show the problem]*

*[While it runs, say:]* Look at the number of series for the trades metric. Hundreds, and growing as new accounts trade. Then open Tempo: most traces are health checks. And in Loki, filter by DEBUG and see how much of the volume it is.

### 0:12 — Bridge

None of this data is wrong. It's just not worth what it costs. The goal is to decide, in one place, what's worth keeping.

### 0:13 — Apply the fix

*[Switch to the runbook: Scenario 4, Apply the fix]*

### 0:15 — What to point out in the result

Hundreds of per-account series became one series per symbol and side. Check the total against the old total: it's the same number. We didn't lose any trades, only the account breakdown.

And this also closes the compliance gap from scenario three: no more account numbers in the metric.

Now look at the filter's own counters: `otelcol_processor_filter_spans_filtered` and `otelcol_processor_filter_logs_filtered`. The log counter is much bigger than the span counter. Each health check produces one span, but every real request also writes several DEBUG lines, and those are dropped too. Watch the ratio over time: when it suddenly changes, something has changed in an application, and that's worth a look.

### 0:16 — Datadog lens

**Datadog lens.** Datadog has its own cost controls: exclusion filters on log indexes, Metrics without Limits to choose which tags are indexed, and ingestion controls for traces. They work, but most of them act *after* the data has reached Datadog. You still send it, and for some of them you still pay ingestion. Doing it in the Collector means the data never leaves. And it works the same for every backend, not just one.

### 0:17 — Watch out and ask the room

*[Watch out:]* Dropped data is gone. If you drop all health-check spans, you also drop the *failing* health checks. A common refinement is to drop health checks only when they succeed, and keep the ones that return errors.

*[Watch out:]* Filtering is not sampling. Filtering decides "this kind of data is never worth keeping". Sampling decides "we keep a representative share of the data that *is* worth keeping". We cover sampling in its own module; it builds directly on this.

*[Ask the room:]* What's the most expensive "noise" in your telemetry today?

*[Take answers. Typical ones: health checks, Kubernetes probes, verbose library logging, per-user or per-pod labels. Say:]* Every one of those can be fixed with exactly the two processors we just used.

---

## Scenario 5: No Lost Logs (25 minutes)

### 0:00 — The problem TradeNova reports

The fifth problem came from an incident review. During a Loki upgrade, Loki was unavailable for about two minutes. Afterwards, the logs for those two minutes were simply missing. There was a trade dispute that week, and the logs that would have settled it weren't there. The audit team asked a fair question: "How can logs that the application wrote successfully just disappear?"

### 0:02 — How it works: why logs were lost

Let's look at the starting state. The agent sends logs straight to Loki. Its exporter has `retry_on_failure` with `max_elapsed_time` of thirty seconds.

Here's what happens when Loki goes down. The exporter tries, fails, and retries with a growing wait between attempts. Meanwhile, new logs keep arriving and piling up in the sending queue, in memory. After thirty seconds of retrying, the exporter gives up on that request and drops it. It logs an error, and the data is gone. Every request that reaches the thirty-second limit is dropped the same way. Two minutes of downtime becomes roughly a minute and a half of missing logs.

You could raise the retry time. But then the data waits in the agent's memory. A long outage fills the queue, and we're back in scenario two. Memory is not a safe place to wait.

### 0:05 — How it works: Kafka, in five ideas

So we need somewhere durable for data to wait. That's Kafka. Some of you run Kafka, some of you don't, so here are the five ideas you need.

A **topic** is a named log of messages — ours is `tradenova-logs`. Messages are appended and stay there; reading does not delete them. A topic is split into **partitions** — we have three — each an ordered sequence, read in parallel. Every message in a partition has an **offset**, its position number.

A **consumer group** is a set of readers sharing the work. Kafka gives each partition to exactly one member, and the group remembers how far it has read in each partition: its committed offset. **Lag** is the newest offset minus the committed offset — in plain words, how many messages are waiting.

And **retention**: our topic keeps messages for twenty-four hours, read or not. So Loki can be down for up to a day before anything is lost.

### 0:09 — How it works: the agent and gateway pattern

Now the new design. The agent sends logs to the Kafka exporter, encoded as `otlp_proto` — the same OTLP protobuf as on the wire, so nothing is lost in translation. Writing to Kafka is fast and Kafka is built to be highly available, so the agent rarely waits.

The gateway Collector runs the Kafka receiver, in a consumer group called `otel-gateway`. It starts from the earliest offset if the group has no committed position yet. And it has `message_marking` set to `after: true`. That means: mark a message as consumed *only after* the pipeline has successfully delivered it.

The gateway's exporter sends to Loki with retry forever — `max_elapsed_time` of zero means "never give up". And this is the important part: the gateway has *no* batch processor and *no* sending queue.

### 0:12 — How it works: why "no batch, no queue" is the guarantee

*[Slow down. This is the heart of the scenario.]*

Let's follow one message when Loki is down. The Kafka receiver passes it into the pipeline. The exporter tries Loki, fails, and retries. Because there's no queue, the call doesn't return — it *blocks*. So the receiver doesn't mark the message as done, and doesn't read the next one. The gateway stops consuming, new logs keep arriving in Kafka, and the lag grows. Nothing waits in memory. Everything waits in Kafka, on disk.

When Loki comes back, the retry succeeds, the call returns, the offset is marked, and the receiver moves on. The gateway reads as fast as it can, the lag drains, and the gap in Grafana fills in. The logs are late, but they're complete.

Now, why would a batch processor or a sending queue break this? Because both of them *acknowledge early*. A batch processor accepts the data into its buffer and immediately returns success to the receiver. A sending queue does the same: it accepts the request into memory and returns success. Either way, the receiver thinks the data was delivered, marks the offset, and moves on. If the gateway then restarts while Loki is still down, the data in that buffer is gone — and Kafka thinks it was delivered. The guarantee only holds if "success" really means "Loki has it".

That's the difference between buffering and backpressure. Buffering says "I'll hold it for you", and holds it in a fragile place. Backpressure says "I can't take more yet", and leaves the data where it's safe.

### 0:15 — Show the problem

*[Switch to the runbook: Scenario 5, Show the problem]*

*[While Loki is stopped, say:]* We stop Loki for about two minutes. Watch the agent's log: retries, and after thirty seconds, dropped data.

*[When Loki is back, say:]* Now look at the log volume in Grafana. There's the gap. And it will stay a gap. Nothing is coming back to fill it.

### 0:18 — Bridge

So the logs weren't lost by Loki. They were lost by the pipeline, which had nowhere safe to keep them. Let's give it somewhere safe.

### 0:19 — Apply the fix

*[Switch to the runbook: Scenario 5, Apply the fix]*

*[While Loki is stopped again, run the consumer-group describe command from the runbook — `kafka-consumer-groups.sh` with `--describe --group otel-gateway` — a few times. Say:]* Look at the LAG column. It grows, partition by partition. That's our logs, waiting safely.

### 0:21 — What to point out in the result

When Loki returns, run the describe command again. The lag drops back towards zero. In Grafana, the gap fills: the log volume for those two minutes appears, a little after the fact. Late but complete.

And point out one more thing: during the outage, the agent didn't struggle at all. It kept writing to Kafka. The outage was invisible to the applications.

### 0:22 — Datadog lens

**Datadog lens.** The Datadog agent remembers its position in the log files it tails, so after a short outage it can pick up where it left off. But anything held in its memory buffer is at risk, and retries are bounded. For a durable buffer in the Datadog world, you'd use Observability Pipelines, which can buffer on disk. The Kafka pattern gives you the same durability, with a buffer you own. And there's a bonus: once logs are in Kafka, you can add another consumer group — for security analytics, or a long-term archive — without touching a single agent.

### 0:23 — Watch out and ask the room

*[Watch out:]* This is at-least-once delivery, not exactly-once. If the gateway crashes after Loki accepted a message but before the offset was committed, that message is delivered again. Expect occasional duplicates.

*[Watch out:]* The backend has to accept late data. A two-minute delay is no problem for Loki. For an outage of many hours, check Loki's settings for out-of-order writes and for rejecting old samples, or the catch-up will be refused.

*[Watch out:]* Kafka itself is now critical infrastructure. Replication, disk space and monitoring of the lag are your job. Alert on lag growing, not just on Kafka being up.

*[Optional]* There's a lighter alternative to Kafka: the Collector exporter's *persistent queue*, which stores the sending queue on local disk using the `file_storage` extension. It survives a Collector restart. We'll look at it on Day 3.

*[Ask the room:]* We have three partitions. What happens if we run five gateway Collectors?

*[Take answers. Then say:]* Three of them get one partition each, and two sit idle. Partitions are the unit of parallelism, so plan the partition count up front.

---

## Scenario 6: Silent Batch Failure (20 minutes)

### 0:00 — The problem TradeNova reports

The sixth problem is a quiet one. The end-of-day reconciliation job failed three nights in a row, and nobody noticed. On the fourth morning, finance found that the books didn't match. The job writes logs, but nobody reads batch logs unless they already know something is wrong. There were no metrics and no alerts.

### 0:02 — How it works: why Prometheus can't see a batch job

Prometheus works by *pulling*. Every fifteen seconds, in our lab, it calls each target's metrics endpoint and stores what it finds.

Now think about a batch job. It starts, works for a few seconds, and exits. Its metrics endpoint exists only while it runs, so a scrape would have to land in exactly those few seconds — and even then it would see the job half-way, never the final result. Pull simply doesn't fit short-lived work.

In the starting state, our job has push turned off, so it only writes logs. That's why nobody noticed.

### 0:04 — How it works: the Pushgateway

The Prometheus answer is the **Pushgateway**. The job pushes its metrics to the Pushgateway when it finishes. The Pushgateway keeps them. And Prometheus scrapes the Pushgateway like any other target.

Our job pushes five metrics. `eod_job_success` is one for success and zero for failure. `eod_job_last_success_timestamp_seconds` is the time of the last successful run. `eod_job_last_run_timestamp_seconds` is the time of the last run, success or not. `eod_job_duration_seconds` is how long it took. And `eod_job_records_processed` is how much work it did. The Pushgateway also adds its own `push_time_seconds` for every group.

On the Prometheus side, the scrape job for the Pushgateway uses `honor_labels: true`. Here's why. Normally Prometheus stamps every scraped series with the `job` and `instance` labels of the *target* — which here is the Pushgateway. That would overwrite the job name the batch job pushed. `honor_labels` tells Prometheus: keep the labels that came with the data.

### 0:07 — How it works: push versus pushadd

*[Slow down. This is the mechanic that makes the alert work.]*

There are two ways to push. The first is a PUT, which in the client library is called `push`. A PUT replaces *every* metric in that group with what you send. The second is a POST, called `pushadd`. A POST replaces only the metrics you actually send, and leaves the others alone.

Our job uses `pushadd`. When it succeeds, it sends all five metrics, including the last-success timestamp. When it fails, it sends success equals zero, the last-run time and the duration — but *not* the last-success timestamp. So the Pushgateway keeps the old success timestamp from the last good run.

And that's what makes a "time since last success" alert possible. The query is `time()` minus `eod_job_last_success_timestamp_seconds`. In words: now, minus the last time it worked. After every failure, that number keeps growing. If the job used a plain PUT, a failed run would wipe the success timestamp, and you'd lose exactly the information you need.

### 0:09 — How it works: what the Pushgateway is not

Three warnings about the Pushgateway, because it's often misused.

It's a *cache*, not an aggregator: if two jobs push the same metric to the same group, the last one wins. It *never forgets*: a retired job's last values stay there, scraped forever, until someone deletes the group. And it's *not for services*: a long-running service should be scraped directly, or you lose its `up` metric and add a single point of failure.

Speaking of `up`: when Prometheus scrapes the Pushgateway, `up` tells you the *Pushgateway* is reachable. Nothing about whether the job ran. That's why we need the timestamps.

### 0:11 — Show the problem

*[Switch to the runbook: Scenario 6, Show the problem]*

*[While it runs, say:]* Look in Prometheus for anything starting with "eod". There's nothing. Now look in Loki: you can see the job running, and some runs say "failed". That's all the evidence there is, and nobody is looking at it.

### 0:13 — Bridge

So the job knows whether it succeeded. It just has no way to tell Prometheus. We give it a mailbox.

### 0:14 — Apply the fix

*[Switch to the runbook: Scenario 6, Apply the fix]*

### 0:16 — What to point out in the result

Open the Pushgateway web page and show the `eod_reconciliation` group, with its metrics and the push time.

In Prometheus, plot `eod_job_success`: a line jumping between one and zero as runs succeed or fail.

Then plot "time minus last success timestamp". Wait for a failed run. The value doesn't reset; it keeps climbing. After the next successful run, it drops back near zero. That's the shape your alert will watch in scenario eight.

### 0:17 — The OpenTelemetry angle

You might ask: we're at an OpenTelemetry training — why use a Prometheus tool?

There are three options. The job can send OTLP to the Collector, preferably with *delta* temporality, because a short-lived process has no meaningful running total. The job can write a file that node exporter's textfile collector picks up, which works well for jobs on a fixed host. Or it can use the Pushgateway, which is the Prometheus-native answer. Whichever you choose, the rule from yesterday still applies: flush and shut down before you exit, or the last data never leaves.

### 0:18 — Datadog lens

**Datadog lens.** Datadog doesn't need a Pushgateway, because DogStatsD is already push. A batch job sends a gauge to the local agent, and you write a monitor on it. And Datadog monitors have a "notify if data is missing" option, which plays the role of our "time since last success" query. The idea is the same: silence must be treated as a signal.

### 0:19 — Watch out and ask the room

*[Watch out:]* If the Pushgateway restarts without persistence, all pushed metrics vanish. Your alerts don't fire — they go silent, because there's nothing to evaluate. Turn on its persistence file, and consider an `absent` alert for critical jobs.

*[Watch out:]* Use stable grouping labels. If you put a run ID or a timestamp in the grouping key, every run creates a new group, and the Pushgateway fills up forever.

*[Ask the room:]* Our job failed three nights in a row. Which would have caught it sooner: an alert on success equals zero, or an alert on time since last success?

*[Take answers. Then say:]* The success alert catches a run that *fails*. The staleness alert also catches a run that *never happens* — the scheduler is broken, the job is stuck, the server is off. You want both. And that's exactly what we'll build in scenario eight.

---

## Scenario 7: Scaling Fleet (20 minutes)

### 0:00 — The problem TradeNova reports

The seventh problem comes from the market-data team. Their servers scale with demand: more servers at the market open, fewer at night. But Prometheus has a hand-written list of servers to scrape, and that list contains exactly one: market-data-1. Every other server is invisible. And when someone does add a server to the list, it stays there after the server is gone, showing as down.

### 0:02 — How it works: service discovery

The fix is *service discovery*. Instead of a static list, Prometheus asks an authority — here, the AWS EC2 API — "what should I scrape right now?" and builds its target list from the answer.

Here's the loop. Every refresh interval, which we set to fifteen seconds, Prometheus calls EC2's DescribeInstances API. We pass filters, and those filters are applied by AWS, on the server side: only instances with the tag `monitoring` equal to "enabled", and only instances in the "running" state. For every instance that comes back, Prometheus creates a candidate target, with port 9100 for node exporter, and attaches a set of metadata labels.

Then relabeling runs. Then the targets are scraped. When an instance appears in the answer, it becomes a target; when it disappears, the target is removed. In the lab, the EC2 API is Moto, a mock AWS server that answers DescribeInstances exactly like real AWS, so Prometheus runs its real EC2 discovery code. Only the endpoint differs.

### 0:05 — How it works: meta labels and relabeling

Every discovered target arrives with labels that start with double underscore "meta". For EC2, there are labels for each tag — `__meta_ec2_tag_Name`, `__meta_ec2_tag_team` — and for the instance ID, the instance type, the availability zone, the private IP and more.

Here's the mechanic people miss. Labels that start with a double underscore exist *only during relabeling*. After relabeling, Prometheus throws them away. If you don't copy them into normal labels, they're gone, and you can't query them.

So our `relabel_configs` copy them. The Name tag becomes `instance`. The team tag becomes `team`. The instance ID becomes `ec2_instance_id`. The instance type becomes `instance_type`. And the availability zone becomes `availability_zone`. Now every metric from every server tells you exactly which machine, which team and which zone it came from.

There are two kinds of relabeling. `relabel_configs` run *before* the scrape, once per target: whether to scrape it, where to connect, which labels it gets. `metric_relabel_configs` run *after* the scrape, on every sample: they drop or rename metrics, and cost more because they run so often.

One lab-only detail, so nobody copies it to production. In real AWS, Prometheus connects to the instance's private IP address, which it gets from `__meta_ec2_private_ip`. In our lab, each "instance" is really a container, and the fake API doesn't have container addresses. So we put the real scrape address in a tag called `scrape_target`, and one relabel rule copies it into the address. In real AWS, you delete that rule.

### 0:09 — Show the problem

*[Switch to the runbook: Scenario 7, Show the problem]*

*[While it runs, say:]* Open the Prometheus targets page. One market-data target. Now the fleet scales to five. Wait. Still one. Four servers are running, and we have no idea how they are.

### 0:11 — Bridge

The static list is a promise someone has to keep by hand. Let's give that job to the cloud API, which already knows the truth.

### 0:12 — Apply the fix

*[Switch to the runbook: Scenario 7, Apply the fix]*

### 0:14 — What to point out in the result

Run `fleet scale 5`. Within fifteen to thirty seconds, five market-data targets appear on the targets page. Why up to thirty? Fifteen for the next discovery refresh, and up to fifteen more for the first scrape.

Hover over a target to see the labels before and after relabeling. That's the best way to understand relabeling — Prometheus shows you both.

Then scale down. The targets disappear on the next refresh. They don't show as "down"; they're gone, because the instances are gone. And not one line of Prometheus config was touched.

### 0:15 — In real AWS and beyond

In real AWS, Prometheus needs permission to call the API. It needs `ec2:DescribeInstances`, and `ec2:DescribeAvailabilityZones` for the zone details. Give it those through an instance role, or a service account role in Kubernetes. Not through access keys in a config file.

EC2 is just one of many discovery mechanisms. There's `kubernetes_sd` for pods, services and nodes. There's Consul. And there's `file_sd`, where any tool can write a list of targets to a file, and Prometheus watches the file. That last one is the escape hatch for any inventory system Prometheus doesn't know.

And the OpenTelemetry angle: the Collector's `prometheus` receiver accepts the same scrape configuration — including `ec2_sd_configs` and relabeling — because it uses Prometheus's own scraping code. So everything you've just learned works in a Collector too.

### 0:16 — Datadog lens

**Datadog lens.** In Datadog, this problem mostly doesn't exist, because the agent runs *on* every host and pushes. A new instance starts, its agent starts, and it registers itself. On top of that, the AWS integration crawls the EC2 API with a role and attaches your EC2 tags to hosts. Prometheus pulls, so it needs to be told where to look. Service discovery is how pull systems get the same "it just appears" behaviour.

### 0:17 — Watch out and ask the room

*[Watch out:]* The filter on `monitoring` equals "enabled" means that an instance without that tag is invisible. That's useful for opting out, and dangerous if someone forgets the tag. Some teams flip it: scrape everything, and let a tag opt *out*.

*[Watch out:]* Discovery calls the AWS API on every refresh, from every Prometheus. Many servers with short intervals can hit API rate limits; fifteen seconds is a lab value.

*[Ask the room:]* We copy the instance's Name tag into the `instance` label. What could go wrong if two instances have the same Name tag?

*[Take answers. Then say:]* Their series collide. Two machines would write to the same series, and the graphs would jump between them. Labels that identify a target must be unique. If Name isn't guaranteed unique, use the instance ID.

---

## Scenario 8: On-Call Alerts (25 minutes)

### 0:00 — The problem TradeNova reports

The eighth problem is the obvious one. The platform now has the right data — but nobody is watching it at three in the morning. There are no alerts. A failed reconciliation should reach finance operations, and a dead server the platform on-call engineer. Today both reach nobody.

### 0:02 — How it works: the alert lifecycle

Prometheus evaluates alert rules on a timer. Each rule is a query plus a duration called `for`.

An alert has four states. **Inactive**: the query returns nothing. **Pending**: the query returns something, but it hasn't been true for long enough yet. That's what `for` controls. **Firing**: it has been true for the whole `for` duration. And **resolved**: it was firing, and now it isn't.

Why the pending state? Because short blips are normal. A server that misses one scrape shouldn't wake anyone. If the condition goes away during pending, the alert goes back to inactive, and nobody is told.

Here's the division of work, and it's important. *Prometheus* decides whether an alert is firing. That's all it does. It sends firing alerts to Alertmanager and keeps re-sending them while they fire. *Alertmanager* decides what humans see. It deduplicates, so the same alert from two Prometheus servers becomes one. It groups related alerts into one notification. It routes them to the right receiver. It applies silences, for planned maintenance. And it applies inhibitions: "if this alert is firing, don't tell me about that one".

### 0:06 — How it works: our rules

Let me walk through the rules in words.

`EodReconciliationFailed`: the job's success metric is zero, for thirty seconds. Team label: finance-ops.

`EodReconciliationStale`: time since the last success is more than three hundred seconds, for one minute. In the lab, three hundred seconds is long enough, because the job runs every ninety. For a real nightly job, you'd use about twenty-six hours: a day, plus some slack for a slow run.

`InstanceDown`: `up` equals zero for one minute, for the platform host and the market-data fleet. `HostHighCpu` and `HostLowDisk`: CPU above eighty-five percent, or less than ten percent disk free, for five minutes.

`ServiceHighErrorRate`: more than five percent of requests return 5xx, for one minute. It's computed from `http_server_request_duration_seconds_count` — the count part of the HTTP duration histogram. And `ServiceHighLatency`: the ninety-fifth percentile above half a second for one minute, excluding the health endpoint, which would pull the percentile down.

Notice which `job` label those service alerts use. For OpenTelemetry metrics, the job is built from `service.namespace` and `service.name` — so "tradenova slash trade-api". You saw that yesterday. If you write a rule with job equals "trade-api", it will never match.

### 0:09 — How it works: Alertmanager routing and timing

Now Alertmanager. Alerts are grouped by alert name and job. So if three market-data servers go down, that's one email with three alerts, not three emails.

Three timers control the emails. `group_wait` is ten seconds: when a new group appears, wait ten seconds in case more alerts join it, then send. `group_interval` is one minute: if the group changes — a new alert joins, or one resolves — wait at least a minute before sending an update. And `repeat_interval` is one hour: if nothing changes, remind people every hour.

Routing works on labels. Alerts with `team` equal to "finance-ops" go to finance-ops@tradenova.example. Everything else falls through to the default, platform-oncall@tradenova.example. And `send_resolved` is on, so people also get an email when the problem is over.

### 0:11 — How it works: symptoms, causes and burn rates

One piece of philosophy before the demo.

Alert on *symptoms*, not *causes*. A symptom is something users feel: errors, slowness, a job that didn't produce its result. A cause is something that might lead to a symptom: high CPU, a full disk. Causes are useful for dashboards and for low-urgency warnings. Symptoms are what should page a human. That's why our CPU and disk alerts are warnings with five-minute durations, and the error-rate alert is critical.

And remember error budgets from yesterday. A raw error rate above five percent is a simple start. A better page is a *burn rate* alert: "we're spending this month's error budget fourteen times too fast". It ignores small blips, and it fires hard when it matters.

### 0:12 — Show the problem

*[Switch to the runbook: Scenario 8, Show the problem]*

*[While it runs, say:]* Open the Prometheus alerts page. No rules. Open the Mailpit inbox. Empty. The data is all there — we just saw a failing job in scenario six — but nothing turns it into a message to a human.

### 0:14 — Bridge

So the eyes are there; we need a voice. Rules in Prometheus, routing in Alertmanager.

### 0:15 — Apply the fix

*[Switch to the runbook: Scenario 8, Apply the fix]*

*[Then run the trigger script again, as the runbook says: it stops a market-data container, makes the batch job fail every run, and injects a 30% error rate on trade-api. Applying the fix restarts some services, so the triggers must be re-applied.]*

### 0:18 — What to point out in the result

Watch the alerts page in Prometheus as the triggers land. Each alert goes yellow first — pending — then red — firing. Point out the time it spends pending. That's the `for` duration doing its job.

Then open Mailpit. The failed-job email is addressed to finance-ops, and the subject line reads: "FIRING colon one, EodReconciliationFailed, eod reconciliation, warning, finance-ops". That subject is built from the labels the alerts in the group share.

The InstanceDown and error-rate emails go to platform-oncall. Because portfolio-service calls trade-api, you may also get an error-rate email for its job. That's the symptom spreading to the caller.

Then undo the triggers. Over the next minutes, "RESOLVED" emails arrive.

### 0:20 — Datadog lens

**Datadog lens.** In Datadog, a monitor both evaluates the query *and* sends the notification. You route with at-mentions, re-notify plays the role of repeat interval, and downtimes the role of silences. Prometheus splits this in two. It's more to run, but many Prometheus servers — or Grafana, or Loki — can send to one Alertmanager that owns routing for the whole company.

### 0:21 — Watch out and ask the room

*[Watch out:]* An alert on a metric that doesn't exist never fires. If the Pushgateway loses the job's metrics, or a label name changes, the rule evaluates to nothing — which looks exactly like "everything is fine". For critical alerts, add an `absent` rule, and test your alerts by actually breaking things, as we just did.

*[Watch out:]* Use inhibition for obvious chains. If a host is down, don't also send its high-CPU alert. If the database is down, don't page every service that calls it.

*[Ask the room:]* When trade-api was failing, portfolio-service also showed errors. Should both page someone?

*[Take answers. Then say:]* Ideally the root symptom pages, and the downstream one is either inhibited or goes to a lower-urgency channel. But be careful: inhibition rules can hide a real second problem. Many teams accept one extra email rather than risk missing something. There's no perfect answer, and that's why the next scenario matters: when two services both look bad, you need to see *which hop* is really slow.

---

## Scenario 9: Slow Trades (25 minutes)

### 0:00 — The problem TradeNova reports

The last problem of the day. Customers complain that the portfolio page is slow. The portfolio team says it's trade-api. The trade-api team says their dashboards look fine and it must be the network. Two teams, one problem, and nobody can prove which hop is slow. Tempo has every trace, but nobody has time to open thousands of traces and compare them by hand.

### 0:02 — How it works: metrics from traces

Here's the idea. Every span already contains a service name, an operation name, a start time, a duration and a status. That's everything you need for RED metrics: rate, errors and duration. So instead of instrumenting metrics separately in every service, you can *compute* them from the spans as they arrive.

Tempo has a component for this, called the **metrics generator**. In our starting state, it's switched off: Tempo stores traces, but it produces no metrics and no service graph. We'll switch on two of its processors: `span-metrics` and `service-graphs`. The metrics generator remote-writes the results into Prometheus — the same remote-write path our agent already uses.

### 0:04 — How it works: span metrics

The span metrics processor looks at every span and increments counters and histograms. It produces `traces_spanmetrics_calls_total` — a count of spans — and `traces_spanmetrics_latency`, a histogram of durations. The labels are `service`, `span_name`, `span_kind` and `status_code`.

Notice the label is called `service`, not `service_name`. That's Tempo's naming. The Collector has a similar component with slightly different label names, so be careful when you copy dashboards between the two.

So one query gives you the rate, errors and percentiles of every operation of every service, including ones nobody built a dashboard for.

And now the cost. Every unique combination of service, span name, kind and status creates series — and for the histogram, one series per bucket. That's why yesterday's rule about generic span names matters so much. If a span name contains an order number, every order creates new series here. Low-cardinality span names aren't just tidy; they're what keeps this affordable.

### 0:07 — How it works: how the service graph is built

*[Slow down. This is the clever part.]*

The service graph processor builds edges between services. Here's how.

When portfolio-service calls trade-api, two spans are created. On the portfolio side, a CLIENT span — "I'm making a call". On the trade-api side, a SERVER span — "I'm handling a call". The SERVER span's parent is the CLIENT span. Same trace, different services.

Tempo watches for these pairs. When it sees a CLIENT span, it keeps it in a short-lived store, waiting. When the SERVER span arrives whose parent is that CLIENT span, it has a pair. Spans can arrive in any order, so it works both ways round. Each pair becomes one request on the edge "portfolio-service to trade-api". It records the count, whether it failed, and the latency, measured from both sides.

That gives us metrics like `traces_service_graph_request_total` and its failed and latency companions. And Grafana draws them as a graph.

Here's the subtle bit. The client-side latency includes the network and any queueing; the server-side latency is only the time the server spent. If the client sees eight hundred milliseconds and the server also sees eight hundred, the server is slow. If the client sees eight hundred and the server sees fifty, the time is lost *between* them. That's how you settle the argument between two teams.

This is also why span kinds matter so much, as we said yesterday. If an instrumentation marks its spans as INTERNAL instead of CLIENT, the pair never forms, and the edge is missing from the graph.

### 0:11 — How it works: navigating in Grafana

In Grafana, you'll use four jumps. The **service graph**, in Explore, under Tempo, in the Service Graph tab. **Trace to logs**: click a span, and Grafana queries Loki for that trace ID. **Trace to metrics**: from a span to its span metrics in Prometheus. And **log to trace**: a log line with a trace ID becomes a link.

And you can search traces with TraceQL. Let me say a few queries in words. "Find spans where the resource service name is trade-api and the duration is greater than five hundred milliseconds." In TraceQL, that's curly braces, resource dot service dot name equals trade-api, and duration greater than 500ms. "Find spans where status equals error." And a structural query: "find portfolio-service spans that have a *descendant* slower than five hundred milliseconds". TraceQL has an operator for "descendant of" — two greater-than signs. That one query answers "which slow portfolio requests were slow because of something further down?"

### 0:14 — Show the problem

*[Switch to the runbook: Scenario 9, Show the problem]*

*[After latency is injected on trade-api with the chaos endpoint at 800 milliseconds, say:]* portfolio-service is now slow. Go to Explore, Tempo, Service Graph. Empty — the metrics generator is off. Search Prometheus for anything starting with "traces". Nothing. We have all the traces, and still no overview.

### 0:16 — Bridge

The information is all inside the traces. We just need something to read every one of them and add them up. Let's turn the generator on.

### 0:17 — Apply the fix

*[Switch to the runbook: Scenario 9, Apply the fix]*

### 0:19 — What to point out in the result

Give it a minute for metrics to appear. Then open the Service Graph. You'll see portfolio-service, trade-api, and the edge between them. Look at the latency on the edge and on the trade-api node.

Now query span metrics for trade-api by span name. The trades endpoint is slow on the *server* side. So the time is spent inside trade-api, not on the network. Argument settled.

Then use a TraceQL query for slow trade-api spans, open one, and jump to its logs.

And exemplars. The metrics generator sends exemplars with its remote write. On a latency panel with exemplars switched on, each dot is a real trace ID. Click one, and you're in the trace behind that point — without anyone instrumenting exemplars in the application.

Finally, remove the injected latency from trade-api and watch the graph recover.

### 0:21 — Datadog lens

**Datadog lens.** This is something Datadog APM does for you automatically. Every service gets trace metrics — hits, errors and latency — and a service map, without any configuration. That's one of the main reasons people like Datadog APM. In the open-source world, you get the same thing from Tempo's metrics generator, as we just saw, or from the Collector's `spanmetrics` and `servicegraph` connectors, which compute the same metrics inside the pipeline and can send them to any backend. The difference is that you run it, and you control its cardinality.

### 0:22 — Watch out and ask the room

*[Watch out:]* Span metrics only see the spans that reach the generator. If you sample traces *before* Tempo, the counts are based on the sample, not on all traffic. That matters a lot with tail sampling. We'll come back to it in the backup answers.

*[Watch out:]* The service graph needs both halves of a call. If a downstream system isn't instrumented, there's no SERVER span, and the edge may never appear. Databases and third-party APIs are often like that.

*[Ask the room:]* We now have HTTP metrics from the SDK *and* span metrics from Tempo. Which would you use for an SLO?

*[Take answers. Then say:]* For an SLO, I'd use the SDK's metrics, because they count every request, regardless of sampling. Span metrics are excellent for exploration and for services that have no other metrics. Use each for what it's good at.

---

## Day 2 Close (5 minutes)

Let's look at what TradeNova's platform has become today.

It's **labelled**: every signal carries environment, region, team and cost centre, added centrally, without redeploying anything. It's **protected**: the agent refuses data before it runs out of memory, and the loss is counted, not hidden. It's **compliant**: account numbers, emails and card numbers are masked before they leave the host. It's **affordable**: health checks, DEBUG logs and per-account series are gone before they cost anything. It's **durable**: when Loki goes down, logs wait in Kafka and arrive late but complete. It **watches batch jobs**, through the Pushgateway and the "time since last success" trick. It **finds its own servers**, by asking the cloud API. It **wakes the right person**, with alerts routed to the team that can fix them. And it **shows where time goes**, with a service graph and RED metrics computed from traces.

*[Pause.]*

Notice something about today. Almost all of it was configuration. Two Collectors, Prometheus, Alertmanager and Tempo. Not a single line of application code changed. That's what a platform is: the place where these decisions are made once, for everyone.

Tomorrow we take this platform somewhere real. We'll deploy it on Kubernetes, with the OpenTelemetry Operator managing Collectors and instrumentation. We'll talk about scaling the Collectors, securing the pipeline end to end, and migrating from an existing Datadog or Prometheus setup without a big-bang cut-over.

*[Ask the room:]* Of the nine fixes today, which one would have saved you the most pain in the last year?

*[Take two or three answers. Thank everyone.]*

---

## Backup Answers for Common Questions

*[Read these only if the question comes up.]*

**"Why not let Datadog, or whichever vendor, do the masking?"**
You can, and it's a good second line of defence. But vendor-side masking happens after the data has left your network and been received by a third party. For many regulators, that's already the problem. Masking in the agent keeps the data on your side, and it applies to every backend at once, not just one vendor. Use the vendor's scanner to catch what slipped through, not as the main control.

**"Is Kafka overkill? Why not use the Collector's persistent queue?"**
For many teams, the persistent queue is enough. It stores the sending queue on local disk with the `file_storage` extension, so data survives a restart and a backend outage, as long as the disk has room. Kafka adds things the persistent queue can't: the buffer is shared and replicated rather than on one machine's disk, you can add more consumers without touching agents, and you can replay data. If you already run Kafka well, it's a natural fit. If you don't, start with the persistent queue. We'll look at it on Day 3.

**"Can the Collector replace Prometheus scraping entirely?"**
It can do the scraping: the `prometheus` receiver uses Prometheus's own scrape code and the same scrape configs, including discovery and relabeling. But the Collector doesn't store data, evaluate rules or answer queries, so you still need a time-series database. And with many Collectors, each target must be scraped by exactly one of them; in Kubernetes, the Target Allocator handles that, which we'll mention on Day 3.

**"Should batch jobs use the Pushgateway or send OTLP?"**
If the job is already instrumented with OpenTelemetry, send OTLP to the Collector, with delta temporality and a proper flush before exit. You get traces and logs from the same job, linked together. If the team only knows Prometheus, or the job is a shell script, the Pushgateway is simple and well understood. In both cases, the important part is the "last success" timestamp and an alert on how old it is.

**"How big should the agent's memory limit be?"**
There's no universal number. Run your real peak traffic plus a backend outage, and watch `otelcol_process_memory_rss`. Then set the container limit with headroom, the memory limiter as a share of it, and `GOMEMLIMIT` below the limiter's hard limit, as we did. Above all, keep the three numbers consistent and change them together.

**"How accurate are span metrics if we use tail sampling?"**
If span metrics are computed *after* sampling, they're wrong: they count only the kept traces, and tail sampling deliberately keeps more errors and slow requests, so error rates and latencies look much worse than reality. The fix is order. Compute span metrics *before* sampling — in the Collector, with the `spanmetrics` connector, ahead of the tail-sampling processor — so they see every span, then sample what you store. Tempo's metrics generator only sees what reaches Tempo.

**"Do we need both Tempo and Jaeger?"**
No. Both store and search traces. Yesterday we used Jaeger because its UI is a great way to learn; today Tempo, because it's integrated with Grafana, stores traces cheaply in object storage, and has the metrics generator. Pick one. Because everything is OTLP, switching later is a Collector config change.

**"What AWS credentials does EC2 discovery need in real life?"**
Read-only permission for `ec2:DescribeInstances`, plus `ec2:DescribeAvailabilityZones` if you want zone details. Give it through an IAM instance role, or a service account role in Kubernetes, so there are no long-lived keys anywhere. If Prometheus runs in a different account from the fleet, use a role it can assume in the target account. And remove the lab's `scrape_target` rule, so Prometheus connects to the private IP.
