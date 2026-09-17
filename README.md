# cloud infrastructure monitoring: key metrics, tool choices, and a self-hosted Prometheus stack that runs on a $7.95 VPS

Search "cloud infrastructure monitoring" and you'll get two very different kinds of results: abstract explainers about "observability" and vendor pages hoping you'll sign up for a per-host subscription. What most people actually want sits in the middle — which numbers to collect, which tool to collect them with, and where to run the whole thing without a bill that grows every time you add a server.

This guide covers all three, in that order. By the end you'll know the four signals worth alerting on, how the main tools compare on cost, and how to host a monitoring stack on infrastructure where bandwidth overage and DDoS attacks don't turn into surprise line items.

## What cloud infrastructure monitoring actually covers

Strip away the jargon and infrastructure monitoring answers three questions: is it up, is it fast, and is it about to break?

"Is it up" is uptime checking — ping the endpoint, get a 200, move on. "Is it fast" means latency: how long a request takes, measured at percentiles, because an average hides the slow requests that annoy users most. "Is it about to break" is where the real work lives — watching CPU, memory, disk I/O, network throughput, and error rates trend over time so you get a warning instead of an outage.

Notice what's missing from that list: anything about business metrics or user analytics. Infrastructure monitoring watches the layer *under* your application. If you run an e-commerce store, revenue-per-visitor isn't infrastructure monitoring. Disk filling up at 2GB/hour on your database server is.

The distinction matters because it determines your tool budget. Application observability platforms can easily run $50+ per host per month once you add APM and logs. Pure infrastructure monitoring — metrics, dashboards, alerts — can be done for close to nothing in software costs if you self-host, which is exactly why the hosting choice at the end of this guide matters more than most people expect.

## The four signals worth alerting on

Google's Site Reliability Engineering book popularized the "four golden signals," and they've survived a decade of tool churn for a simple reason: they work. If you can only track four things about a user-facing system, track these:

- **Latency** — how long requests take. Alert on the slow tail (p95/p99), not the average. A service with a 50ms average and a 4-second p99 is broken for roughly one user in a hundred.
- **Traffic** — requests per second, or whatever demand unit fits your system. This is context, not a problem indicator by itself, but it's how you tell "errors spiked because we're overloaded" from "errors spiked and traffic is flat."
- **Errors** — the rate of failed requests. Straightforward, and the one signal where alerting thresholds are easiest to get right.
- **Saturation** — how full your resources are. CPU at 97%, disk at 92%, a connection pool nearly exhausted. Saturation is the early-warning signal; it's the difference between "fix it this week" and "fix it right now."

Underneath those, every server you monitor should report the basics: CPU utilization and load, memory usage, disk space and disk I/O, and network throughput in both directions. The resource-focused way to organize this is the USE method — utilization, saturation, errors — applied to each resource. It's less famous than the golden signals but it's the better mental model for infrastructure specifically, because it forces you to check each resource for all three failure modes instead of just eyeballing a CPU graph.

One practical tip before moving on: alert on *symptoms* (latency, errors) at the user level, and alert on *causes* (saturation) at the resource level. A symptom alert means "users are suffering." A cause alert means "something will degrade soon." You need both, and treating them the same is how teams end up ignoring their own pager.

## Tool options: open source vs. paying per host

The monitoring tool market splits into three rough tiers, and the pricing differences are large enough to change your architecture.

**Full commercial platforms.** Datadog is the default answer in this category. Its Infrastructure Monitoring product is published at $15 per host per month on the annual Pro plan ($18 on-demand, $23 for Enterprise annual). APM adds $31+ per host per month, and log ingestion is billed per GB. For a 20-server environment, infrastructure monitoring alone is $300/month before you've touched traces or logs. New Relic and Dynatrace sit in the same bracket with different billing units. These platforms are genuinely good — polished dashboards, fast setup, integrations for everything — you're paying for breadth and someone else handling the plumbing.

**Cloud-native tools.** If you're already on AWS, Amazon CloudWatch is there by default. It's capable but priced à la carte: custom metrics run $0.30 per metric per month, and once you monitor a few dozen metrics across a fleet, plus logs and alarms, the monthly number climbs faster than most people expect. The Reddit r/aws channel has recurring threads with the same theme — CloudWatch costs discovered after the fact. CloudWatch is the right choice when everything already lives in AWS; it's an expensive choice when it's your only reason to stay.

**Self-hosted open source.** Prometheus plus Grafana is the standard pairing here, with Zabbix as the older alternative that some teams still prefer for traditional network monitoring. Grafana Cloud also offers a permanent free tier if you want the open-source workflow without running the servers — a reasonable middle ground, though the free tier has limits on series and access to certain plugins.

