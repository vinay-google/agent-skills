---
name: workspace-studio-node-addon
description: >-
  End-to-end automation skill for building, deploying, and upgrading HTTP
  Google Workspace Studio Add-ons (Custom Starter Steps and Custom Action
  Steps) in Node.js on Cloud Run with Firestore and Gemini Enterprise Agent
  Platform.
---

# Google Workspace Studio HTTP Add-on Skill (Node.js + Firestore)

You are an expert Google Workspace Studio developer operating in Google
Cloud Shell. Whenever the user asks you to build, deploy, or upgrade the
Google Workspace Studio extension, automatically execute all required CLI
steps (enabling APIs, provisioning the Firestore database, writing code with
Firestore collections, deploying to Cloud Run, granting IAM roles, and
installing the add-on deployment) so the user doesn't have to run manual
commands.

## 1. Automated End-to-End Build & Deployment Workflow
When asked to build and deploy the extension, execute these steps in order:

1. **Enable Required APIs & Provision Firestore Automatically**:
   - Always begin by enabling all required Google Cloud and Workspace APIs
     and ensuring the `(default)` Firestore database exists:

     ```bash
     gcloud services enable \
       aiplatform.googleapis.com \
       firestore.googleapis.com \
       run.googleapis.com \
       cloudbuild.googleapis.com \
       artifactregistry.googleapis.com \
       appsmarket-component.googleapis.com \
       gsuiteaddons.googleapis.com \
       workspacestudio.googleapis.com

     gcloud firestore databases describe --database="(default)" >/dev/null 2>&1 || \
       gcloud firestore databases create \
         --database="(default)" \
         --location=us-central1 \
         --type=firestore-native \
         --quiet
     ```

2. **Scaffold the Modular Node.js Express App (`package.json`, `firestore.js`, `starter.js`, `action.js`, `portal.js`, and `index.js`)**:
   - Create `package.json` with `"type": "module"` and install `express`,
     `@google-cloud/firestore`, and `@google/genai`.
   - Organize the source into five focused files for readability:
     - `firestore.js`: Initializes the `@google-cloud/firestore` client with
       `ignoreUndefinedProperties: true`, exports `triggersCollection`
       (`studio_triggers`) and `ticketsCollection` (`support_tickets`), and
       exports `refreshTokenInFirestore(req)`.
     - `starter.js`: Exports `onConfigTrigger` (`POST /onConfigTrigger`) and
       `onManageTrigger` (`POST /onManageTrigger`) for the `workflowTrigger`
       starter step.
     - `action.js`: Exports `onConfigUrgency` (`POST /onConfigUrgency`) and
       `onExecuteUrgency` (`POST /onExecuteUrgency`) for the `workflowAction`
       step, calling `gemini-3.8-flash` on the Gemini Enterprise Agent Platform
       with structured enum output (`["High", "Normal"]`) and a
       `detectKeywordUrgency()` fallback (`urgent`, `blocker`, `critical`).
     - `portal.js`: Exports `renderPortal` (`GET /`) and `submitTicket`
       (`POST /api/submit-ticket`), firing active triggers via `triggers.fire`
       and logging tickets to `support_tickets`.
     - `index.js`: Wires all Express routes (`GET /`, `POST /api/submit-ticket`,
       `POST /onConfigTrigger`, `POST /onManageTrigger`,
       `POST /onConfigUrgency`, `POST /onExecuteUrgency`) and listens on
       `process.env.PORT || 8080`.

3. **Deploy to Cloud Run**:
   - Deploy as `support-extension` in `us-central1` with the project and
     Firestore collection environment variables:

     ```bash
     gcloud run deploy support-extension \
       --source . \
       --region us-central1 \
       --allow-unauthenticated \
       --set-env-vars="GOOGLE_CLOUD_PROJECT=$(gcloud config get-value project),FIRESTORE_TRIGGERS_COLLECTION=studio_triggers,FIRESTORE_TICKETS_COLLECTION=support_tickets" \
       --quiet
     ```

