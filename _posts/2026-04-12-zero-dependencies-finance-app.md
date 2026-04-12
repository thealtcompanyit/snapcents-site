---
layout: blog
title: "Zero Dependencies: Building a Finance App with Only Apple Frameworks"
date: 2026-04-12
description: "How SnapCents handles receipt scanning, on-device AI, bank statement parsing, and budgeting with zero third-party libraries."
reading_time: "9 min read"
---

SnapCents has zero third-party dependencies. No SPM packages. No CocoaPods. No Carthage. The entire app, from receipt scanning to on-device AI to bank statement parsing, runs on Apple frameworks alone.

![SnapCents home screen showing expense tracking dashboard](IMAGE: screenshot-home.png)

This was a deliberate choice, not an accident. When your app handles people's financial data, every dependency is a liability. Every third-party SDK is code you didn't write, can't fully audit, and might phone home with data you promised to protect. The supply chain attacks on popular open-source packages over the last few years made this an easy call. Fewer dependencies also means a smaller binary, faster builds, and one less thing to break when Xcode updates.

The stack: SwiftUI and SwiftData for the UI and persistence, Vision and Foundation Models for OCR and AI, StoreKit 2 for monetization, WidgetKit for widgets, NaturalLanguage for intent classification, PDFKit for statement import, CloudKit for sync, TipKit for feature discovery, App Intents for Shortcuts. That covers receipt scanning, budgeting, AI chat, PDF export, bank statement parsing, and everything in between.

The tradeoff is real, though. You build more things yourself. There's no Alamofire, no Kingfisher, no Realm. Every persistence layer uses what Apple ships. Sometimes that means writing 50 extra lines. Sometimes it means fighting an API that doesn't quite fit. But you know exactly what every line of your app does with user data, and that matters when you're asking people to trust you with their finances.

## Receipt OCR: Three Tries to Get It Right

The OCR pipeline went through three major rewrites before landing on something reliable.

The first attempt used zone-based parsing. The idea was simple: totals are usually at the bottom of a receipt, so scan the bottom third of the image. This works great on the standard thermal receipt from a grocery store. It falls apart on mall receipts with multiple stores, restaurant receipts with tips below the total, and any receipt where the layout deviates from the assumed structure. Real receipts vary wildly. Zone-based parsing was scrapped.

The second attempt used a two-pass strategy: a fast recognition pass to handle "easy" receipts, then an accurate pass for everything else. In practice, the fast pass almost never succeeded. It just added latency to the common case. One pass with accurate recognition turned out to be both faster and simpler.

![SnapCents receipt scanning showing extracted merchant, amount, and items](IMAGE: screenshot of receipt scan result screen)

The current pipeline is a multi-tier fallback. The top tier uses Apple's structured document parsing API for receipts with clear formatting. The second tier runs text recognition with pattern matching and a proximity scorer that measures how close keywords like "total" appear to dollar amounts on the page. The third tier uses the on-device LLM as a last resort for receipts that resist structured parsing.

One detail that took experimentation: image sizing before OCR matters a lot. Too small and you lose fine print. Too large and you add latency without improving accuracy. Finding the right balance took several rounds of testing across different receipt types.

## On-Device AI That Actually Works

Apple's Foundation Models framework ships an on-device model starting with iOS 26. SnapCents uses it for a spending assistant that can answer questions like "how much did I spend eating out this month?" by querying your actual spending data.

Getting it to work reliably required learning constraints that aren't well documented yet. The combined token budget is tight, so every word in your tool descriptions costs you response capacity. The system prompt had to be compressed significantly, with formatting instructions placed early because the model pays more attention to what comes first.

![SnapCents AI chat showing spending analysis response](IMAGE: screenshot-insights.png or AI chat screenshot)

The most useful pattern I found: using constrained enums for model outputs. When the model generates against a defined set of cases, it physically cannot produce values outside that set. This is constrained decoding at the token level, not post-processing validation. For a finance app where hallucinated categories or time ranges could produce wrong answers, this was essential.

One pitfall that cost hours of debugging: structured output generation conflicts with tool calling. You can't use both at the same time. If your session uses tools, get your structured data back through the tool's return value instead.

### The Classification Problem

The first version of the chat assistant used the LLM itself to classify what the user was asking before routing to the right handler. This caused a visible triple flash in the UI and added multiple seconds of latency on every single query.

