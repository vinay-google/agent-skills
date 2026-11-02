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

### **Canonical Response Format (`hostAppDataAction`)**
* **Important:** Do **NOT** populate both `text` and `cardsV2` with the same content in `message`—Google Chat renders both, which duplicates the response. Use **either** a text-only response (for simple greetings/echoes) **or** a card-only response (for structured commands, summaries, and Gemini AI responses):

* **Text-only response:**
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

* **Card-only response (recommended for Gemini AI responses, `/ask`, and `Summarize message`):**
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
                  subtitle: "Powered by Gemini"
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

## **4. Gemini API on the Gemini Enterprise Agent Platform**

When asked to add AI capabilities in Phase 3:
* Call the **Gemini API on the Gemini Enterprise Agent Platform** (`aiplatform.googleapis.com`) using `UrlFetchApp.fetch` with model **`gemini-3.8-flash`**.
* Endpoint: `https://aiplatform.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/publishers/google/models/gemini-3.8-flash:generateContent`
* Headers:
  * `'Authorization': 'Bearer ' + ScriptApp.getOAuthToken()`
  * `'x-goog-user-project': PROJECT_ID`
* Always wrap API calls in `try / catch` and return a helpful message if an error occurs.
