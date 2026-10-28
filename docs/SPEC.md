# Donation Sorter: Product and Technical Spec

Working name: Donation Sorter. Status: v1 draft, written Oct 28, 2026. This is a proposal to edit, not a finished decision list. Items marked **[DECIDE]** are open.

Repo: `donation-sorter` (Next.js, deployed on Vercel). Live URL: https://donation-sorter-kpft.vercel.app

---

## 1. Problem

College students move a lot (campus, internship locations, home). At move-out they end up with usable items (blankets, laundry baskets, hangers, small appliances) that could help shelters and reuse groups, but most of it gets thrown away. The barrier is not that students don't care. They are busy, and working out which organization accepts which item means researching several places and splitting a pile by each one's rules.

## 2. User

Primary user: a busy college student (starting with students at RPI **[DECIDE: RPI broadly, or my sorority first]**) who wants to reduce waste and donate, but won't spend an hour researching where things go.

Beachhead for testing: a small, reachable group (sorority members or friends) who can try it with a real closet cleanout, a winter break pack, or an internship move. Spring move-out is months away, so testing should not wait for it.

## 3. Goal of v1

A student takes a photo of a pile of items and gets back a list grouped by destination: what each organization accepts, its address and hours, and when that information was last verified. Items the photo can't judge are flagged with a question.

### Why this is a portfolio project

- Shows product thinking: a clear user problem, scope decisions, and measured results.
- Shows engineering: a deployed app in a public repo that I can explain line by line.
- Shows judgment about AI: the model describes, plain code decides, and uncertainty is shown to the user.

## 4. Non-goals for v1

- No pickups, storage, sorting, or any operations. The user drops items off themselves.
- No food. Shelf-stable food is a possible later extension (different organizations, expiration and safety rules).
- No accounts, database, or saved history.
- No claim that the organization data is complete or always current.

## 5. Principles (rules for me and for any AI coding tool)

1. **The AI describes; code decides.** The model only identifies items. Whether an organization accepts an item comes from the organization data file, never from the model's guess.
2. **Show uncertainty.** If confidence is low or a photo can't answer something (does the heater work? is the cord intact?), ask or say "check with the organization" instead of guessing.
3. **Honest data.** Every organization rule shows a `lastVerified` date. Anything I haven't verified is labeled sample data on the page.
4. **Photos are not stored by the app.** Process in memory only. Photos are sent to the model provider, so say that plainly on the page.
5. **Secrets stay server-side.** API keys live in environment variables, never in the browser or in git.
6. **Keep scope small.** If something isn't in this spec, ask before adding it.

## 6. User flow

1. **Intro:** two or three sentences on what this is and who it's for, plus a note on whether data is sample or verified, and a "Try it" button.
2. **Upload:** choose or take a photo (mobile first). The image is shrunk in the browser before upload.
3. **Results:** items detected, grouped by destination organization. Each group shows what the organization accepts, address, hours, and `lastVerified`. Items with no accepting organization are listed separately. Unsure items show a short follow-up question.
4. **Edit:** the user can remove or correct a detected item (the AI will sometimes be wrong), and the grouping updates.

Design reference: the Figma Make prototype. Save screenshots of each screen in `docs/design/` and describe them here **[DECIDE: add short descriptions once screenshots are saved]**.

## 7. Data contract

### 7.1 Categories (v1)

`blanket`, `laundry_basket`, `hangers`, `fan`, `heater`, `other`

Reason for this list: it's small, but varied enough to route to different places and to include a risky case (the heater) that should trigger a question.

### 7.2 Detected item (what the model returns)

```ts
type Category =
  | "blanket"
  | "laundry_basket"
  | "hangers"
  | "fan"
  | "heater"
  | "other";

type Condition = "good" | "worn" | "unclear";

interface DetectedItem {
  category: Category;
  quantity: number;          // rough count from the photo
  condition: Condition;      // visible cues only (stains, wear)
  confidence: "high" | "medium" | "low";
  note?: string;             // short free text, e.g. "possible stain on corner"
}

interface AnalysisResult {
  items: DetectedItem[];
}
```

The model must return JSON matching this shape. Validate it in code and reject or retry on anything else. Categories outside the list become `other`.

### 7.3 Organization data

Stored in `data/organizations.json`. No database in v1.

```ts
type Rule = "accept" | "reject" | "ask";

interface OrgRule {
  rule: Rule;
  note?: string;             // e.g. "clean only", "working condition only"
}

interface Organization {
  id: string;
  name: string;
  type: "shelter" | "thrift" | "campus_group" | "other";
  address: string;
  hours: string;
  website?: string;
  lastVerified: string;      // ISO date, shown in the UI
  isSample: boolean;         // true = placeholder data, labeled on the page
  rules: Record<Category, OrgRule>;
}
```

Placeholder organizations for early development: "Example Shelter", "Example Thrift Store", "Campus Reuse Group", all with `isSample: true`. Replace with 2 or 3 real local organizations once verified.

### 7.4 Matching

A plain function: `(items: DetectedItem[], orgs: Organization[]) => MatchResult`