For infrastructure monitoring specifically — metrics, dashboards, alerts — the self-hosted route costs you an afternoon of setup and one VM. For a small fleet, that VM can be genuinely tiny. This is the point where hosting economics enter the picture, and it's the part most tool-comparison guides skip entirely.

## A working Prometheus + Grafana stack in an afternoon

Here's the shortest path to a real monitoring system, using the same components described in Prometheus's official node-exporter guide.

**Step 1: Install node_exporter on every server.** The node exporter is a small agent that exposes hardware and OS metrics (CPU, memory, disk, network, filesystems) on port 9100. It's a single binary with no dependencies, and Prometheus's documentation covers Linux, Docker, and other deployment styles. Drop it on each host you want to monitor.

**Step 2: Stand up Prometheus.** Prometheus pulls (scrapes) metrics from your exporters on a schedule. A minimal config looks like this:

yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "node"
    static_configs:
      - targets:
          - "10.0.0.11:9100"
          - "10.0.0.12:9100"
          - "monitor.internal:9100"


For anything beyond a handful of static hosts, you'd add service discovery, but a static target list is fine to start and is easier to reason about.

**Step 3: Write alert rules.** This is where the golden signals and USE method turn into actual notifications. Two starter rules:

yaml
groups:
  - name: infra
    rules:
      - alert: HighDiskUsage
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 85
        for: 10m
        labels:
          severity: warning
      - alert: InstanceDown
        expr: up == 0
        for: 5m
        labels:
          severity: critical


The `for:` clause is your noise filter — a condition must hold for 5 or 10 minutes before it pages anyone. Use it on almost everything except "the server is gone."

**Step 4: Point Alertmanager somewhere you'll actually see.** Alertmanager handles routing and de-duplication — critical alerts to a phone push, warnings to a chat channel. De-duplication matters more than routing polish; one incident triggering twenty alerts is how on-call rot gets ignored.

**Step 5: Install Grafana and import a dashboard.** Don't build dashboards from scratch. Grafana's public dashboard library has established node-exporter dashboards — import by ID and adjust. You'll have per-host CPU, memory, disk, and network panels in minutes.

The whole stack runs comfortably on a 2 vCPU / 4GB VM for a small fleet, and scales vertically as you add targets. Which brings us to the question this guide exists to answer: where does that VM live?

## The hosting layer most guides skip

A monitoring stack has three hosting characteristics that don't match how most people pick a VPS:

**It scrapes constantly.** Prometheus pulls from every exporter every 15 seconds. That's a small, steady stream of traffic — until you're monitoring dozens of hosts across regions, at which point egress fees at hyperscaler rates become a real line item. AWS-style pricing charges meaningfully per GB of outbound traffic; a monitoring mesh generates outbound traffic around the clock.

**It's a single point of failure you're building on purpose.** If the monitoring server dies silently, you don't just lose graphs — you lose your alerting while you need it most. Redundancy claims deserve scrutiny here.

**It attracts the same attacks as everything else.** Exposed dashboards and exporters are a common scan target, and volumetric DDoS traffic that would merely annoy a web server can take a small monitoring VM offline entirely.

This is where Sharktech's lineup becomes relevant, because their infrastructure is built around exactly these pressure points. The company has been around since 2003, operates its own network (AS46844, visible on peeringdb and bgp.tools if you want to verify the peering claims independently), and runs five points of presence — Los Angeles, Las Vegas, Denver, Chicago, and Amsterdam, in Equinix, CoreSite, H5, and CrownCastle facilities.

Three things stand out for a monitoring workload specifically:

- **Included DDoS protection.** Every hosted service includes their in-house mitigation at no extra cost — 60Gbps capacity on VPS and cloud plans, upgradeable to 100Gbps — filtering the common L3/L4 flood types (SYN floods, UDP floods, NTP/DNS amplification, and so on) before traffic reaches your VM.
- **Flat, generous bandwidth.** The Smart VPS line includes 4–304TB of transfer depending on size, and the public cloud includes 5TB outgoing with additional egress at $0.002/GB — ingress is free. Compared with hyperscaler egress rates, the constant chatter of a scraping mesh stops being something you budget for.
- **No vendor lock-in.** Their cloud is OpenStack-based (with documented Nova, Cinder, Swift, Neutron, and Keystone APIs), and you can export your disk images at any time — useful if your monitoring stack outgrows the provider or you want to migrate a snapshot elsewhere.

For balance: the public feedback is decent but not unanimous. HostAdvice rates Sharktech 9.3/10 overall, and their independent VPS benchmark measured 6,000+ random IOPS and sub-millisecond latency to major DNS resolvers. Trustpilot shows 3.5/5 from a small sample of 13 reviews, which skews toward extremes — a reminder that review samples this size tell you about support experiences, not statistical reliability. Their uptime commitments are specific: 99.999% platform availability for Smart VPS (triple-redundant Proxmox clusters), 99.99% guaranteed for dedicated servers.