4. **Configure IAM Permissions Automatically**:
   - Fetch the Google Workspace Add-ons service account and grant it
     `roles/run.invoker` on the Cloud Run service so Google Workspace Studio
     can invoke the webhook endpoints:

     ```bash
     ADDON_SA=$(gcloud workspace-add-ons get-authorization --format="value(serviceAccountEmail)")
     gcloud run services add-iam-policy-binding support-extension \
       --region=us-central1 \
       --member="serviceAccount:${ADDON_SA}" \
       --role="roles/run.invoker" \
       --quiet
     ```

   - Grant the Cloud Run runtime service account `roles/datastore.user` (to
     read and write the `studio_triggers` and `support_tickets` Firestore
     collections) and `roles/aiplatform.user` (to call Gemini on the Gemini
     Enterprise Agent Platform):

     ```bash
     PROJECT_ID=$(gcloud config get-value project)
     PROJECT_NUMBER=$(gcloud projects describe "${PROJECT_ID}" --format="value(projectNumber)")
     RUNTIME_SA=$(gcloud run services describe support-extension --region=us-central1 --format="value(spec.template.spec.serviceAccountName)")
     RUNTIME_SA=${RUNTIME_SA:-"${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"}

     gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
       --member="serviceAccount:${RUNTIME_SA}" \
       --role="roles/datastore.user" \
       --condition=None \
       --quiet

     gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
       --member="serviceAccount:${RUNTIME_SA}" \
       --role="roles/aiplatform.user" \
       --condition=None \
       --quiet
     ```

5. **Generate `deployment.json`, Register, and Install the Add-on**:
   - Retrieve the deployed Cloud Run URL:

     ```bash
     SERVICE_URL=$(gcloud run services describe support-extension --region=us-central1 --format="value(status.url)")
     ```

   - Write `deployment.json` (Section 2) using `${SERVICE_URL}`.
   - Create (or replace if it already exists) and install the add-on
     deployment:

     ```bash
     gcloud workspace-add-ons deployments create support-extension --deployment-file=deployment.json || \
       gcloud workspace-add-ons deployments replace support-extension --deployment-file=deployment.json
     gcloud workspace-add-ons deployments install support-extension
     ```

6. **Report the Portal URL**:
   - Clearly print `${SERVICE_URL}` so the user can open the Support Quick
     Portal in their browser.

## 2. HTTP Add-on Deployment Manifest (`deployment.json`)
Every webhook URL in `deployment.json` MUST use `https://` and include a
non-empty path:

```json
{
  "oauthScopes": [
    "https://www.googleapis.com/auth/workspace.studio.trigger",
    "https://www.googleapis.com/auth/userinfo.email"
  ],
  "addOns": {
    "common": {
      "name": "Support Extension",
      "logoUrl": "https://www.gstatic.com/images/branding/productlogos/buganizer/v1/192px.svg"
    },
    "studio": {
      "flows": {
        "workflowElements": [
          {
            "id": "supportTicketTrigger",
            "state": "ACTIVE",
            "name": "New Support Ticket",
            "description": "Fires when someone submits a ticket in the Support Quick Portal.",
            "workflowTrigger": {
              "inputs": [],
              "outputs": [
                {
                  "id": "ticketTitle",
                  "description": "Ticket Title",
                  "cardinality": "SINGLE",
                  "dataType": { "basicType": "STRING" }
                },
                {
                  "id": "ticketDescription",
                  "description": "Ticket Description",
                  "cardinality": "SINGLE",
                  "dataType": { "basicType": "STRING" }
                }
              ],
              "onConfigFunction": "https://YOUR_CLOUD_RUN_URL/onConfigTrigger",
              "onManageFunction": "https://YOUR_CLOUD_RUN_URL/onManageTrigger"
            }
          },
          {
            "id": "urgencyDetectorStep",
            "state": "ACTIVE",
            "name": "Detect Urgency",
            "description": "Evaluates the ticket description to detect High or Normal urgency.",
            "workflowAction": {
              "inputs": [
                {
                  "id": "ticketDescription",
                  "description": "Description to scan",
                  "cardinality": "SINGLE",
                  "dataType": { "basicType": "STRING" }
                }
              ],
              "outputs": [
                {
                  "id": "urgencyLevel",
                  "description": "Detected Urgency (High / Normal)",
                  "cardinality": "SINGLE",
                  "dataType": { "basicType": "STRING" }
                }
              ],
              "onConfigFunction": "https://YOUR_CLOUD_RUN_URL/onConfigUrgency",
              "onExecuteFunction": "https://YOUR_CLOUD_RUN_URL/onExecuteUrgency"
            }
          }
        ]
      }
    },
    "httpOptions": {
      "authorizationHeader": "SYSTEM_ID_TOKEN"
    }
  }
}
```

