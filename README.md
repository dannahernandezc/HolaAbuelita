# HolaAbuelita

**A reasoning agent that turns a short daily check-in into peace of mind for families caring from afar.**

Built for the Microsoft Agents League Hackathon (June 2026) — **Reasoning Agents track**, with Foundry IQ as the core Microsoft IQ layer.

This project was designed with four judging angles in mind: a grounded reasoning agent (Reasoning Agents track), genuinely meaningful use of Foundry IQ (Best Use of IQ Tools), an accessibility-first design for an elderly, non-technical user (Accessibility Award), and a real community need around eldercare coordination (Hack for Good).

---

## The problem

This project started from a real situation: my grandmother spends part of the day alone, and as a family, we can't be physically present with her all day. Like many families, we found ourselves in that familiar gap — wanting to know she's okay, but not having a simple way to check in consistently, and never quite sure who was supposed to call her that day.

This is far from a unique situation. Many families care for an elderly relative who spends part of the day alone, and this creates two recurring problems:

1. **Uncertainty** — family members don't know how their relative is really doing day to day.
2. **Coordination breakdown** — when something does come up, everyone assumes someone else has already checked in, and nobody actually does.

HolaAbuelita addresses both: a short daily conversation gathers real information, a reasoning agent evaluates it against care guidelines, and the right family member is notified automatically — with fair rotation so responsibility doesn't fall on one person.

This is not a surveillance tool. There are no cameras. The goal is reducing daily uncertainty and coordination overhead through grounded AI reasoning and a simple conversation.

**Accessibility is at the core of this design.** The primary user is an elderly person — often not comfortable with apps, dashboards, or new technology. The entire interface is a short daily chat conversation in a tool many people already have (Telegram), in their own language, using plain everyday questions. No login, no app to learn, no screen to navigate. The "interface" is a conversation, which is the most accessible interface there is for this audience.

---

## How it works

1. **Daily check-in (Telegram).** Each morning (or on demand, for this demo), the elderly person receives a short conversation with four questions:
   - How are you feeling today?
   - Have you had breakfast or eaten something yet?
   - Did you take your medication today?
   - Is there anything you'd like to tell me, or anything you need?

2. **Reasoning with Foundry IQ.** The four responses are sent to an agent built on **Azure AI Foundry** (gpt-4o), grounded with **Foundry IQ** over a knowledge base of elder care guidelines covering:
   - Mood and language warning signs
   - Medication adherence
   - Nutrition and hydration

   The agent classifies the day as **normal**, **moderate attention**, or **urgent**, and explains its reasoning with a citation to the relevant guideline.

3. **Family notification (Telegram).** Based on the classification, a message is sent to a shared family group:
   - **Normal** → a brief "all good today" summary.
   - **Moderate attention** → a summary plus an assigned family member (rotating fairly across four relatives) who should follow up.
   - **Urgent** → an immediate alert to everyone, including the case where the elderly person doesn't respond to the check-in at all.

4. **Bilingual by design.** The check-in and the family notification both work in Spanish and English — reflecting the reality of families split across countries, where the elderly relative and some family members may not share a first language.

5. **Orchestration with n8n.** The end-to-end conversation flow — sequential questions, conversational state, calling the Azure AI Foundry agent, and routing the family notification — is built in n8n.

---

## Why Foundry IQ is central to this project (not bolted on)

A common failure mode for "AI agent" projects is using an LLM to generate plausible-sounding advice with no real basis — which is especially dangerous in an eldercare context, where a wrong judgment call has real consequences.

HolaAbuelita's reasoning agent doesn't classify a check-in "from vibes." It is grounded with **Foundry IQ** over a knowledge base of six documents (three in English, three in Spanish) covering:

- **Mood and language warning signs** — what distinguishes a normal bad day from language that warrants a closer look or urgent attention.
- **Medication adherence** — when a missed dose is routine versus when a pattern becomes a real risk.
- **Nutrition and hydration** — how to read between the lines of "I wasn't very hungry today."

When the agent classifies a check-in, it returns **which guideline it used and why** — this is Foundry IQ's grounded, cited retrieval in action, returned as part of the agent's structured output. This is the difference between "the AI thinks this seems fine" and "according to the medication adherence guideline, missing a dose once is routine, but two consecutive days warrants attention" — a transparent, auditable reasoning trail that a family member (or a judge) can actually verify.

---

