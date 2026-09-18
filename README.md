# ReliefPulse — Hyper-Local Emergency & Community Dispatch

Single-file web app. No build step, no npm install, no server required.

## Run it

**Fastest:** double-click `index.html`. It opens in your default browser and works offline.

**In VS Code (recommended):**
1. Put `index.html` in a folder and open that folder in VS Code.
2. Install the **Live Server** extension (Ritwick Dey).
3. Right-click `index.html` → **Open with Live Server** → it serves at `http://127.0.0.1:5500`.

Live Server is worth the extra step: microphone dictation in the triage modal needs a secure context (`http://localhost` counts, a raw `file://` path does not), and you get hot reload while you edit.

**Any other static server also works:**
```bash
python3 -m http.server 5500      # then open http://localhost:5500
npx serve .                      # or this
```

## Browser support

Chrome, Edge, Firefox, Safari, and mobile browsers, current versions. Two graceful degradations:

- **Speech input** uses the Web Speech API — available in Chrome, Edge, and Safari. Firefox has no support, so the mic button disables itself and tells the user to type instead.
- **`:has()` selector** styles the checked state of checkboxes. Firefox 121+, Chrome 105+, Safari 15.4+. On anything older the checkboxes still work, they just lose the green highlight.

State persists in `localStorage` under `reliefpulse.table.v1`. Clear it from DevTools → Application → Local Storage to get the seed data back.

## Code map

Everything lives in `index.html`, in numbered sections inside the one `<script>` block:

| Section | What it is |
|---|---|
| 1. Domain constants | Categories, urgency bands, score→band mapping |
| 2. `Dynamo` | DynamoDB single-table simulation. `PK: INCIDENT#id \| VOLUNTEER#id \| HAZARD#id`, `SK: METADATA` |
| 3. `S3` | Presigned-upload simulation with progress and a real bucket-style key |
| 4. `Bedrock` | Triage. Live model path plus a deterministic on-device classifier fallback |
| 5. `api` | One function per Lambda route: `fetchIncidents`, `createSOS`, `createOffer`, `matchVolunteer`, `resolveIncident`, `triage`. `rank()` is the matching engine |
| 6. Seed data | 5 incidents, 4 volunteers, 3 hazards |
| 7–11 | View state, toasts, dispatch log, filters, map/feed rendering |
| 12–13 | Actions and modals (SOS, offer aid, triage) |
| 14–15 | View toggle, map tools, boot |

## Wiring the real AWS stack

Running outside claude.ai, the triage automatically uses the local classifier — the `window.claude` check in `Bedrock.link()` fails and it falls back silently. To point it at your own backend, replace the bodies in section 5; the UI calls nothing else.

```js
const ENDPOINT = "https://<api-id>.execute-api.us-east-1.amazonaws.com/prod";

const api = {
  async fetchIncidents() {
    const r = await fetch(`${ENDPOINT}/incidents`);
    return r.json();
  },
  async createSOS(input) {
    const r = await fetch(`${ENDPOINT}/incidents`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(input)
    });
    return r.json();
  },
  async triage(text) {
    const r = await fetch(`${ENDPOINT}/triage`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ text })
    });
    return r.json();   // { urgencyScore, categories, summary, rationale, recommendedAction, volunteerIds }
  }
};
```

Your Lambda behind `/triage` calls `bedrock-runtime:InvokeModel` with the prompt already written in section 4 — it asks for JSON only and names the exact keys, so you can paste it straight into the handler.

For S3, swap `S3.upload()` for a request to a `/uploads/presign` route, then `PUT` the file at the returned URL and keep the object key on the incident item.

## Splitting it into Next.js

If you'd rather have a component tree than one file, the natural break is:

```
app/page.tsx                  → the stage: metrics, controls, map/feed
app/api/triage/route.ts       → Bedrock InvokeModel
app/api/incidents/route.ts    → DynamoDB CRUD
components/IncidentMap.tsx    → SVG map + pins + callout
components/FeedGrid.tsx       → card grid
components/TriageModal.tsx    → sections 13 + renderTriage
components/SosForm.tsx
components/OfferForm.tsx
lib/aws/dynamo.ts             → section 2
lib/aws/s3.ts                 → section 3
lib/aws/bedrock.ts            → section 4
lib/api.ts                    → section 5
```