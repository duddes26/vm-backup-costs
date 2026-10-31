# backup virtual machines: How to Build a VM Backup Setup That Actually Restores, With Real Costs From $7.95/Mo

Backing up virtual machines comes down to two decisions: how you capture consistent copies of your VMs, and where those copies live. Most guides you'll find cover the first half in detail and then get vague about the second half, which is exactly where budgets blow up — cloud storage and egress fees can cost more than the VMs themselves. This article covers both: the actual mechanics of VM backups, why snapshots alone will fail you, and a concrete setup using Sharktech's infrastructure, where a VPS starts at $7.95/month and S3-compatible backup storage runs $4.90 per TB with no egress gotchas.

> **Quick picks for impatient readers**
> - Run your VMs on **SmartVPS** (from **$7.95/mo**, 2 vCPU / 4 GB RAM / 40 GiB NVMe / 4TB transfer) or **Public Cloud** (Small tier from **$39/mo**).
> - Push backups to **S3 Object Storage** at **$4.90/TB/mo**, first 1TB of outbound transfer free each month.
> - 👉 [Check Sharktech's current plans and pricing](https://bit.ly/SharKtech)

## What backing up virtual machines actually involves

A VM is just a set of files: a virtual disk image (VMDK on VMware, VHD/VHDX on Hyper-V), a config file describing its CPU, memory and network settings, plus snapshots and logs. A complete backup captures all of it. Copying only the disk file means rebuilding the VM by hand and reattaching the disk when disaster hits.

There are three ways to get that copy, and they behave very differently:

1. **File-level copy.** Copy the disk files off the datastore. Simple, works even on ESXi's free tier, but you need to shut the VM down for consistency, and restoring a single file means restoring the whole VM first.
2. **Agent-based backup.** Install a backup agent inside the guest OS. Best application consistency for SQL Server, Exchange, Active Directory — and the only option for fault-tolerant VMs, which VMware can't snapshot at all. The cost is operational: one agent per VM to install, update and troubleshoot.
3. **Agentless, host-level backup.** The backup tool talks to the hypervisor directly through VMware's VADP APIs, takes a snapshot, and reads the disk data without touching the guest. This is the standard approach at scale, and it's what tools from vendors like NAKIVO, Datto SIRIS and Unitrends are built around. One caveat: it requires a licensed ESXi host, so the free hypervisor is off the table.

Modern tools make the transfer incremental using changed-block tracking — CBT on VMware, RCT on Hyper-V — so after the first full backup, only the blocks that changed get sent. CBT has prerequisites worth knowing: ESXi 4.0 or later, VM hardware version 7 or later, and CBT enabled on the VM (it is by default; the `ctkEnabled` parameter in the VM's settings should read True). It can also silently reset after storage migrations or some vMotion operations, which turns your next "incremental" into a full backup. If a backup job suddenly moves gigabytes when it normally moves megabytes, that's usually why.

The target those backups land on can be a local appliance, a NAS, or an offsite S3-compatible repository. The last option is where Sharktech fits in, and we'll get to the numbers shortly.

## Hypervisor snapshots are not backups

If you take one idea from this article, take this one. A VMware or Hyper-V snapshot is a delta file living on the same datastore as your production VM. A storage failure, datastore corruption, or ransomware that encrypts your datastore takes out the VM and its snapshots in one stroke. Snapshots also degrade performance the longer they stick around — VMware's own guidance is to keep no more than 2–3 per VM and retain them for 24–72 hours at most.

Backups, done properly, live somewhere else entirely. Purpose-built tools take the snapshot, pull the data off to a separate repository, then delete the snapshot immediately. That offsite copy is your actual backup; the snapshot is just a momentary handling mechanism.

This is also why object storage with versioning and object locking matters. Sharktech's S3 storage supports object versioning plus Object Lock, which makes stored backups immutable — a ransomware payload that compromises your backup server can't delete or encrypt what's already in the bucket. Vendors like Datto market this as "immutable cloud storage" as a premium feature; here it's part of a plan that costs less per terabyte than a coffee.

## Sharktech plans at a glance

Sharktech has been running infrastructure since 2003, with five data center locations — Los Angeles, Las Vegas, Denver, Chicago and Amsterdam — and DDoS protection baked into the network. For the purposes of backing up virtual machines, three product lines matter: hosting the VMs themselves, isolating them on dedicated hardware if needed, and storing the backups.

| Product | Core specs | Starting price | Billing | Link |
| --- | --- | --- | --- | --- |
| SmartVPS | 2 vCPU / 4 GB RAM / 40 GiB NVMe / 4TB transfer / 1 IPv4, scalable to 128 vCPU / 256 GB | $7.95/mo | Monthly; 25% off quarterly, 35% off semi-annual, 50% off annual | [ Order SmartVPS](https://bit.ly/SharKtech) |
| Public Cloud (OpenStack) | Small tier from $39/mo; NVMe/SSD/HDD storage tiers; Linux or Windows VMs | $39.00/mo | Monthly base + metered overage | [ Compare Public Cloud plans](https://bit.ly/SharKtech) |
| Dedicated Cloud | Fixed resource allocation on the same OpenStack platform, no automatic overages | Custom quote | Fixed price for your chosen allocation | [ Request a Dedicated Cloud quote](https://bit.ly/SharKtech) |
| Private Cloud | Dedicated physical infrastructure, single-tenant | Custom quote | Contact sales | [ Discuss a Private Cloud build](https://bit.ly/SharKtech) |
| S3 Object Storage | 1TB / 10TB / 100TB buckets, S3-compatible API, Object Lock | $4.90 per TB/mo | Monthly; first 1TB egress free, then $0.90/TB | [ See S3 storage plans](https://bit.ly/SharKtech) |

A few details that don't fit in a table but will affect your bill:

- **SmartVPS annual billing halves the price.** Quarterly, semi-annual and annual cycles get automatic discounts of 25%, 35% and 50% respectively, applied with no coupon code — so the $7.95/month starting plan works out to roughly $3.98/month paid yearly, about $47.76 for the year instead of $95.40. Since a backup utility VM runs 24/7 anyway, there's little reason to pay monthly.
- **One SmartVPS resource pool, many VMs, five locations.** You buy a pool of resources and carve it into one or more VMs, choosing a location per VM. You can spread VMs across Los Angeles, Denver, Chicago, Las Vegas and Amsterdam — useful if you want backup copies landing in a different city from production.
- **Public Cloud resource ceilings aren't spending caps.** The Small, Medium and Large tiers have resource ceilings, and Sharktech explicitly notes these are not monetary spending caps — usage above your included allocation is billed at published hourly rates. If a runaway process spikes your usage, you pay for it. If you want a bill you can predict to the dollar, Dedicated Cloud gives you a fixed allocation at a fixed price with no automatic overages.
- **S3 storage has a sales threshold.** 1TB, 10TB and 100TB are self-serve; beyond 100TB of storage or 300TB of monthly transfer, it's a conversation with the sales team. Plan your retention window accordingly before ordering.

For hosting the VMs you're protecting, HostAdvice's benchmark review put it this way: "With 6,000+ random IOPS, sub-millisecond network latency … Sharktech delivers genuine power." And one customer who migrated from AWS and Azure to Sharktech's Dedicated Cloud described the pricing as "shockingly reasonable" in a review displayed on the company's site — the kind of comparison that only means something coming from someone who paid both bills.

## Using S3 Object Storage as your backup target

Sharktech's S3 Object Storage is built for exactly this job: large, rarely-read datasets. It pairs high-capacity HDDs for capacity with an NVMe cache layer for the active data path, exposed through a standard S3-compatible API. The company is refreshingly upfront about what it's not for — boot volumes and low-latency block storage workloads should look elsewhere. Backup repositories are the ideal use case: write-heavy, read-rarely, and cost-sensitive.

Getting started takes minutes because it's plain S3:

1. Order a bucket — 1TB at $4.90/mo, 10TB at $49/mo, 100TB at $490/mo. Beyond 100TB of storage or 300TB of monthly transfer, contact sales.
2. Point your backup tool at it. Rclone, Cyberduck, Nextcloud and S3 Browser are officially supported, or connect your existing S3-compatible backup software directly through the API. If your backup software can write to any S3 endpoint, it can write to this one.
3. Enable versioning and Object Lock on the bucket so your restore points can't be tampered with.
4. Set lifecycle rules to expire old backups automatically — daily incrementals kept 30 days, weekly fulls kept 90, whatever your retention policy says.

If you need a hands-on walkthrough of the VMware side itself — configuring hypervisors, tag mappings, IAM roles — the official AWS Backup documentation covers the mechanics in detail, and the same concepts (snapshots, CBT, retention) apply whichever backup tool you use.

## What your backup storage really costs

This is where the math gets interesting, and where most comparison articles stop short. Let's price a realistic setup: a handful of VMs producing about 5TB of backup data, with a disaster recovery where you restore 4TB.

**Sharktech:** 5TB of S3 storage runs **$24.50/month**. That disaster recovery restore? The first 1TB of outbound transfer each month is free, so 4TB costs you 3TB at $0.90/TB — **$2.70**. Total first-year cost: about $296.70.

**AWS S3 Standard**, listed at $0.023 per GB, runs about **$118/month** for the same 5TB. And the restore is where the pain arrives: egress runs on the order of $90 per TB, so pulling 4TB back out costs roughly **$360** — for a single recovery event. First-year cost, without even one disaster: around $1,416.

That's roughly a 4–5x difference on storage and two orders of magnitude on egress. For backup workloads, egress pricing matters more than storage pricing, because the whole point of backups is that you'll eventually pull large amounts of data out. Sharktech's flat $4.90/TB with included transfer is the whole reason its own object storage page leads with "without the big-cloud bill."

## Which setup fits your situation

A few honest rules of thumb:

- **Testing backups or protecting one or two low-traffic VMs?** The $7.95 SmartVPS starting allocation is plenty, and annual billing brings it to roughly $3.98/month. 👉 [Start a SmartVPS at the annual rate](https://bit.ly/SharKtech)
- **Running a proper backup repository of several terabytes?** The 1TB or 10TB S3 plan plus Rclone (or your existing backup software pointed at it) covers it. You can scale to 100TB before you ever need to talk to sales.
- **Predictable production workloads with a strict budget?** Dedicated Cloud's fixed allocation beats metered Public Cloud billing for peace of mind. If you need single-tenant physical infrastructure under your backup environment, Private Cloud is the quote-based option.
- **Already married to a big-cloud provider for other reasons?** Fair enough — AWS Backup's VMware integration is solid, and Datto SIRIS or Unitrends make sense if you're an MSP or want appliance-style protection. Just go in knowing what the storage and egress line items will look like at restore time.

One thing to be clear about: Sharktech provides the infrastructure — the VMs, the storage, the network — and supports it 24/7. Its product line doesn't include managed backup software, so you'll bring your own backup tool. For most technical readers that's exactly what they wanted anyway, since it means their existing jobs and retention policies just work with any S3-compatible endpoint.

## FAQ

**Can I just rely on hypervisor snapshots?**
No. Snapshots sit on the same datastore as the VM and share its fate. Keep them under 2–3 per VM and 24–72 hours, and treat them as a short-term mechanism, never as your backup.

**Does Sharktech charge extra for API requests on object storage?**
No. Pricing is storage plus transfer — $4.90 per TB monthly, first 1TB of outbound transfer free, $0.90/TB beyond that. There are no per-request or per-object fees to rediscover on your invoice.

**Can my VMs and backups live in different locations?**
Yes. SmartVPS lets you pick a location per VM — Los Angeles, Denver, Chicago, Las Vegas or Amsterdam — so production and backup copies can sit in different cities.

**What about Windows VMs?**
Both SmartVPS and Public Cloud run Linux or Windows, and you can upload your own VM images or ISOs on the OpenStack platform.

## Before you order

Match the plan to the job: a single utility VM or test environment → SmartVPS at the annual rate; a multi-terabyte backup repository → the S3 plan sized to your retention window, with Object Lock switched on from day one; production VMs with a strict budget → Dedicated Cloud; single-tenant hardware requirements → a Private Cloud quote. And before committing to terabyte-scale storage anywhere, price out one full disaster recovery restore — that number, more than the monthly storage fee, is what separates cheap backups from affordable ones. 👉 [Check current Sharktech plans and pricing](https://bit.ly/SharKtech)
