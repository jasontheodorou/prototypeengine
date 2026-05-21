# Build guide: Prototype Engine

A plain English spec for a developer building a fresh version of the Prototype Engine.

---

## What you are building

A web app that lets a service designer describe a service in plain English and get back a live, shareable GOV.UK prototype within five minutes.

The designer never sees code. They write a brief. The app does everything else.

---

## How it works

1. The designer opens the app and writes a short brief. They can also upload a PDF and add a reference URL.
2. The app sends the brief to Claude.
3. Claude returns a JSON spec for the prototype.
4. The app turns that JSON into a full set of GOV.UK Prototype Kit files.
5. The app creates a new GitHub repo and pushes those files into it.
6. The app creates a new Render web service that deploys the repo.
7. The app waits for the prototype to go live.
8. The app returns a shareable URL to the designer.

That is the whole flow.

---

## What you need

### Stack

- Node.js 22
- Express
- Multer (for PDF uploads)
- pdf-parse (to read uploaded PDFs)
- node-fetch (or native fetch)
- No frontend framework. The UI is plain HTML, CSS and a small amount of JavaScript.

### Accounts and keys

You need accounts and API keys for:

- **Anthropic** — for Claude
- **GitHub** — a token with `repo` scope to create and push to repos
- **Render** — an API key and an owner ID to create web services