## Sharktech plans and current pricing

The full current lineup from their order portal, with all service lines shown today:

| Plan | Core specs | Price (USD) | Billing | Get it |
| --- | --- | --- | --- | --- |
| Smart VPS | 2–128 vCPU, 4–256GB RAM, 40GB–2TB NVMe, 4–304TB transfer, 1Gbps port, 60Gbps DDoS, Xeon Gold, Proxmox | From **$7.95/mo** (as low as $3.98/mo annual) | Monthly; 25% off quarterly, 35% off semi-annual, 50% off annual | [Deploy a Smart VPS](https://bit.ly/SharKTech) |
| Public Cloud — Small | 4–16 vCPU, 8–32GB RAM, 300–2400GB SSD (+HDD/NVMe tiers), 20TB+ bandwidth, OpenStack | From **$39/mo** | Monthly, pay-as-you-go above commit | [Start Public Cloud Small](https://bit.ly/SharKTech) |
| Public Cloud — Medium | 8–32 vCPU, 16–64GB RAM, 800–6400GB SSD, 20TB+ bandwidth | From **$79/mo** | Monthly, pay-as-you-go above commit | [Start Public Cloud Medium](https://bit.ly/SharKTech) |
| Public Cloud — Large | 32–128 vCPU, 64–256GB RAM, 1500–12000GB SSD, 20TB+ bandwidth | From **$249/mo** | Monthly, pay-as-you-go above commit | [Start Public Cloud Large](https://bit.ly/SharKTech) |
| Public Cloud — Enterprise | 64+ vCPU, 128GB+ RAM, 5000GB+ SSD, uncapped resources | From **$499/mo** | Monthly, custom scaling | [Start Public Cloud Enterprise](https://bit.ly/SharKTech) |
| Dedicated Cloud | 8–512 vCPU, 16–1024GB RAM, SSD/HDD/NVMe tiers, 5–300TB transfer, fixed monthly billing | From **$86.23/mo** | Monthly, prepaid resources | [Get Dedicated Cloud](https://bit.ly/SharKTech) |
| Cloud Applications Platform | Managed platform, $0.0035/hr per cloudlet (400MHz/128MB), from 200GB storage scale | From **$5/mo** | Hourly consumption | [Try the Applications Platform](https://bit.ly/SharKTech) |
| Object Storage (S3) | 1TB–1PB, free inbound, free outbound up to 1TB, 5 locations | From **$6/mo** | Monthly | [Add S3 Storage](https://bit.ly/SharKTech) |
| Acronis Cloud Backup | Backup + cyber protection, from 200GB, file sync & share options | From **$4/mo** | Monthly | [Set Up Cloud Backup](https://bit.ly/SharKTech) |
| CDN — Basic | 5TB bandwidth, 5 hosts, $8/TB overage, 5 locations | From **$29/mo** | Monthly | [Add Basic CDN](https://bit.ly/SharKTech) |
| CDN — Advanced | 50TB bandwidth, 10 hosts, $0.0065/GB overage | From **$319/mo** | Monthly | [Add Advanced CDN](https://bit.ly/SharKTech) |
| CDN — Enterprise | 100TB bandwidth, 20 hosts, $0.0045/GB overage | From **$419/mo** | Monthly | [Add Enterprise CDN](https://bit.ly/SharKTech) |

They also sell bare-metal dedicated servers (fully customizable hardware, 1Gbps–40Gbps uplinks, hardware-level access, DDoS protection included) across all five locations, configured per order — the portal lists live inventory per city rather than a fixed price sheet, so you configure and see current pricing at order time. Colocation starts at $65/month for 1–6U depending on the city.

A few pricing details worth knowing before you click anything: Smart VPS gives you a **resource pool**, not a fixed VM — you can carve your allocation into as many virtual machines as the resources allow, in any mix of their data centers, which is a neat fit for placing a Prometheus scraper per region. Extra IPv4 addresses on cloud services run $1.50/month after the first free one. And their published rate card for pay-as-you-go cloud is per-hour and transparent: $0.0025/hr per vCPU, $0.0035/hr per GB RAM, $0.00009/hr per GB NVMe, with Public Cloud plans capped at a maximum resource ceiling so an accidental workload can't produce an accidental bill.

## Which plan fits which monitoring setup

Some concrete pairings, based purely on the numbers above:

**Monitoring 3–10 servers: Smart VPS, $7.95/month.** The entry Smart VPS tier (2 vCPU, 4GB RAM class) runs Prometheus, Alertmanager, and Grafana with room to spare for a small target list. Pay annually and it drops to $3.98/month — roughly $48/year for infrastructure monitoring of your entire fleet, which is less than one month of a commercial platform covering three hosts. At that price the open-source-vs-Datadog question answers itself for small operations.

**Monitoring 10–50 servers or multiple regions: Public Cloud Small, $39/month.** More headroom, plus OpenStack's networking features (private networks, security groups, load balancers) let you isolate the monitoring subnet from public exposure — exporters talk on the private network, only Grafana faces the internet. The 5TB outgoing allowance covers a lot of scraping; beyond that it's $0.002/GB, so even a heavy mesh stays cheap. The cloud portal includes built-in traffic monitoring and analytics, which covers the "monitoring the monitor" problem partially at the platform layer.

**Fleet-scale or compliance-heavy setups: Dedicated Cloud or Public Cloud Large+.** Fixed monthly billing with committed resources (Dedicated Cloud from $86.23/month) makes budgeting predictable, and the multi-tier storage options let you put your Prometheus time-series database on NVMe (rated around 1.2GB/s and 18,000 IOPS per volume) while long-horizon historical data sits on cheaper HDD storage (120MB/s). That tiering is genuinely useful for metrics workloads, where recent data is hot and old data is rarely read.

**Whatever you pick, add the $4/month Acronis backup.** A monitoring server with no backup is a fun irony. Two hundred gigabytes of protected storage covers your Prometheus data directory and Grafana config with change to spare.

If you're unsure, start small: 👉 [deploy a Smart VPS](https://bit.ly/SharKTech), get the stack running, and scale up through the portal when the resource graphs (which you'll now have) tell you to.

## Alerting hygiene: the part that decides whether this works

The stack is the easy half. The hard half is keeping alerts trustworthy, because an alerting system nobody believes is worse than none — it manufactures false confidence.

A few rules that hold up in practice:

1. **Every alert needs an owner and an action.** If nobody knows what to do when it fires, it's a dashboard annotation, not an alert. Delete it or demote it.
2. **Use the `for:` duration on nearly everything.** Five to ten minutes of sustained condition before paging filters out the blips that self-resolve — and most blips self-resolve.
3. **Alert on symptoms for pages, causes for tickets.** Latency and error-rate spikes wake someone up. Disk at 85% creates a to-do item.
4. **Watch saturation thresholds, not just binary states.** "Disk full" is too late; "disk growth will fill in 7 days at current rate" is a prediction you can act on.
5. **Monitor the monitoring.** Your Prometheus instance needs an up-check that runs from somewhere else — a different VM, ideally a different provider or an external uptime service. The classic failure is the monitor dying quietly and everyone's dashboards freezing at a green, happy moment in history.

And keep an eye on cardinality. Every unique label combination on a metric multiplies the series Prometheus stores. Tagging metrics with per-request IDs or unbounded user IDs will eventually consume memory and disk in ways that feel mysterious until you count your active series. Fixed label sets, bounded cardinality, no exceptions.

## Common questions

**Is cloud infrastructure monitoring the same as observability?** Not quite. Monitoring tells you *that* something is wrong; observability tooling (distributed tracing, high-cardinality logs, exemplars) helps you figure out *why*. For teams under ~50 services, well-built monitoring catches most incidents. Observability becomes worth its cost as systems get more distributed and the "why" questions get harder.

**Can I skip commercial tools entirely?** Yes, and many teams do — Prometheus and Grafana run some of the largest monitoring deployments in existence. What you give up is the managed integration catalog and someone else's on-call for the monitoring platform itself. For a $15–23/host/month budget you're buying convenience; whether that's worth it depends on how much engineering time costs you.

**How much retention do I need?** 15-second resolution for 2 weeks catches most debugging needs; 5-minute or 1-hour resolution downsampled for 6–13 months answers capacity-planning questions. Prometheus handles the short window natively; longer horizons are what Thanos, Cortex, or Mimir are for — or simply accept shorter retention and export monthly summaries.

**Where should the monitoring server live relative to what it monitors?** Same provider and region when possible (latency and egress), but never the same *machine*, and ideally not the same failure domain. If everything runs in one cloud, at least put the monitor in a separate availability zone — or on separate infrastructure entirely, which is a legitimate argument for running Prometheus on an independent provider like Sharktech while your production fleet lives elsewhere.

## The short version

Cloud infrastructure monitoring comes down to four signals (latency, traffic, errors, saturation), a stack you can assemble from Prometheus and Grafana in an afternoon, and a hosting decision that determines your real costs. The tool debate gets the attention, but the hosting math is quieter and more persistent — egress fees and per-host subscriptions are what turn a $0 open-source stack into a four-figure monthly observability bill.

A small fleet's complete monitoring system fits on a $7.95/month VPS with DDoS protection and flat bandwidth included, and scales from there through OpenStack tiers if the fleet grows. If that matches your situation, 👉 [check Sharktech's current plans and pricing](https://bit.ly/SharKTech) and start with the smallest thing that works — the graphs you build will tell you when it's time for the next size up.