## 3. Firestore Setup & Webhook Endpoint Schemas (`RenderActions` JSON)
CRITICAL:
- Do NOT use local temp files (such as `/tmp/triggers.json`). Always persist
  triggers and tickets in Firestore:

  ```javascript
  import { Firestore } from '@google-cloud/firestore';
  const db = new Firestore({
    projectId: process.env.GOOGLE_CLOUD_PROJECT || undefined,
    databaseId: '(default)',
    ignoreUndefinedProperties: true,
  });
  const triggersCollection = db.collection(
    process.env.FIRESTORE_TRIGGERS_COLLECTION || 'studio_triggers'
  );
  const ticketsCollection = db.collection(
    process.env.FIRESTORE_TICKETS_COLLECTION || 'support_tickets'
  );
  ```

- Every HTTP webhook endpoint called by Google Workspace Studio MUST return
  HTTP 200 with a JSON object body (`res.status(200).json(...)`). Returning
  an empty HTTP body causes an `EmptyHttpBody` error.

### A. `POST /onConfigTrigger` (Starter Configuration Card)

```json
{
  "action": {
    "navigations": [
      {
        "pushCard": {
          "sections": [
            {
              "header": "Support Ticket Listener",
              "widgets": [
                {
                  "textParagraph": {
                    "text": "This starter listens for new tickets submitted through the Support Quick Portal and stores subscriptions in Firestore."
                  }
                }
              ]
            }
          ]
        }
      }
    ]
  }
}
```

### B. `POST /onManageTrigger` (Starter Subscription Lifecycle in Firestore)
- When a user turns on a flow in Google Workspace Studio, Google sends:
  - `req.body.workflow.triggerCreation`: `{ triggerId, notifyUri, inputs }`
  - `req.body.authorizationEventObject.userOAuthToken`: OAuth 2.0 access
    token authorized with the `workspace.studio.trigger` scope.
- When a flow is turned off or deleted, Google sends
  `req.body.workflow.triggerDeletion`: `{ triggerId }`.
- Persist active triggers in the `studio_triggers` Firestore collection
  using `encodeURIComponent(String(triggerId))` as the document ID so
  special characters in `triggerId` are always safe:

  ```javascript
  const triggerCreation = req.body?.workflow?.triggerCreation;
  const triggerDeletion = req.body?.workflow?.triggerDeletion;
  const userOAuthToken = req.body?.authorizationEventObject?.userOAuthToken || null;

  if (triggerCreation?.triggerId) {
    const triggerId = String(triggerCreation.triggerId);
    const docId = encodeURIComponent(triggerId);
    await triggersCollection.doc(docId).set(
      {
        triggerId,
        notifyUri: triggerCreation.notifyUri || null,
        ...(userOAuthToken ? { userOAuthToken } : {}),
        updatedAt: new Date().toISOString(),
      },
      { merge: true }
    );
  }

  if (triggerDeletion?.triggerId) {
    const triggerId = String(triggerDeletion.triggerId);
    const docId = encodeURIComponent(triggerId);
    await triggersCollection.doc(docId).delete();
  }
  ```

- Whenever any webhook receives a fresh
  `req.body.authorizationEventObject?.userOAuthToken`, update
  `userOAuthToken` across existing documents in `studio_triggers` using a
  Firestore batch write so the cached token stays current.
- Return `res.status(200).json({})`.

### C. `POST /onConfigUrgency` (Action Step Configuration Card)
Return a `pushCard` navigation object containing a `textInput` widget whose
`name` matches `"ticketDescription"` and enables the Google Workspace Studio
variable picker via `hostAppDataSource.workflowDataSource.includeVariables`:

```json
{
  "action": {
    "navigations": [
      {
        "pushCard": {
          "sections": [
            {
              "header": "Urgency Detector Step",
              "widgets": [
                {
                  "textInput": {
                    "name": "ticketDescription",
                    "label": "Source Description",
                    "hintText": "Select the description variable outputted by the starter",
                    "hostAppDataSource": {
                      "workflowDataSource": {
                        "includeVariables": true
                      }
                    }
                  }
                }
              ]
            }
          ]
        }
      }
    ]
  }
}
```

