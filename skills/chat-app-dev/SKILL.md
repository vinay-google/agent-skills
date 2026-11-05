---
name: chat-app-dev
description: >-
  Rules and specifications for building an Apps Script Google Chat App and
  managing its configuration using the Google Chat App Config API.
---

# Google Chat App & Chat App Config API Development Skill

You are operating as a methodical **Google Workspace extensibility and integration expert**. Your goal is to build a Google Chat app in the `genai-chat-app` project using Apps Script (`clasp`) and configure it programmatically using the **Google Chat App Config API** (`gsuiteaddons.googleapis.com`).

---

## **1. Environment & Project Discovery**

* You are operating in **Google Cloud Shell** with **`clasp`**, **`gcloud`**, and **`curl`** installed.
* Always dynamically look up the current GCP Project ID, Project Number, and User Email using:
  * `PROJECT_ID=$(gcloud config get-value project)`
  * `PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format="value(projectNumber)")`
  * `USER_EMAIL=$(gcloud config get-value account)`
* Use `clasp create --type standalone --title "GenAI Chat App"` to initialize the Apps Script project, `clasp push -f` to push code changes, and `clasp deployments` to retrieve the `@HEAD` Deployment ID (the string before `@HEAD`).

---

## **2. Google Chat App Config API (`gsuiteaddons.googleapis.com`)**

Never ask the user to manually configure the Chat app in the Google Cloud console UI. Always manage the Chat app configuration using REST calls to the **Google Workspace Add-ons API**:

* **Base Resource Name:** `projects/${PROJECT_NUMBER}/chatAppConfig`
* **Endpoint URL:** `https://gsuiteaddons.googleapis.com/v1/projects/${PROJECT_NUMBER}/chatAppConfig`
* **Required Headers for `curl`:**
  * `-H "Authorization: Bearer $(gcloud auth print-access-token)"`
  * `-H "x-goog-user-project: ${PROJECT_NUMBER}"`
  * `-H "Content-Type: application/json"`

### **Methods**
1. **`GetChatAppConfig` (`GET`)**:
   * `GET https://gsuiteaddons.googleapis.com/v1/projects/${PROJECT_NUMBER}/chatAppConfig?view=FULL`
   * Always pass `view=FULL` to inspect `interactiveFeatureConfig` (connection settings, app commands, URL patterns, and allowlisted testing principals).
2. **`UpdateChatAppConfig` (`PATCH`)**:
   * `PATCH https://gsuiteaddons.googleapis.com/v1/projects/${PROJECT_NUMBER}/chatAppConfig`
   * Acts as an **upsert**: creates the Chat app configuration if none exists on the project, or updates the existing configuration.
   * Supports optional query parameter `?updateMask=<comma-separated-field-paths>` (for example, `?updateMask=interactiveFeatureConfig.appCommands,interactiveFeatureConfig.urlPatterns`). Note that updating repeated fields (`appCommands`, `urlPatterns`, `allowlistedTestingPrincipals`) replaces the entire list for that field.

### **`ChatAppConfig` JSON Schema & Validation Constraints**

Example `ChatAppConfig` JSON payload:

```json
{
  "name": "projects/${PROJECT_NUMBER}/chatAppConfig",
  "displayName": "GenAI Chat Assistant",
  "description": "AI-powered Google Chat helper app",
  "avatarUri": "https://developers.google.com/workspace/add-ons/images/quickstart-app-avatar.png",
  "state": "ACTIVE",
  "errorLoggingEnabled": true,
  "markupSyntax": "MARKUP_SYNTAX_MARKDOWN",
  "interactiveFeatureConfig": {
    "supportsMultiuserSpaces": true,
    "allowlistedTestingPrincipals": [
      "${USER_EMAIL}"
    ],
    "connectionConfig": {
      "appsScriptEndpointConfig": {
        "deploymentId": "${HEAD_DEPLOYMENT_ID}",
        "perEventEndpointConfig": {
          "messageEndpoint": "onMessage",
          "addedToSpaceEndpoint": "onAddedToSpace",
          "removedFromSpaceEndpoint": "onRemovedFromSpace",
          "appCommandEndpoint": "onAppCommand"
        }
      }
    },
    "appCommands": [
      {
        "commandId": 1,
        "commandType": "QUICK_COMMAND",
        "displayName": "Help",
        "description": "Show available commands and features",
        "dialogEnabled": false
      },
      {
        "commandId": 2,
        "commandType": "SLASH_COMMAND",
        "command": "/ask",
        "description": "Ask a question",
        "dialogEnabled": false
      },
      {
        "commandId": 3,
        "commandType": "MESSAGE_ACTION",
        "displayName": "Summarize message",
        "description": "Summarize the selected message",
        "loadingNotificationMessage": "Summarizing message...",
        "dialogEnabled": false
      }
    ],
    "urlPatterns": [
      {
        "hostPattern": "example.com",
        "pathPrefix": "cases/"
      }
    ]
  }
}
```

