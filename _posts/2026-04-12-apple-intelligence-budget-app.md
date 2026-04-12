---
layout: blog
title: "How Apple Intelligence Works Inside a Real Budget App"
date: 2026-04-12
description: "An honest look at what Apple's on-device Foundation Models can and can't do for personal finance. Based on shipping a real app, not marketing slides."
reading_time: "10 min read"
---

Apple shipped Foundation Models in iOS 26. It's an on-device language model that runs locally on the iPhone's Neural Engine. No cloud. No API keys. No data leaving the device. For a privacy-first finance app, this sounded perfect.

After months of building with it, here's what it actually does well, what it fails at, and why the gap between Apple's marketing and the practical reality matters.

## What the On-Device Model Is

Apple's Foundation Models framework gives developers access to a language model that runs entirely on the iPhone. It requires an iPhone 15 Pro or later (the Neural Engine in the A17 Pro chip and newer is what makes it possible). The model supports text generation, structured output with constrained decoding, and tool calling.

"On-device" means exactly what it sounds like. The model weights live on the phone. Inference happens on the phone. Your prompts and data never leave the phone. There's no fallback to a cloud API. If the phone can run it, it runs locally. If it can't, it doesn't run at all.

This is fundamentally different from how most AI features work in apps. When Copilot Money or Cleo say they have "AI insights," they're sending your financial data to servers running GPT-4 or similar models. The AI is smart, but your data is in the cloud. Apple's approach trades model capability for genuine privacy.

## What It Does Well

### Categorizing Expenses

When you scan a receipt or enter an expense manually, the model suggests a category. "Chipotle" gets categorized as "Food & Dining." "Shell" goes to "Gas & Fuel." "Blue Bottle Coffee" maps to "Coffee."

This works reliably because it's a classification task with a constrained output set. The model can only pick from your existing categories. It physically cannot hallucinate a category that doesn't exist because the output is generated against a defined enum using constrained decoding. The model's token generation is restricted to valid category values at each step.

For common merchants, accuracy is high. For ambiguous ones ("Target" could be groceries, household, or clothing), it picks the most common category, which you can correct with a tap.

### Answering Spending Questions

"How much did I spend on food this month?" The AI assistant queries your actual transaction data through a tool calling system and responds with a real number.

This works because the model doesn't need to know your spending data. It needs to understand the question, call the right tool with the right parameters, and format the response. The tool does the actual database query. The model translates between human language and structured function calls.

Tool calling with constrained parameters is where on-device models perform well. The model picks which tool to call and fills in parameters from a defined set of options. Time ranges, categories, and query types are all enums. The model can't ask for a time range that doesn't exist or a category your app doesn't have.

### Cleaning Up Merchant Names

Bank statements and credit card charges have terrible merchant names. "AMZN MKTP US*2K4X7" is Amazon. "SQ *BLUE BOTTLE COF" is Blue Bottle Coffee. "GOOGLE *STORAGE" is Google One.

The model is good at this. It's a pattern recognition task where being approximately right is fine. If it turns "AMZN MKTP US*2K4X7" into "Amazon" instead of "Amazon Marketplace," nobody cares. Close enough is genuinely good enough.

## What It Fails At

### Structured Data Extraction from Dense Text

This is the big one. We tested Foundation Models extensively for extracting transactions from bank statement PDFs. Given a page of statement text with dates, descriptions, and amounts in tabular format, the model was asked to pull out structured transaction data.

The results were not usable for a finance app.

It hallucinated transactions that didn't exist on the page. A 30-transaction page might come back with 32 entries, two of which were fabricated. It dropped negative amounts, meaning credits and refunds would disappear. It extracted the FDIC insurance limit ($250,000) from the footer as if it were a transaction amount. And the same input produced different outputs across multiple runs.

Non-determinism is the real killer for financial data. If you run the same extraction twice and get different transaction counts, you can't ship that. Users need to trust that their imported bank statement matches reality.

We removed Foundation Models from the extraction pipeline entirely. Bank statement parsing now uses regex patterns and structured text analysis through Apple's Vision framework. It's less elegant but deterministic.

### Long-Form Financial Analysis

Asking the model to write a paragraph analyzing your spending trends over three months produces output that reads like a horoscope. Vague, non-specific, and occasionally contradicted by the actual data.

The on-device model is smaller than cloud models like GPT-4 or Claude. That's the tradeoff for running locally. It handles short, structured tasks well. It struggles with nuanced multi-factor analysis.

The workaround: do the analysis in code, present numbers and trends through charts and summaries, and use the AI only for the conversational interface layer. The model answers questions. The app does the math.

### Handling Ambiguous Financial Queries

"Am I doing okay financially?" is a question the model can't meaningfully answer. It doesn't know your income, your savings goals, your debt situation, or your risk tolerance. The on-device model doesn't have the context window or reasoning capability to combine multiple data sources and produce nuanced financial advice.

We limit the assistant to factual spending queries. What did you spend, where, when, and how does it compare to your budget. Questions it can answer by looking up data, not questions that require judgment.

## The Tool Calling System

The AI assistant in SnapCents uses a tool calling architecture. The model doesn't have direct access to your spending database. Instead, it has a set of tools it can invoke, each designed for a specific type of query.

When you ask "how much did I spend at restaurants this week?", the model determines the intent (spending query), selects the appropriate tool, and passes parameters: category = dining, time range = this week. The tool runs the actual database query, formats the result, and hands it back to the model for presentation.

Every parameter in the tool definitions uses constrained types. Categories are enums matching your category list. Time ranges are predefined periods. This eliminates an entire class of errors where the model might generate invalid queries.

The token budget matters here. Apple's on-device model has a tight combined limit for input and output tokens. Every word in your system prompt, tool descriptions, and conversation history eats into the space available for the response. Tool descriptions had to be compressed to single sentences. The system prompt was rewritten multiple times to stay under budget while keeping the model on track.

## What "On-Device AI" Means for Privacy

Most finance apps that advertise AI features are sending your data to cloud APIs. Your transaction history, merchant names, amounts, categories, and spending patterns go to OpenAI, Anthropic, or Google's servers. The app's privacy policy might say the data is "used only for providing the service" or "not stored after processing," but it still leaves your device.

Apple's on-device approach is different because there's no network call. The model runs on the Neural Engine. Your prompts are processed in local memory. Nothing is transmitted. This isn't a privacy policy promise, it's a technical architecture constraint. The data can't leave because there's nowhere for it to go.

For a finance app, this distinction matters. People have strong feelings about who sees their spending data. The on-device approach lets you offer AI features without the privacy compromise that typically comes with them.

## The Honest Tradeoff

On-device AI is less capable than cloud AI. GPT-4 and Claude would produce better spending analyses, catch more nuanced patterns, and handle ambiguous queries more gracefully. If you don't care about privacy and just want the smartest possible AI assistant, a cloud-connected app will give you that.

The tradeoff is genuine. You get privacy (your data never leaves your phone), speed (no network latency), and offline capability (works in airplane mode). You give up model capability, context window size, and the ability to handle complex reasoning tasks.

For the specific use case of a spending assistant that answers factual questions about your own data, the on-device model is good enough. It categorizes accurately with constrained outputs. It routes queries to the right tools reliably. It formats responses clearly.

Where it falls short, the app compensates with traditional code. Charts, trends, budget calculations, and reports are all computed deterministically. The AI is a conversational interface to your data, not the system that processes it.

That's probably the right framing for on-device AI in 2026. It's a UI layer, not an intelligence layer. And for a finance app where accuracy matters more than cleverness, that's fine.