All of these are read from environment variables at runtime. See [Environment variables](#environment-variables).

---

## The three parts of the app

The app has three responsibilities. Keep them in separate files.

### 1. The host app

An Express server that:

- Serves the designer-facing pages (the brief form, the progress screen, the success screen)
- Accepts the form submission
- Calls Claude
- Calls the generator
- Calls GitHub and Render
- Returns the live URL

### 2. The Claude integration

A single prompt that tells Claude:

- Act as a senior GOV.UK service designer
- Read the brief
- Return a JSON object that matches a specific shape (see [The spec](#the-spec-claude-returns))
- Use GOV.UK Prototype Kit v13 conventions
- Use GDS content rules (plain English, reading age 9, active voice, "you" for the user)

You call Claude through a direct HTTPS POST to `api.anthropic.com/v1/messages`. You do not need an SDK.

If a PDF is uploaded, do a first Claude call that summarises the PDF against the brief. Then pass the summary into the main generation call.

### 3. The generator

A set of functions that take Claude's JSON and write out a complete Prototype Kit project as an in-memory file map (filename to file contents).

The generator must:

- Validate Claude's output and patch it where needed (missing fields, bad IDs, missing branches)
- Emit the file tree shown in [What the generator produces](#what-the-generator-produces)
- Escape user input safely for HTML, Nunjucks attributes and JavaScript strings

---

## The spec Claude returns

Claude returns a single JSON object. This is the shape.

```json
{
  "serviceName": "Apply for free school meals",
  "referencePrefix": "FSM",
  "startPage": {
    "heading": "Apply for free school meals",
    "description": "Use this service to apply for free school meals for your child.",
    "whatYouNeed": ["your National Insurance number", "your child's date of birth"],
    "timeToComplete": "5 minutes"
  },
  "questions": [
    {
      "id": "child-age",
      "type": "radio",
      "question": "How old is your child?",
      "hint": null,
      "options": [
        { "text": "Under 5", "value": "under-5", "next": "ineligible" },
        { "text": "5 to 16", "value": "school-age", "next": "parent-benefits" }
      ],
      "validation": "Select your child's age.",
      "ineligibleReason": "This service is only for children aged 5 to 16."
    }
  ],
  "checkAnswersHeading": "Check your answers before sending",
  "confirmationHeading": "Application submitted",
  "confirmationBody": "We have received your application.",
  "confirmationTimeframe": "We will be in touch within 5 working days."
}
```

### How branching works

Every radio option has a `next` field. It tells the generator where to send the user.

- A question ID — go to that question
- `"check-answers"` — skip to the check your answers page
- `"ineligible"` — send the user to an ineligibility page for this question

Text and textarea questions do not have options. They always go to the next question in order.

At least one option somewhere in the form should branch to `"ineligible"`. This gives the prototype a realistic eligibility check.

---

## What the generator produces

For every prototype, the generator writes these files:

```
package.json
app/config.json
app/routes.js
app/views/layouts/main.html
app/views/start.html
app/views/<question-id>.html        (one per question)
app/views/ineligible-<id>.html      (only where a question branches to ineligible)
app/views/check-answers.html
app/views/confirmation.html
app/assets/sass/application.scss
```

### Key things the generated files must do

**`package.json`** declares two dependencies and nothing else:

```json
{
  "dependencies": {
    "govuk-prototype-kit": "^13.16.2",
    "govuk-frontend": "^5.10.0"
  },
  "scripts": {
    "start": "node node_modules/govuk-prototype-kit/listen-on-port.js"
  },
  "engines": { "node": "22.x" }
}
```

**`app/routes.js`** uses the kit's router:

```js
const govukPrototypeKit = require('govuk-prototype-kit')
const router = govukPrototypeKit.requests.setupRouter()
```

It then adds a GET and POST handler for each question, with server-side validation and branching logic based on the radio answers.

**`app/views/layouts/main.html`** extends the kit's branded layout:

```
{% extends "govuk-prototype-kit/layouts/govuk-branded.njk" %}
```

It imports the GDS macros the question pages need (`govukRadios`, `govukInput`, `govukTextarea`, `govukButton`, `govukErrorSummary`, `govukSummaryList`, `govukBackLink`).

**Each question page** uses the right GDS component for the question type, shows an error summary when there is an error, and posts back to itself.

---

## The integrations

### Claude

- Endpoint: `POST https://api.anthropic.com/v1/messages`
- Model: a current Claude Sonnet
- Headers: `x-api-key`, `anthropic-version: 2023-06-01`, `Content-Type: application/json`
- System prompt: the one in `gds-prompt-v2.js`
- User message: the designer's brief, plus the PDF summary if there is one

Tell Claude to return only the JSON object. Then `JSON.parse` it. Validate it. Patch what is missing.

### GitHub

- Create a new repo with `POST /user/repos`
- Push each file with `PUT /repos/{owner}/{repo}/contents/{path}` (file content must be base64 encoded)
- Use a small delay between pushes to stay under rate limits

### Render

- Create a new web service with `POST /v1/services`
- Set `type: web_service`, `env: node`, `buildCommand: npm install`, `startCommand: npm start`
- Poll the prototype URL until it responds (usually 60 to 180 seconds)
- Return the URL to the designer

---

## Environment variables

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | For calling Claude |
| `GITHUB_TOKEN` | For creating repos and pushing files |
| `GITHUB_USERNAME` | The account that owns the new repos |
| `RENDER_API_KEY` | For creating Render services |
| `RENDER_OWNER_ID` | The Render account or team ID |
| `PORT` | Optional. Render sets this automatically |

---

## Why Render

The generated prototype is a long-running Node server. The GOV.UK Prototype Kit is an Express app that holds state in a session. It is not a static site and it is not a serverless function.

This means:

- **Netlify does not work.** It only hosts static sites.
- **Vercel is a poor fit.** Its serverless model breaks the kit's session and persistent process assumptions. You would have to refactor the kit.
- **Render works well.** It runs `npm install && npm start` on a long-lived web service. The free tier is enough to start with.

If you want to use a different host, pick one that runs Node as a persistent process. Fly.io, Railway and a small VPS all work.

---

## Build order

A suggested order if you are starting from scratch.

1. **Skeleton.** Express app with a single `/` route that shows the brief form.
2. **Claude call.** Add `POST /generate` that takes the brief, calls Claude with the GOV.UK prompt, and returns the JSON in the response.
3. **Generator.** Add a module that turns the JSON into the file map. Test it by writing files to a local folder.
4. **Local preview.** Run the generated folder locally with `npm install && npm start` and check the prototype works end to end.
5. **GitHub push.** Add the GitHub repo creation and file push.
6. **Render deploy.** Add the Render service creation and polling.
7. **PDF upload.** Add Multer, pdf-parse, and the PDF summarisation Claude call.
8. **Progress UI.** Add the progress steps on the frontend so the designer can see what is happening.

Steps 1 to 4 get you a working tool. Steps 5 to 8 make it shareable.

---

## Content rules for generated prototypes

Bake these into the Claude prompt. They are the same rules the GDS service manual uses.

- Plain English. Reading age 9.
- Maximum 20 words per sentence.
- Active voice.
- "You" for the user.
- Labels ask the question. Hints give the format or an example.
- Error messages say what went wrong and how to fix it. Never use "please", "sorry", "invalid" or "this field is required".
- The continue button always says "Continue". Never "Next" or "Submit".

---

## Things to avoid

- **Do not let Claude write code.** Keep it on the JSON spec. The generator writes the code. This is what stops the prototypes drifting from the kit.
- **Do not skip validation.** Claude will sometimes return malformed JSON or miss a field. Patch it with safe defaults rather than crashing.
- **Do not embed user input directly into HTML, JS or Nunjucks.** Use the escape helpers (`htmlStr`, `jsStr`, `njkAttr`) on every string that came from Claude or the designer.
- **Do not put auth on the host app last.** Anyone with the URL can burn your API credits. Add a shared password early.
- **Do not put the whole app in one file.** The original `server.js` is 2,000 lines and mixes HTML, CSS, JavaScript and server logic. Split the routes, the prompt and the generator from day one.

---

## What is out of scope for v1

You do not need any of this to ship the first version.

- Editing existing prototypes
- User accounts
- Versioning of prototypes
- A design history
- Research synthesis features

Build the one-shot flow first. Add the rest later.
