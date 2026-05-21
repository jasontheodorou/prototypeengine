# GOV.UK Prototype Engine

A web app that generates GOV.UK Prototype Kit v13 prototypes from a plain English brief.

---

## What it does

A designer describes a service in plain English. The app calls Claude, which returns a complete prototype spec. The app turns that spec into a working GOV.UK Prototype Kit project and gives the designer a zip to download.

---

## Running locally

```
npm install
ANTHROPIC_API_KEY=... \
GITHUB_TOKEN=... \
GITHUB_USERNAME=... \
RENDER_API_KEY=... \
RENDER_OWNER_ID=... \
npm start
```

Open http://localhost:3000

For day-to-day development, put these in a `.env` file (already gitignored) and load it with a tool like `dotenv-cli`.

---

## Deploying to Render

1. Push this repo to GitHub.
2. Create a new Web Service on Render pointing at the repo.
3. Build command: `npm install`
4. Start command: `npm start`
5. Set the environment variables listed below.
6. Deploy.

---

## Environment variables

All five are required. The app will return a 500 from the generate endpoints if any are missing.

| Variable | Required | Description |
|----------|----------|-------------|
| `ANTHROPIC_API_KEY` | Yes | Anthropic API key from console.anthropic.com. Used for the Claude calls. |
| `GITHUB_TOKEN` | Yes | GitHub personal access token with `repo` scope. Used to create a repo and push files per prototype. |
| `GITHUB_USERNAME` | Yes | The GitHub account that will own each generated prototype repo. |
| `RENDER_API_KEY` | Yes | Render API key. Used to create a new web service per prototype. |
| `RENDER_OWNER_ID` | Yes | The Render workspace or team ID the new services are created under. |
| `PORT` | No | Port to run on. Render sets this automatically. |

---

## What the generated prototype includes

- Start page with service description and what the user needs
- Three to five question pages with working GDS components
- Server-side validation with error summaries on every question
- Check your answers page
- Confirmation page with a generated reference number
- At least one ineligibility route

---

## Cost

Each prototype generation costs roughly 2–5p in Claude API usage depending on brief length.