* **Strict Field Validation Rules:**
  * `displayName`: Required on create, **3 to 25 characters**, no leading/trailing spaces.
  * `description`: Required on create, **1 to 40 characters** (must NOT exceed 40 characters!).
  * `avatarUri`: Valid HTTPS URL.
  * `appCommands`:
    * `commandId`: Integer `1` to `1000`, unique across all commands.
    * `description`: Non-empty, max **50 characters**.
    * For `QUICK_COMMAND`: Set `displayName` (max 50 chars). Do NOT set `command`.
    * For `SLASH_COMMAND`: Set `command` starting with `/` followed by 2 to 49 characters (no spaces). Do NOT set `displayName`.
    * For `MESSAGE_ACTION`: Set `displayName` (max 50 chars) and optional `loadingNotificationMessage` (max 50 chars).
  * `urlPatterns`:
    * `hostPattern`: Valid domain (e.g., `"example.com"` or `"*.example.com"`).
    * `pathPrefix`: Must NOT start with `/`, and MUST end with `/` or be empty `""` (e.g., `"cases/"`).

---

## **3. Apps Script Google Workspace Add-on for Chat (`Chaddon`) Architecture**

Apps configured using the Chat App Config API are **Google Workspace Add-ons that extend Google Chat**. You MUST follow the Workspace Add-ons for Chat manifest, event object, and response formats:

### **Manifest (`appsscript.json`)**
Always include `"addOns": { "common": { ... }, "chat": {} }` and required OAuth scopes:

```json
{
  "timeZone": "America/New_York",
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8",
  "oauthScopes": [
    "https://www.googleapis.com/auth/script.external_request",
    "https://www.googleapis.com/auth/cloud-platform"
  ],
  "addOns": {
    "common": {
      "name": "GenAI Chat Assistant",
      "logoUrl": "https://developers.google.com/workspace/add-ons/images/quickstart-app-avatar.png"
    },
    "chat": {}
  }
}
```

### **Event Object Structure (`event.chat`)**
* **`onMessage(event)`**:
  * Message object: `const message = event.chat.messagePayload.message;`
  * Message text: `message.text` (or `message.argumentText` for clean text without `@mention` or slash command prefix)
  * Space object: `const space = event.chat.messagePayload.space;`
  * User object: `const user = event.chat.user;`
  * Link preview URL (when a message matches `urlPatterns`): `message.matchedUrl && message.matchedUrl.url`
* **`onAddedToSpace(event)`**:
  * Space object: `const space = event.chat.addedToSpacePayload.space;`
  * User object: `const user = event.chat.user;`
* **`onRemovedFromSpace(event)`**:
  * Space object: `const space = event.chat.removedFromSpacePayload.space;`
* **`onAppCommand(event)`**:
  * Payload: `const payload = event.chat.appCommandPayload;`
  * Metadata: `const { appCommandId, appCommandType } = payload.appCommandMetadata;`
  * Note: `appCommandType` is `"QUICK_COMMAND"`, `"SLASH_COMMAND"`, or `"MESSAGE_ACTION"`.
  * For `SLASH_COMMAND`, the user's message/arguments are in `payload.message.argumentText` (or `payload.message.text`).
  * For `MESSAGE_ACTION`, the target message that the user invoked the action on is in `payload.message.text`.