The second version ran a keyword scorer and LLM classifier in parallel, racing them. The keyword scorer was hundreds of lines of hand-tuned patterns that broke every time someone phrased a question slightly differently. "How much at Target" would match both "merchant query" (Target the store) and "spending total" (how much). The parallel approach added complexity without meaningfully improving accuracy.

The solution that actually works uses Apple's NaturalLanguage framework for embedding-based classification. Instead of pattern matching on words, it generates a vector representation of the user's query and compares it against precomputed reference points for each intent category. Classification dropped from seconds to milliseconds, works on iOS 17+ (not just iOS 26), and understands semantic meaning rather than keyword matches.

## Swift Charts: The Constraints Nobody Tells You

Four constraints discovered through production bugs, not documentation.

![SnapCents budget charts showing spending trends](IMAGE: screenshot-budget.png)

**Annotation position clipping.** Certain annotation positions on chart marks get clipped at the chart edge. If your data point is near the right side of the chart, the label just disappears. Took a while to figure out it was a positioning issue and not a data issue.

**Duplicate axis labels.** The automatic axis mark API sounds reasonable for letting the framework decide label count. In practice, it produces duplicate labels. January shows up three times. Explicit stride-based control is the fix.

**Budget period adjustment.** This is a data modeling issue, not a chart issue, but it surfaces in charts first. Budget values stored as monthly amounts need adjustment when displayed on different time periods. Weekly view needs the monthly value divided down, yearly needs it multiplied up. Forgetting this produces charts where the budget line is wildly wrong and nobody notices until the screenshot looks absurd.

**Reference line placement.** Budget reference lines only make visual sense on cumulative charts. On daily bar charts, they obscure the actual data. This seems obvious in retrospect. It wasn't when I was adding budget lines to every chart type.

## SwiftData: What Worked

In SnapCents, every expense is the same model regardless of how it entered the app. Scanned receipt, manual entry, voice entry, bank statement import: they all create the same record. This avoids the synchronization nightmare of maintaining parallel models for "receipts" and "transactions."

Enum storage was a source of early pain. The pattern that stuck: store the raw string value in the database, use a computed property for the typed enum. When you add new enum cases later, no migration needed. The string column accepts any value, and old rows with old values keep working.

![SnapCents business profile showing separate expense tracking](IMAGE: screenshot-business.png)

Profile-based filtering (personal vs. business expenses) required a specific pattern because SwiftData's query predicates are set at view creation time. You can't change them dynamically. The solution involves a wrapper view that rebuilds the query when the active profile changes.

## Bank Statement Parsing: When the LLM Lies

PDF bank statement parsing uses a multi-tier approach similar to receipt OCR. But the failure that led to this design is worth calling out.

Foundation Models was tested extensively for extracting transactions from bank statements. Given a page of statement text, the model was asked to pull out dates, descriptions, and amounts. The results were not usable.

It hallucinated transactions that didn't exist on the page. It dropped negative amounts (credits and refunds). It extracted the FDIC insurance limit ($250,000) as a transaction because the number appeared in the footer. Across multiple runs on the exact same input, it produced different results.

This wasn't a prompt engineering problem. The on-device model isn't reliable enough for structured financial data extraction from dense tabular text. The non-determinism alone is disqualifying for a finance app. You can't show someone their bank transactions and have them be wrong.

Foundation Models was removed from the extraction pipeline entirely. It now handles only two post-parse tasks: cleaning up merchant names (turning "AMZN MKTP US*2K4X7" into "Amazon") and suggesting categories. These are tasks where approximate answers are acceptable and being slightly wrong is harmless.

## What Building This Way Taught Me

The honest answer is that zero dependencies costs you time. There's no charting library that handles edge cases for you. There's no battle-tested OCR wrapper that abstracts away Vision's quirks. You write it, you debug it, you own it.

![SnapCents dark mode showing the full app experience](IMAGE: screenshot-dark.png)

But "you own it" cuts both ways. When the OCR produces wrong totals, you can read every line of the pipeline. When the AI chat gives a weird response, you know exactly which prompt produced it. When a user asks "does your app send my data anywhere?", you can answer with certainty, not with "we trust our third-party partners to respect privacy."

For a finance app, that certainty is the product.