## Architecture

```
Elderly person (Telegram)
        │
        ▼
 n8n: sequential check-in
 (4 questions, state tracked
  between messages)
        │
        ▼
 Consolidated daily responses
        │
        ▼
 Azure AI Foundry Agent (gpt-4o)
 grounded with Foundry IQ
 (care guideline knowledge base,
  ES + EN)
        │
        ▼
 Classification: normal /
 moderate_attention / urgent
 + reasoning + family message
        │
        ▼
 n8n: rotation logic
 (4 caregivers)
        │
        ▼
 Family group (Telegram)
```

*A visual version of this diagram is included in `screenshots/architecture.png`, as required by the contest submission guidelines.*

---

## What's in this repository

| File / folder | Description |
|---|---|
| `knowledge-base/` | The 6 care guideline documents used by Foundry IQ (3 in Spanish, 3 in English): mood & language signals, medication adherence, and nutrition/hydration. |
| `n8n-workflow.json` | Exported n8n workflow: Telegram bot, sequential check-in logic, Azure AI Foundry agent call, classification handling, and family notification with caregiver rotation. |
| `agent-system-prompt.md` | The system prompt used to configure the reasoning agent in Azure AI Foundry, including the classification schema and bilingual rules. |
| `screenshots/` | Screenshots of the Azure AI Foundry portal showing Foundry IQ grounded responses with citations, plus `architecture.png` (visual architecture diagram). |

---

## Why these technical choices

**Why Telegram instead of WhatsApp?** Telegram's bot API is free, has no approval process, and integrates natively with n8n. For a hackathon timeline, this removed an entire layer of friction. In a production deployment for Latin American families, this would migrate to the WhatsApp Business API, which is more familiar to the target users.

**Why n8n alongside Azure AI Foundry?** Foundry IQ and the Azure AI Foundry agent provide the grounded reasoning — the core "Microsoft" piece of this project. n8n orchestrates the conversation: sequencing questions, tracking state between messages, and routing the final notification. This combination reflects a realistic integration pattern: Microsoft AI reasoning embedded inside an existing automation stack.

**Why gpt-4o instead of gpt-4o-mini?** Foundry IQ's query planning requires a model from the gpt-4o/4.1/5 family. gpt-4o-mini's relevant model version was deprecated at the time of building this project, and gpt-4o had available quota in our region — making it both the available and the correct choice for Foundry IQ compatibility.

**Why "extractive" output mode for the knowledge base?** This ensures Foundry IQ returns grounded excerpts from the actual guideline documents (which the agent can cite), rather than generating free-form answers that might drift from the source material — directly supporting the "Reliability & Safety" goals of this project.

---

## Demo

https://www.youtube.com/watch?v=6oHuiQCaowE

The demo shows:
- A full check-in conversation in Telegram (English)
- The Azure AI Foundry portal, showing the agent's grounded classification with a citation to the knowledge base
- The resulting family notification, including the urgent case
- The same flow completed in Spanish, showing the bilingual behavior

---

## Roadmap / next steps

- **Voice support via Azure AI Speech.** The elderly person could respond via voice note (speech-to-text with automatic ES/EN language detection), with optional spoken responses from the agent — improving accessibility for users who find typing difficult or have limited vision.
- **WhatsApp Business API** for real-world deployment with Latin American families, where WhatsApp adoption among elderly users is far higher than Telegram.
- **Configurable setup.** Move language, caregiver names, and check-in questions into a single configuration block, so the same flow can be reused for any family without editing the workflow logic — making this genuinely reusable for other families facing the same problem (Hack for Good).
- **No-response detection.** Currently the flow is triggered manually for this demo; in production, a scheduled trigger combined with a timeout check would flag a missed check-in as an "urgent" signal on its own — arguably the single most important real-world feature, since silence can be the most important signal of all.

---

## A note on the community need

This situation is far from unique. As populations age and families become more geographically dispersed (whether across cities or across countries), the "sandwich generation" caring for both children and aging parents faces a growing coordination problem that most existing tools don't address: not health monitoring, but peace of mind and fair distribution of care responsibility.

HolaAbuelita is a small, concrete step toward solving that — built to be reusable, bilingual, and centered on the person who is least likely to adapt to new technology.

---

## Built by

Danna Hernández Cardoso — Senior Product Analyst & Industrial Engineer.

Built solo, motivated by my own family's situation.