* **`onSystemCommand(event)`** (called when a user clicks **Stop** on an active streaming message):
  * Cancelled stream resource name: `const streamMessage = event.chat && event.chat.systemCommandPayload && event.chat.systemCommandPayload.systemCommandMetadata && event.chat.systemCommandPayload.systemCommandMetadata.cancelStream && event.chat.systemCommandPayload.systemCommandMetadata.cancelStream.streamMessage;`

### **Canonical Response Format (`hostAppDataAction`)**
* **Important:**
  * When replying synchronously (Phase 1 echo, Phase 2 `Help` quick command, or Phase 2 error fallbacks), return a `hostAppDataAction` object. Do **NOT** populate both `text` and `cardsV2` with the same content in `message`—Google Chat renders both, which duplicates the response.
  * When streaming a reply asynchronously using `spaces.streamMessages` (Phase 3 DMs, `/ask`, and `Summarize message`, or `onSystemCommand`), return an empty object `{}` from the Apps Script handler because the message is created and committed through the Chat API.

* **Text-only synchronous response:**
```javascript
return {
  hostAppDataAction: {
    chatDataAction: {
      createMessageAction: {
        message: {
          text: "Your message text here"
        }
      }
    }
  }
};
```

* **Card-only synchronous response (for `Help` command or structured cards):**
```javascript
return {
  hostAppDataAction: {
    chatDataAction: {
      createMessageAction: {
        message: {
          cardsV2: [
            {
              cardId: "response-card",
              card: {
                header: {
                  title: "GenAI Chat Assistant",
                  subtitle: "Available Commands"
                },
                sections: [
                  {
                    widgets: [
                      {
                        textParagraph: {
                          text: "Card body text",
                          textSyntax: "MARKDOWN"
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
    }
  }
};
```

* To attach a **Link Preview Card** when `event.chat.messagePayload.message.matchedUrl` is present in `onMessage(event)`, return `updateInlinePreviewAction`:

```javascript
return {
  hostAppDataAction: {
    chatDataAction: {
      updateInlinePreviewAction: {
        cardsV2: [
          {
            cardId: "link-preview-card",
            card: {
              header: {
                title: "Case Preview",
                subtitle: matchedUrl
              },
              sections: [
                {
                  widgets: [
                    {
                      textParagraph: {
                        text: "Preview details for " + matchedUrl
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
  }
};
```

---

## **4. Streaming Gemini AI Responses with the Google Chat API (`spaces.streamMessages`)**

When asked to add AI capabilities and message streaming in Phase 3:

### **A. App Authentication (`https://www.googleapis.com/auth/chat.bot`) via Keyless Service Account Impersonation**
Message streaming (`spaces.streamMessages`) requires **app authentication** with the `https://www.googleapis.com/auth/chat.bot` scope (user OAuth tokens are not supported). Never export a JSON private key (`credentials.json`); instead, use keyless service account impersonation via the **IAM Service Account Credentials API** (`iamcredentials.googleapis.com`):

1. In Cloud Shell, create a service account in the project and grant the user `roles/iam.serviceAccountTokenCreator`:
   ```bash
   gcloud services enable iamcredentials.googleapis.com
   gcloud iam service-accounts create chat-app-sa --display-name="Chat App Service Account" || true
   gcloud iam service-accounts add-iam-policy-binding "chat-app-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
     --member="user:${USER_EMAIL}" \
     --role="roles/iam.serviceAccountTokenCreator"
   ```
2. In `Code.gs`, mint a short-lived `chat.bot` access token on demand using `ScriptApp.getOAuthToken()`:
   ```javascript
   function getChatAppAccessToken() {
     const saEmail = `chat-app-sa@${PROJECT_ID}.iam.gserviceaccount.com`;
     const url = `https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/${encodeURIComponent(saEmail)}:generateAccessToken`;
     const response = UrlFetchApp.fetch(url, {
       method: 'post',
       contentType: 'application/json',
       headers: { Authorization: 'Bearer ' + ScriptApp.getOAuthToken() },
       payload: JSON.stringify({
         scope: ['https://www.googleapis.com/auth/chat.bot']
       })
     });
     return JSON.parse(response.getContentText()).accessToken;
   }
   ```

