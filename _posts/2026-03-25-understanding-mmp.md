---
layout: post
title: "Understanding MMPs: Do You Actually Need One?"
date: 2026-03-25
description: "Mobile Measurement Partners are sold as essential infrastructure for paid acquisition. The honest answer is more nuanced. Here is when they earn their cost and when they do not."
author: Arnaud Aubry
---

### Information is the real currency of paid acquisition

In a [previous post](/blog/paid-acquisition-mobile-apps/), we walked through how paid acquisition works for mobile apps: from first install campaigns to trial optimization and cohort monitoring. The through-line of all of it is information. Your campaigns only get better if the algorithm understands what happened. Your decisions only get sharper if you understand what is working.

There are two distinct information flows in paid UA, and they are often confused.

The first is feedback to the platforms. Telling Meta and TikTok what happened after someone saw your ad. Did they install? Did they start a trial? Did they pay? Without this signal, the algorithm is optimizing blindly. It is making targeting decisions without knowing whether its choices led to anything valuable. This is the fastest way to burn money on paid ads.

The second is your own understanding: attribution, ROAS measurement, separating paid from organic traffic, knowing which campaign drove which result. This is what lets you make decisions.

A Mobile Measurement Partner, or MMP, lives at the intersection of both. It is a third-party SDK you integrate into your app that collects user events, attributes them to specific campaigns and ad networks, and relays the data back to both your advertising platforms and your own analytics. AppsFlyer, Adjust, Branch, Singular, and Kochava are the main players. Prices and positioning vary, but they all solve the same core problem.


### Why feeding the algorithm matters more than most people realize

When you run a campaign, the platform algorithm is watching everything. Which users it showed your ad to, which users clicked, which installed. What it cannot see by default is what happened after the install. Did those users start a trial? Did they pay?

This is where the feedback loop breaks down without proper event tracking. The algorithm has no idea whether the people it sent to your app were actually valuable. So it keeps optimizing for what it can measure, which is often just the install, regardless of downstream quality.

The fix is pushing events back to the platform. You tell Meta: this user started a trial. You tell TikTok: this user subscribed. The algorithm can now optimize for outcomes that actually matter to you, not just raw install volume. The impact of this on campaign efficiency is not marginal. It is the difference between a campaign that learns and one that runs in circles.

In practice, managing this event pipeline is harder than it sounds. Every platform has its own SDK, its own event naming conventions, its own integration requirements. When you have events firing from the app directly to Meta and TikTok simultaneously, you accumulate surface area for bugs. An SDK update, a mis-mapped event, a tracking permission edge case: suddenly your campaigns are running on bad signal without you realizing it. We have been there.

The main practical benefit of an MMP is centralizing this. One SDK in the app, one source of truth, routing to all platforms. And because the MMP handles the routing on the server side, you can update your attribution configuration without pushing a new app version. No App Store review cycle. No delay between when you want to fix something and when it is actually fixed. For anyone who has gone through an emergency fix waiting on Apple review, this matters.


### What attribution actually tells you

Attribution is the process of connecting a downstream event to its upstream cause. A user installs your app on Thursday. They start a trial on Friday. They pay on the following Monday. Attribution answers: which ad did this user see? Which campaign, which creative, which platform?

Without attribution, you can see that you spent money and got some subscribers, but you cannot tell which spending produced which subscribers. You cannot measure ROAS per campaign. You cannot compare Meta against TikTok on equal terms. You cannot tell whether the users coming from one campaign retain better than another.

With attribution, you can build a clean picture: campaign A cost you €12 per trial start and converted at 55%, campaign B cost €9 per trial start and converted at 30%. These are very different situations that look identical in aggregate if you are not attributing correctly.

Attribution also separates organic from paid, which is genuinely important and often underestimated. If your app has any organic traction through search, word of mouth, or editorial features, that traffic will blend into your paid numbers without attribution. Your ROAS calculation will look better than it is. You will be attributing organic revenue to paid campaigns and drawing the wrong conclusions about what is working.

That said, be honest about the practical limits. Attribution today operates in an increasingly constrained environment. Apple's App Tracking Transparency reduces the signal available on iOS. SKAdNetwork, Apple's privacy-preserving attribution framework, introduces delays and data loss. No MMP solves this entirely. They work around it, model probabilistically, and give you the best available picture. The numbers will never be perfect, and you should factor that uncertainty into your decisions.