### D. `POST /onExecuteUrgency` (Action Step Execution)
- Extract the input variable from the request body:

  ```javascript
  const inputs = req.body?.workflow?.actionInvocation?.inputs || {};
  const description = inputs.ticketDescription?.stringValues?.[0] || "";
  ```

- Determine `urgencyLevel` (`"High"` or `"Normal"`):
  - Stage 1 (Keyword-based): Return `"High"` if `description.toLowerCase()`
    contains `"urgent"`, `"blocker"`, or `"critical"`; otherwise return
    `"Normal"`.
  - Stage 2 (Gemini AI upgrade): Call `gemini-3.8-flash` on the Gemini
    Enterprise Agent Platform (`aiplatform.googleapis.com`) to classify the
    ticket description as `"High"` or `"Normal"`.
- Return the output variable using
  `hostAppAction.workflowAction.returnOutputVariablesAction`:

  ```json
  {
    "hostAppAction": {
      "workflowAction": {
        "returnOutputVariablesAction": {
          "variables": {
            "urgencyLevel": {
              "stringValues": ["High"]
            }
          }
        }
      }
    }
  }
  ```

## 4. Support Quick Portal & Firing the Starter (`triggers.fire`)
- Serve a clean, modern HTML form at `GET /` with a Ticket Title input
  (`#title`), an Issue Description textarea (`#description`), a Submit
  Ticket button, and a live status indicator querying
  `await triggersCollection.get()` to show how many active Google Workspace
  Studio triggers are stored in the `studio_triggers` Firestore collection.
- Handle form submissions at `POST /api/submit-ticket`:
  - Query all active trigger documents from the `studio_triggers` Firestore
    collection (`const snapshot = await triggersCollection.get()`).
  - For each document (`const trigger = doc.data()`), send a `POST` request
    to `https://workspacestudio.googleapis.com/v1/triggers/${encodeURIComponent(trigger.triggerId)}:fire`
    with headers `Authorization: Bearer ${trigger.userOAuthToken}` and
    `Content-Type: application/json`, and body:

    ```json
    {
      "name": "triggers/" + trigger.triggerId,
      "outputs": {
        "ticketTitle": { "stringValues": [title] },
        "ticketDescription": { "stringValues": [description] }
      },
      "requestId": "req_" + Date.now() + "_" + Math.floor(Math.random() * 1000)
    }
    ```

  - Save the submitted ticket record in the `support_tickets` Firestore
    collection (`await ticketsCollection.add({ title, description, triggersFired, createdAt: new Date().toISOString() })`).

## 5. Stage 2 Upgrade: Gemini 3.8 Flash on Gemini Enterprise Agent Platform
When asked to upgrade `/onExecuteUrgency` to use Gemini:
- Ensure `@google/genai` is installed and initialize the client for the
  Gemini Enterprise Agent Platform:

  ```javascript
  import { GoogleGenAI, Type } from '@google/genai';
  const ai = new GoogleGenAI({
    vertexai: true,
    project: process.env.GOOGLE_CLOUD_PROJECT,
    location: 'global',
  });
  ```

- Call `ai.models.generateContent` with `model: 'gemini-3.8-flash'` and
  configure structured output (`responseMimeType: 'text/x.enum'` and an enum
  `responseSchema`) so the response is strictly constrained to `'High'` or
  `'Normal'`:

  ```javascript
  const response = await ai.models.generateContent({
    model: 'gemini-3.8-flash',
    contents: `Evaluate the business impact and time sensitivity of this support ticket description and classify its urgency:\n\n${description}`,
    config: {
      responseMimeType: 'text/x.enum',
      responseSchema: {
        type: Type.STRING,
        enum: ['High', 'Normal'],
      },
    },
  });
  ```

- IMPORTANT: Don't pass deprecated or unsupported sampling parameters
  (`temperature`, `topP`, `topK`, `candidateCount`, `frequencyPenalty`,
  `presencePenalty`) to `gemini-3.8-flash`.
- Fall back to keyword matching if an API error occurs.
- Redeploy `support-extension` to Cloud Run with the same Firestore
  environment variables so existing trigger subscriptions in the
  `studio_triggers` collection remain intact across revisions.
