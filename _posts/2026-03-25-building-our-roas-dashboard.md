---
layout: post
title: "We Built Our Own ROAS Dashboard. Here Is How and Why."
date: 2026-03-25
description: "No analytics tool was connecting our ad spend to subscription revenue the way we needed. So we built one. Here is what we learned."
author: Arnaud Aubry
---

### The problem no analytics tool was solving for us

After a few weeks running paid campaigns across Meta and TikTok for our apps, we had a practical problem. We could see our spend in the ad platforms. We could see our subscription revenue in RevenueCat. But there was no tool connecting the two in the way we actually needed.

The core issue is timing. Our apps use 7-day free trials. A user installs on Monday, starts a trial the same day, and we will not know whether they paid until the following Monday at the earliest. We are spending money continuously on campaigns, and for any given week of spend, we are blind for at least seven days. We cannot calculate ROAS on the current week because the revenue is not in yet.

AppsFlyer gives us attribution data, but its ROAS reporting blends campaign-level spend with aggregate subscription revenue in ways that are hard to reconcile with the specific cohort logic we needed. RevenueCat gives us subscription data but no spend context. Neither tool gave us the core view we wanted: for every week of users we acquired and paid to acquire, what is the total expected revenue, and how does it compare to what we spent?

So we built it ourselves.


### What the dashboard actually does

The core view is a grid of weekly cohorts. Users are grouped by the week they first opened the app. For each cohort, the dashboard shows the spend that week across Meta and TikTok, the number of users acquired, the trial funnel (how many started a trial, how many converted to paid, how many churned), the revenue generated so far, and a predicted final ROAS.

The predicted ROAS is the part that required the most thinking. For recent cohorts where trials are still active, actual ROAS is meaningless because revenue is incomplete. We built a two-pass prediction model.

Pass one: identify the last four cohorts that have fully settled, meaning all trials have either converted to paid or churned. From these settled cohorts, extract the average trial-to-paid conversion rate and the average net revenue per subscription type.

Pass two: for any cohort with active trials, apply the benchmark conversion rate to estimate future paid subscribers, multiply by the average revenue per subscriber, add the revenue already collected, and divide by total spend.

The result is a predicted ROAS you can read within days of a campaign launching. It is not a guarantee. We estimate it is right about 80 percent of the time on direction (above or below 1.0x). The main source of error is conversion rate volatility week to week. But having a directional signal on day three instead of day ten changes how quickly you can act.


### The LTV problem for weekly subscribers

Annual subscriptions are simple to model in a 25-week window. One payment, no renewals within the window, LTV multiplier of 1.0. Weekly subscribers are more complex.

A weekly subscriber who converted in week two of a cohort has already paid roughly 1.9 times at week two: one initial payment plus roughly 90 percent retention at week one. Their expected total payments over 25 weeks is around 4.06 times the unit price, based on the industry-standard mid-tier retention curve. The expected future payments from that subscriber are 4.06 minus 1.9, so about 2.16 more payments.

The dashboard models this automatically for each cohort. As time passes, the alreadyPaid value for weekly subscribers increases and the predicted future revenue from those subscribers decreases proportionally. When you look at a settled cohort, the predicted and actual revenue should converge. When they do not, it means your real retention is diverging from the model curve, which is exactly the signal you want.

One bug we hit early: we were attributing revenue to subscriber types by headcount proportion. If a cohort had 35 yearly subscribers and 5 weekly subscribers, the weekly subscribers were getting credited with 5/40 of total revenue. But yearly subscribers pay 15 to 20 times more per initial payment than weekly subscribers. The distortion was massive. A weekly subscriber was being modeled with a unit price of €7 when the actual product price was €2.09. We fixed this by tracking revenue per subscription type directly from RevenueCat's transaction data, which gives us the actual proceeds per customer broken down by product.


### How it is built

The stack is a Node.js Express server with a Vue 3 frontend, deployed on an AWS EC2 instance behind nginx with Let's Encrypt for SSL. The whole thing runs under PM2 with systemd auto-restart on reboot.

The data comes from three API sources. RevenueCat for subscription revenue and trial funnel data. Meta's Marketing API for spend by date range. TikTok's Business API for spend on the other side. Currency conversion runs through an open exchange rates endpoint, since RevenueCat reports in USD and our campaigns run in EUR.

The RevenueCat integration was the most technically involved part. The v2 API does not return an expires_date field, which is what most documentation references. The correct field for determining whether a trial or subscription is still active is current_period_ends_at. Getting this wrong produces phantom active subscribers that never resolve. We spent a few sessions debugging cohorts where the trial count kept showing one active trial that would not settle before realizing the field reference was wrong.

Product type detection was another non-obvious problem. RevenueCat does not expose the subscription product type directly in a reliable way. We detect it from the billing period duration: anything under 14 days is weekly, 15 to 60 days is monthly, up to 400 days is yearly. This turns out to be a stable heuristic across all the products we have tested.

The TikTok API requires date ranges capped at 30 days per request and will not accept array parameters unless they are JSON-stringified in query parameters. The Meta API has a 25k events per request limit. Neither of these are documented clearly. We hit both in production.

Cohort cache is disk-backed to survive server restarts. Each cohort key includes the project, date range, and a version prefix so we can invalidate everything cleanly on model changes. Revenue data per customer is also cached with a six-hour TTL to avoid hammering the RevenueCat API on every page load. A hard refresh from the UI clears all caches and re-fetches from APIs.

Authentication is handled via magic links through Resend. Enter an email on the login page, receive a one-time link valid for 15 minutes, and get a 30-day session cookie. The whitelist is just an array in the server config.


### What it tells us that nothing else did

The most useful thing the dashboard surfaces is the relationship between spend weeks and revenue weeks. When you look at a cohort that is six weeks old, you can see exactly how much of the eventual LTV has already materialized and how much is still pending.

Here is a concrete example. The Mar 9 to Mar 15 cohort for Provenance has 34 yearly subscribers, 3 monthly, and 7 weekly. At 16 days old, the yearly subscribers have paid once and will not renew within our 25-week window. The 7 weekly subscribers are at 2.3 weeks and have around 2.16 expected payments remaining. Total predicted future revenue from the weekly renewal tail: roughly €32. Small in absolute terms, but it would be missed entirely by a model that only looks at initial subscription events.

The bigger picture: that cohort spent €2,174 across Meta and TikTok. Predicted total revenue is €605. Predicted ROAS is 0.28x. The campaigns were not optimized yet that week. The Mar 16 to Mar 22 cohort on the same app spent €2,634 and is already trending toward better conversion as the trial campaign matured. That week-by-week delta is exactly what we built this to see.


### What we would do differently

A few things we would change with the benefit of hindsight.

We would have added per-type revenue tracking from RevenueCat from day one. We added it later after discovering the headcount attribution bug. The correct data was always available in the RevenueCat API. We just were not collecting it at the aggregation level.

We would have built the cohort cache invalidation strategy earlier. We had a period where cached cohort data survived model changes and the UI was showing stale predictions. Versioning the cache keys with a model version prefix made this trivial to handle but we did it after the first few debugging sessions rather than before.

We would have added proper error logging sooner. RevenueCat returns 429 rate limit errors occasionally, especially during hard refreshes. Early on, error responses were being cached alongside successful ones, which silently poisoned the data. We fixed this by only writing to cache on successful responses, but the corruption was subtle to detect.

The dashboard is not a polished product. It is working infrastructure built for a specific purpose. But it does the job better than any off-the-shelf tool we evaluated for our situation, and building it cost about two weekends of engineering time using modern AI coding tools.