- For each item, find organizations whose rule for that category is `accept` (or `ask`).
- `ask` rules and `low` confidence items are flagged for a follow-up question.
- Items with no accepting organization go in an "unmatched" group.
- Deterministic, no AI, and covered by unit tests.

## 8. Architecture

- **Framework:** Next.js (App Router, TypeScript), deployed on Vercel from GitHub on every push to `main`.
- **Front end:** a mobile-friendly page with the four screens above.
- **Server route:** `app/api/analyze/route.ts` receives the image, calls the provider wrapper, validates the JSON, and returns it.
- **Provider wrapper:** one function, `analyzeImage(image, provider)`, that returns `AnalysisResult`. The provider is a setting.
  - Gemini first, using a free API key from Google AI Studio and a current vision-capable Gemini model **[DECIDE: check AI Studio for the model name and free limits]**.
  - Groq (or another free vision-capable provider) added later so both can be compared on the evaluation set. Verify Groq's current free limits before relying on it.
- **Environment variables** (in `.env.local` locally and in Vercel settings): `GEMINI_API_KEY`, `GROQ_API_KEY` (later), `ACCESS_CODE`, `ANALYSIS_PROVIDER`.
- **Matching:** `lib/match.ts`, reading `data/organizations.json`.

## 9. Safety, privacy, and abuse limits

- **Rate limit and access code:** a free tier can't produce a surprise bill, but strangers can use up the quota and break the demo. Add a simple rate limit and an optional access code while it's in testing.
- **Image handling:** shrink images in the browser before upload. Serverless request size limits are typically a few MB, so verify the current limit for the Vercel plan.
- **Privacy note on the page:** "Photos are processed to identify items and are not stored by this app. They are sent to an AI provider for analysis. Free-tier AI providers may use submitted data to improve their products (verify the current terms), so please don't upload photos with faces or personal items."
- **Test photos:** staged photos of piles I own. Don't use other people's rooms without asking.

## 10. Evaluation plan

Build a labeled set of 20 to 30 photos of real piles (my own closet, willing friends), with the correct items written down for each.

Measure, for each model (Gemini, then a second provider):

- Share of real items correctly identified (by category).
- Number of false items invented.
- Share of items routed to the right destination after matching.
- Any case where an unsafe or unsuitable item would have been recommended (these matter most).
- Where it fails (clutter, bad lighting, items overlapping).

Targets for "good enough" **[DECIDE: set thresholds before running, not after]**. A starting suggestion to edit: no unsafe items routed as accepted, and most items correctly identified on clear photos.

Record the results and failure examples in the README. The numbers and the failures are the interview material.

## 11. Success metrics for later user testing (after Nov 7)

Try it with 5 to 10 real students on a real cleanout. Track:

- Did they finish the flow without help?
- How long did sorting take compared with doing it by hand?
- Would they say the grouping was right and useful?
- Would they use it again?

Also compare against a plain "what goes where" list: does the photo step actually add value? Be ready to report that honestly either way.

## 12. Schedule (Oct 28 to Nov 4; WE26 is Nov 5 to 7)

| Day | Goal |
| --- | --- |
| Wed Oct 28 | This spec, types, placeholder organization data, a throwaway script that sends 3 to 5 photos to Gemini and prints results |
| Thu Oct 29 | Research 2 to 3 real organizations (their posted accept lists, or a call for unclear items); build the `/api/analyze` route with the provider wrapper |
| Fri Oct 30 | Matching function plus unit tests |
| Sat Oct 31 to Sun Nov 1 | Interface: upload, results, edit list; test on a phone |
| Sun Nov 1 to Mon Nov 2 | Evaluation set; rate limit and access code |
| Mon Nov 2 to Tue Nov 3 | README, privacy note, screenshots, final deploy, 2 to 3 people try it |
| Wed Nov 4 | Buffer and 60-second pitch for the conference |

**If behind, cut in this order:** (1) follow-up questions for unsure items, (2) edit-the-list screen, (3) the second provider, (4) the full evaluation (can slip until after the conference).

**Do not cut:** the organization data file with `lastVerified`, the README, the deployed URL, or the public repo.

## 13. Done criteria for the conference

- Public GitHub repo with a clear README (problem, user, how it works, limits).
- Deployed URL that works on a phone.
- Honest labels on any sample data.
- At least a first evaluation number, if time allows.
- I can explain every file in the repo.

## 14. Open questions

- **[DECIDE]** Real organizations to include, and how to verify what they accept.
- **[DECIDE]** App name.
- **[DECIDE]** Focus: RPI broadly, or my sorority first.
- **[DECIDE]** Whether follow-up questions are fixed per category (simpler) or generated.
- Later: shelf-stable food as an extension; more categories; a second provider as a fallback.

## 15. Working agreement for AI coding tools

When helping on this repo, please:

1. Read this file first and follow the principles in section 5.
2. **Propose a plan before writing code**, and wait for my review.
3. Work on one feature at a time and keep changes small so I can read each diff.
4. Explain non-obvious code in plain language. I need to be able to explain every file in an interview.
5. Write simple tests for the matching logic.
6. Never commit secrets or `.env` files, and never put API keys in browser code.
7. Don't add accounts, a database, pickups, food, or other features that aren't in this spec without asking.
