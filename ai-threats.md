# Threats from using a large language model

The app uses the Claude API to read county claim forms, review drafts, answer questions about the app, and research public information. These threats follow the OWASP Top 10 for LLM Applications where they apply.

## A1: Prompt injection from content the app reads

**Threat:** a county PDF, a web page or a search result contains text such as "ignore your instructions and…". The model then follows it.

**Why the impact is limited here:**
- **The model can't act.** It has no tools that change data, send messages or place calls.
- **Answers are checked.** Every answer must match a JSON schema, and it is then validated again with zod before the app uses it. Free text where a structure was expected is discarded.
- **A person approves everything.** Answers are kept as **suggestions**. A checklist line read from a claim form, for example, isn't on the checklist until an admin presses Add, and each suggestion carries the form's own words as a quote, so the person can check it against the source.
- **Research prompts set the content apart.** Outside content is wrapped and labelled as material to read, never as instructions.

## A2: Personal data sent to the model provider

**Threat:** more personal data than needed leaves the system, and the business's privacy promises are broken.

**Controls:**
- **Only the content a task needs is sent.** Claim forms are public county documents, and research works with public web pages.
- **Ask AI** (help with using the app) replaces names, phone numbers and email addresses with labels before sending. The screenshot of the page is not sent.
- **Privacy tests:** about 13 test files assert what may and may not reach the model for each feature.

## A3: Spend abuse

**Threat:** a loop, a stuck button or a stolen session runs up the API bill.

**Controls:**
- **Limits per user:** one call at a time, with a per-feature cap per minute and per day.
- **A monthly cap** on scheduled research, counted in cents.
- **An account-wide stop:** when a batch of claim-form readings hits an error that every request would hit, such as no credits left, the batch stops at the first one instead of repeating it.
- **Costs are shown** after each AI action, so unusual spend is visible.

## A4: Over-trust in model output

**Threat:** a person accepts a wrong answer because it looks confident. On a legal filing, that could cost a claimant their money.

**Controls:**
- **Every result is a suggestion,** with the source quoted where possible.
- **Facts that decide a filing** (each state's requirements, a county's rules) can't be used to generate documents until they're recorded with their sources and are current.

## A5: The app can't work without the model

**Threat:** the provider is down or credits run out, and work stops.

**Control:** every feature that uses the model documents a manual way to do the same job, shown beside the feature. Running out of credits is announced in a banner that links to those steps.

## A6: Which code may call the model

**Threat:** model calls spread through the codebase, and the controls above get bypassed by a new feature.

**Control:** only two modules may import the AI SDK. A test fails the build if any other file does.