### **B. Gemini API on the Gemini Enterprise Agent Platform (`gemini-3.8-flash`)**
* Call the **Gemini API on the Gemini Enterprise Agent Platform** (`aiplatform.googleapis.com`) using `UrlFetchApp.fetch` with model **`gemini-3.8-flash`**.
* Endpoint: `https://aiplatform.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/publishers/google/models/gemini-3.8-flash:generateContent`
* Headers:
  * `'Authorization': 'Bearer ' + ScriptApp.getOAuthToken()`
  * `'x-goog-user-project': PROJECT_ID`

### **C. Streaming Messages Lifecycle (`spaces.streamMessages`)**
For direct messages (`onMessage`), `/ask` (`SLASH_COMMAND` ID `2`), and `Summarize message` (`MESSAGE_ACTION` ID `3`), stream the response incrementally into the thread of the triggering message (`referencedMessage` = `message.name`, e.g., `spaces/{space}/messages/{message}`):

1. **Start a stream (`spaces.streamMessages.create`):**
   * `POST https://chat.googleapis.com/v1/${spaceName}/streamMessages`
   * Headers: `'Authorization': 'Bearer ' + appAccessToken`, `'Content-Type': 'application/json'`
   * Body:
     ```json
     {
       "referencedMessage": "spaces/SPACE_ID/messages/MESSAGE_ID",
       "progressText": "Thinking...",
       "markupSyntax": "MARKUP_SYNTAX_MARKDOWN"
     }
     ```
   * Save `streamName = stream.name` (`spaces/{space}/streamMessages/{stream_message}`) and `chunkToken = stream.nextChunkToken`.

2. **Append intermediate steps and text chunks (`spaces.streamMessages.appendStreamMessageContent`):**
   * `POST https://chat.googleapis.com/v1/${streamName}:appendStreamMessageContent`
   * Pass the latest `chunkToken` and update `chunkToken = response.nextChunkToken` after each call:
     * Before calling Gemini, append an initial reasoning step and update `progressText`:
       ```json
       {
         "chunkToken": "CURRENT_CHUNK_TOKEN",
         "progressText": "Querying Gemini 3.8 Flash...",
         "streamContentChunks": [
           {
             "step": {
               "title": "Analyze request",
               "description": "Preparing prompt for Gemini 3.8 Flash."
             }
           }
         ]
       }
       ```
     * After Gemini returns the generated Markdown text, stream a completion `step` and the `text` chunks (for example, splitting paragraphs or sections across `appendStreamMessageContent` calls):
       ```json
       {
         "chunkToken": "LATEST_CHUNK_TOKEN",
         "progressText": "Streaming response...",
         "streamContentChunks": [
           {
             "step": {
               "title": "Synthesize response",
               "description": "Generated answer with Gemini 3.8 Flash."
             }
           },
           {
             "text": "Generated Markdown text chunk...\n\n"
           }
         ]
       }
       ```

3. **Finalize the stream (`spaces.streamMessages.commit`):**
   * `POST https://chat.googleapis.com/v1/${streamName}:commit`
   * Body:
     ```json
     {
       "chunkToken": "LATEST_CHUNK_TOKEN"
     }
     ```
   * Return `{}` from `onMessage` / `onAppCommand` once the stream is committed so Google Chat does not post a duplicate synchronous reply.

4. **Handle errors & user-initiated stream cancellation (`spaces.streamMessages.delete` & `onSystemCommand`):**
   * If an error occurs after a stream has started, call `DELETE https://chat.googleapis.com/v1/${streamName}` (or commit an error message) so the stream does not hang for the 5-minute timeout.
   * Implement `onSystemCommand(event)` to handle the **Stop** button (`event.chat.systemCommandPayload.systemCommandMetadata.cancelStream.streamMessage`), calling `DELETE https://chat.googleapis.com/v1/${streamMessage}` (or committing partial output) and returning `{}`.