### When an MMP is overkill, and when it becomes essential

This is the part most MMP vendors will not tell you. For a solo developer running a single app on a single platform with a modest paid budget, an MMP is probably not worth it.

Here is the honest cost structure: most MMP pricing is event-based, and if your app uses trials and subscriptions, renewal events count against your quota. Plans get expensive quickly once you have any real subscription volume. Several thousand euros per month is not unusual at moderate scale. For a single app doing its first €3,600 per month of paid spend, that overhead on an investment that might only be marginally profitable is hard to justify.

For a single app on a single ad network, native SDK integrations from Meta and TikTok directly, combined with server-to-server events from RevenueCat, can get you most of the signal you need without MMP fees. It is more fragile and takes more maintenance, but it works.

The calculus changes with complexity. If you are running two apps across Meta and TikTok simultaneously, you are managing four SDK integrations and four attribution streams. Attribution becomes genuinely hard to keep clean. An MMP starts earning its fee through saved engineering time and reduced error surface.

Add a second operating system, Android alongside iOS, and attribution becomes significantly more complex given the different platform rules and signal constraints. Add a third ad network. Add regional campaigns with different targeting. At that point, an MMP is not a luxury. It is the only realistic way to keep your data clean and your feedback loops functioning.

Our current rule of thumb: one app, one platform — probably not worth it. Two or more apps, two or more platforms: the math starts working. Portfolio approach with multiple operating systems: essentially required.


### Server-to-server tracking, and why it beats in-app events

Whatever path you take, one principle applies universally: prefer server-to-server event tracking over in-app events whenever possible.

In-app events fire from the user's device. They are subject to network conditions, background execution limits, and increasingly aggressive privacy restrictions. More importantly, subscription renewals happen server-side. When a user's annual subscription renews twelve months after they first subscribed, that renewal happens between Apple's servers and your RevenueCat account. There is no app launch. There is no in-app event to capture. If your tracking relies entirely on in-app instrumentation, you will miss all renewal revenue.

Server-to-server tracking catches renewals because it sits at the infrastructure level. RevenueCat fires webhooks on every subscription event: new subscriptions, renewals, cancellations, refunds. These fire reliably regardless of whether the user has opened the app. For a business built on annual subscriptions where a significant share of long-term revenue comes from renewals, this is not a minor detail.

Apple also provides a transaction webhook you can configure per app, which delivers a cryptographically signed confirmation of every purchase event. This gives you an absolute source of truth on payment events that does not depend on any third-party SDK chain. Worth integrating for any serious monetized app.


### What we actually use

We use AppsFlyer. It works well for our current setup: two iOS apps, Meta and TikTok on both, one team managing campaigns across both.

Is it irreplaceable? Probably not. The attribution layer is genuinely useful, but the analytics dashboard it provides overlaps significantly with what we built ourselves. Our cohort ROAS dashboard gives us per-week spend attribution, predicted revenue based on trial conversion rates, and LTV modeling across subscription types. These are features that approximate a good chunk of what AppsFlyer's analytics layer offers, built specifically around how our apps monetize.

We are currently evaluating whether to stay on the free plan, move to a paid plan as volume grows, or eventually replace the attribution dependency with a more custom server-to-server setup using RevenueCat webhooks as the primary event source. The right answer depends on how the portfolio evolves. With vibe coding making custom tooling increasingly accessible, replacing parts of the paid analytics stack is a realistic option that was not a few years ago.


### The takeaway

An MMP is not a prerequisite for paid acquisition. It is a solution to a complexity problem, and complexity has to be real before the solution makes sense.

Start without one if you are single app, single platform. Get your Meta and TikTok SDK integrations right, push events server-to-server through RevenueCat, and build whatever dashboard visibility you need. The marginal benefit of adding an MMP is low when you can see and manage everything yourself.

Introduce one when complexity requires it. Two platforms, two apps, multiple people touching campaign configuration: that is when centralized event management and clean attribution earn their cost.

Regardless of what you use: prioritize server-to-server. Events that fire without user action, at the infrastructure level, are the only reliable way to capture the full revenue picture from a subscription business.

---

*Arnaud Aubry is building mobile apps and sharing what he learns.*
