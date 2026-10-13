# The human evaluation Tinder 📱

## The bright idea

Human evaluations are really tiring to obtain. So let’s turn language-generation evaluation into a swipe-based mobile experience: the human evaluation Tinder. Instead of dragging annotators into a spreadsheet or a clunky web form, present them with one comparison at a time (e.g., two model outputs, or one output + a specific quality dimension) and let them swipe: right for "better/pass," left for "worse/fail," up for "flag/uncertain/tie," etc. Sessions are short, async, and mobile-native, so people can knock out evaluations in idle moments (waiting for coffee, on the bus) instead of sitting down for a dedicated labeling session. Layer in gamification (streaks, levels, leaderboards, "taste badges" (e.g., "Readability Referee," "Fact-Checker Tier 3"), …) to turn a tedious annotation chore into something people actually want to open. 

## The goal

Lower the friction and boredom of human evaluation enough to get more labels, more often, from a more diverse pool of evaluators.

**Basic**
1. Set up your development database backend (see below) and confirm that a test vote shows up (emulator UI or Firestore console).
2. Load [evaluation_data.csv](https://github.com/MiriUll/hackathon-2026/blob/main/task1-tinder/evaluation_data.csv); show cards with prompt, response(s) and the question.
3. Implement the different annotation swipes and save them to the database; add a **undo** function.
4. Polish the UI and make your tool publicly available to run on phones.

**Extended**  
5. Add gamification elements (streaks, points, levels, badges, leaderboard, onboarding). Rewarding speed can encourage careless swiping: reward careful voting.  
6. Data-quality safeguards (attention checks, cool-down after very fast swipes, "read first" nudge).

## Data description

### Data to judge

We are using a small subset from [Arena Human Preference 90K](https://huggingface.co/datasets/hrnrxb/Arena_Human_Preference_90K_features_verified). This enables us to judge single-model responses, but also compare and select competing answers. The subset is provided in [evaluation_data.csv](https://github.com/MiriUll/hackathon-2026/blob/main/task1-tinder/evaluation_data.csv).  
It provides these columns:
| Column | Use |
|---|---|
| `id` | item id (`item_id` in votes) |
| `user_prompt` | the prompt given to the model |
| `model_a_response`, `model_b_response` | the two responses (model names are never shown) |
| `is_correctness_applicable`, `is_response_a_correct`, `is_response_b_correct`, `winner` | reference labels of the dataset, that should be obtained from the annotators |

### Data to obtain from the annotators

Every submitted annotation is **one document per annotated item**, written to Cloud Firestore at
`annotations/{team_name}/entries/{auto-id}` (see below for Firebase details).  
The security rules are **create-only**: apps cannot read, change or delete documents, and documents that violate the format below are rejected.

| Field | Type | Description |
|---|---|---|
| `item_id` | string | `id` of the sample that was annotated |
| `team_name` | string | Name of the submitting team (as registered by the organizers; must match the path) |
| `annotator_id` | string | Device-specific ID to tell annotators apart: generate a random UUID on first launch and store it locally. **Not shown to the user** |
| `is_correctness_applicable` | bool | Can the correctness of a response to this prompt be judged? (see dataset) |
| `is_response_a_correct` | bool or `null` | Is response A correct? `null` if not rated (e.g. only B was shown, or correctness is not applicable) |
| `is_response_b_correct` | bool or `null` | Is response B correct? Same as above |
| `winner` | string or `null` | Comparison result as in the dataset: `model_a`, `model_b`, `tie`, `both_bad`; `null` if the comparison was not asked |
| `response_latency` | int | Milliseconds from first showing the sample to submitting the annotation (useful to identify lazy swiping) |
| `created_at` | timestamp | Set by the app with `FieldValue.serverTimestamp()` |

All fields must be present (use `null` where allowed). If `is_correctness_applicable` is `false`, both correctness fields should be `null`. Extra fields are allowed.

```json
{
  "item_id": "a1b2c3",
  "team_name": "team-07",
  "annotator_id": "6f1c0c2e-8d2a-4a53-9b5e-0c1f0e7d4a11",
  "is_correctness_applicable": true,
  "is_response_a_correct": true,
  "is_response_b_correct": null,
  "winner": null,
  "response_latency": 5320,
  "created_at": "<serverTimestamp>"
}
```

*Undo / changed mind:* submit a new annotation for the same `item_id` and `annotator_id`; the latest `created_at` counts.

## Technical Details

### Stack
- **Recommended:** [Flutter](https://flutter.dev) (Android, iOS, **Web**). Alternatives: Any web-based or cross-platform application that users can access easily.
- **Build for the web** and host it at a public URL (GitHub Pages, Netlify, Firebase Hosting, …) so others can open it on a phone. The organizers' Firebase is set up for web apps.
- Useful Flutter packages: `flutter_card_swiper`, `firebase_core`, `firebase_auth`, `cloud_firestore`, `flutter_markdown`.

### Data collection
The annotations are collected in a [Firebase database](https://console.firebase.google.com/). For development, you need to setup your own database or use a local emulator.  You can find the [firestore.rules](https://github.com/MiriUll/hackathon-2026/blob/main/task1-tinder/firestore.rules) in this repo to mimic the final submission setup.
The configuration for the annotations database will be provided at the end of the hackathon.

## Rating your apps
Teams are judged based on **the app and the presentation**; the submitted annotations are **not** analyzed or scored.

| # | Criterion | Pts | What the jury looks at |
|---|---|---|---|
| 1 | **Working app and dimension coverage** | 30 | Demo: the required questions work, and every swipe arrives in Firestore with the right fields |
| 2 | **Usability and design** | 25 | Clear, quick, pleasant on a phone; long responses are readable |
| 3 | **Engagement and gamification** | 20 | Creative mechanics that make people want to evaluate, and to evaluate carefully |
| 4 | **Data-quality safeguards** | 10 | Randomizing A and B order, undo, latency logging, attention checks |
| 5 | **Presentation and reflection** | 15 | Clarity, insights, honest limitations |
