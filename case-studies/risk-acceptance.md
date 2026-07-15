# Case Study #01: End-to-End Automated Risk Acceptance Workflow 🛡️⚡
> **How I automated a mountain of manual compliance work into a zero-touch, self-healing SecOps pipeline.**

![Integrations](https://img.shields.io/badge/Integrations-Jira%20%7C%20AWS%20Lambda%20%7C%20Google%20Workspace%20%7C%20DocuSign-blue?style=for-the-badge)
![Type](https://img.shields.io/badge/Type-Enterprise%20Integration-success?style=for-the-badge)

---

## 📌 Background & The Pain Point: "The SLA Avalanche"

Let’s be honest: security scanners do their job *too* well. 

At the scale of the high-growth tech company I currently work for, I was facing a classic security team nightmare: **thousands of open vulnerabilities** with expired Service Level Agreements (SLAs) assigned to busy engineering teams. 

Standard security compliance dictated that every single one of these bypasses required a formal, legally binding **Risk Acceptance Document** signed by the respective engineering manager, director, and occasionally C-levels.

### The Old Way (Or: "How to crush an analyst's soul")
1. **Manual Creation:** An analyst would find an expired SLA in Jira, open a Google Doc template, and copy-paste hostnames, CVE details, and ticket numbers. (A single typo in the CVE? Congratulations, start over).
2. **The Detective Work:** The analyst had to play corporate detective to find out: *"Who owns this microservice? Who is their manager? Who is the current Director of this Business Unit?"*
3. **The Email Limbo:** Upload the PDF to DocuSign, send it, and pray it didn't sink into an inbox black hole.
4. **The Chasing Loop:** Spend 40% of the week sending Google Chat messages saying, *"Hey, sorry to bother you, but could you sign that DocuSign from last Tuesday?"*
5. **The Archival Nightmare:** Once signed, manually download the PDF and the certificate, zip them, and upload them to a specific Google Drive folder for audits.

**My Goal:** Eliminate human touch from the entire lifecycle. From SLA breach detection to final archival, the machine should do the work.

---

## 🏗️ The Architecture (The Big Picture)

Here is the architectural flow of the automated Risk Acceptance engine I designed and implemented. 

To keep the system highly decoupled and stable, I separated the engine into two distinct, single-purpose workflows: **CreateDoc** (the creator) and **DocuSign Signal Receiver** (the event listener).

```mermaid
sequenceDiagram
    autonumber
    participant Jira as Jira Cloud
    participant WF as Workflow (Risk Acceptance - CreateDoc)
    participant Lambda as AWS Lambda (Auth & Token)
    participant Google as Google APIs (Docs/Admin)
    participant Drive as Google Drive
    participant DB as Temp DB (Astroflow)
    participant DocuSign as DocuSign (App & Rules)
    participant Cron as Daily Cron Workflow (Astroflow)
    participant Aegis as Bot Aegis (Google Chat Gateway)
    participant WF_Recv as Workflow (DocuSign Signal Receiver)

    Jira->>WF: 1. Trigger Webhook (Vulnerability past SLA)
    rect rgb(240, 248, 255)
        note right of WF: Sub-workflow: Risk Acceptance - GetSigners
        WF->>Google: Query responsible manager/director
        Google-->>WF: Return Org Details & Emails
        WF->>DB: Save Signers & Info
    end
    WF->>Lambda: 2. Fetch Signed JWT for Google
    Lambda-->>WF: Return Google Access Token
    WF->>Google: 3. Clone Risk Acceptance Template (Drive API)
    WF->>WF: 4. JS Payload Compiler (create_googledoc_payload)
    WF->>Google: 5. Populate Template with Vulnerability Details
    WF->>Lambda: 6. Request Doc PDF as Base64
    Lambda-->>WF: Return Base64 PDF
    WF->>Lambda: 7. Fetch Signed JWT for DocuSign
    Lambda-->>WF: Return DocuSign Token
    WF->>DocuSign: 8. Upload Doc & Route Envelopes (routing_order)
    DocuSign-->>WF: Envelope Sent Successfully (CreateDoc finishes its job!)
    
    rect rgb(255, 240, 245)
        note right of Cron: Daily Signature Chaser
        Cron->>DocuSign: 9. Check Pending Signatures
        DocuSign-->>Cron: Return Pending Signers
        Cron->>Aegis: 10. Dispatch custom notification payload
        Aegis->>Aegis: 11. Route message to Google Chat API
    end
    
    DocuSign->>WF_Recv: 12. Webhook Callback (Envelope Signed)
    WF_Recv->>Jira: 13. Call Jira Automation Webhook (Pass issue key in URL)
    
    rect rgb(240, 255, 240)
        note right of DocuSign: Native Agreement Action (Rule)
        DocuSign->>Drive: 14. Auto-save ZIP (Doc + Cert) via Connection Rule
    end
```

---

## 🛠️ The Tech Stack I Integrated

* **Jira Cloud Automation:** Monitors ticket queues, SLA states, and dispatches JSON payloads.
* **Astroflow:** An in-house developed workflow automation platform (similar to `n8n` but lightweight and proprietary) that serves as my central brain and logical orchestrator.
* **AWS Lambda:** My serverless cryptographic gatekeeper, dynamically generating signed JWTs for both Google and DocuSign endpoints.
* **Google Workspace Suite (Docs, Drive, Admin Directory & Chat APIs):** Automated the document lifecycle, organizational charting, and chat notifications.
* **DocuSign REST API:** Standardizing legally binding e-signatures with custom routing logic.

---

## 🔍 Deep Dive: Overcoming Technical Hurdles & Boss Fights

Building an enterprise-grade automation isn't just about plugging APIs together; it's about surviving rate limits, strict auth rules, and picky schema validations. Here are the major challenges I solved:

### 1. The Doc-Hunting Crusade (AI doesn't do magic)
While modern AI assistants were incredibly valuable to quickstart the project and bootstrap the initial boilerplate logic, they don't solve everything by magic. 

The subtle and critical nuances of integrating enterprise-level platforms (especially around custom directory searches, security scopes, and webhook routing) still demanded rolling up my sleeves, diving deep into the official Google Workspace and DocuSign documentation, and manually figuring out how to thread these distinct ecosystems together securely.

### 2. The Service Account Battle: Sustainability > Employee Identity
One of my primary architectural requirements was sustainability. 
* **The Pitfall:** I refused to bind these automations to an individual employee's account (My own, for instance). If that person leaves the company, their account is deactivated, and the entire compliance engine instantly breaks.
* **The Solution:** I established a dedicated, non-human, generic **Service Account** specifically for this pipeline. This decoupled our corporate operational dependencies from physical employees and secured our long-term automation posture.

### 3. The IAM & IT Ops Quest (Because Cyber is not God)
Contrary to popular belief, Cyber Security teams are not omnipotent deities inside an enterprise ecosystem (and we shouldn't be!). 

To set up the proper execution privileges, I had to negotiate and coordinate with our IT Operations and IAM (Identity & Access Management) teams. I defined exactly what scopes were needed, ensuring we followed the **Principle of Least Privilege** while still acquiring the muscle needed to orchestrate Google Workspace resources.

### 4. Domain-Wide Delegation (DWD) & The JWT Impersonation Dance
To allow our service account to dynamically clone and edit Google Docs on behalf of our team, we needed **Domain-Wide Delegation (DWD)**. 

During our Lambda authorization requests, we had to explicitly define the required Drive and Document scopes:
`https://www.googleapis.com/auth/drive https://www.googleapis.com/auth/documents`

This allows our serverless functions to generate a signed JSON Web Token (JWT), pass it to Google's OAuth 2.0 endpoint, and impersonate an authorized administrative email (e.g., `your-automation-account@yourcompany.com`) to handle document mutations seamlessly.

---

## ⚡ Serverless Authentication: The Lambdas

To handle cryptographic signing and securely interact with the APIs without exposing keys, I designed and deployed three micro-lambdas on AWS, leveraging **AWS KMS** to encrypt/decrypt sensitive variables at rest.

### Lambda #1: `your-google-token-authenticator` (Google Token Exchange)
This function decrypts credentials via AWS KMS, generates a signed JWT assertion (RSA-SHA256), and exchanges it for a short-lived Google OAuth Bearer Token. It supports dynamic scopes and impersonation (`sub` claim).

```javascript
import { KMSClient, DecryptCommand } from "@aws-sdk/client-kms";
import crypto from 'node:crypto';

const kmsClient = new KMSClient();

async function getSecret(envName) {
  const encryptedValue = process.env[envName];
  if (!encryptedValue) throw new Error(`Missing variable: ${envName}`);

  const command = new DecryptCommand({
    CiphertextBlob: Buffer.from(encryptedValue, 'base64'),
    EncryptionContext: { LambdaFunctionName: process.env.AWS_LAMBDA_FUNCTION_NAME }
  });

  const { Plaintext } = await kmsClient.send(command);
  return Buffer.from(Plaintext).toString('utf-8');
}

const b64url = (input) => {
  const buf = typeof input === 'string' ? Buffer.from(input) : input;
  return buf.toString('base64').replace(/=/g, '').replace(/\+/g, '-').replace(/\//g, '_');
};

export const handler = async (event) => {
  try {
    const [EXPECTED_SECRET, CLIENT_EMAIL, PRIVATE_KEY_RAW] = await Promise.all([
      getSecret('LAMBDA_SECRET'),
      getSecret('CLIENT_EMAIL'),
      getSecret('PRIVATE_KEY')
    ]);
    
    const PRIVATE_KEY = PRIVATE_KEY_RAW.replace(/\\n/g, '\n');
    const receivedSecret = event.headers['x-api-key'] || event.headers['X-API-KEY'];
    if (!EXPECTED_SECRET || receivedSecret !== EXPECTED_SECRET) {
      return { statusCode: 401, body: JSON.stringify({ error: "Unauthorized" }) };
    }

    const body = event.body ? (typeof event.body === 'string' ? JSON.parse(event.body) : event.body) : {};
    const TOKEN_URI = "https://oauth2.googleapis.com/token";
    const now = Math.floor(Date.now() / 1000);
    
    const payload = {
      iss: CLIENT_EMAIL,
      aud: TOKEN_URI,
      iat: now,
      exp: now + 3600,
      scope: body.scopes || 'https://www.googleapis.com/auth/chat.bot https://www.googleapis.com/auth/documents https://www.googleapis.com/auth/drive'
    };

    if (body.sub) {
      payload.sub = body.sub;
    }

    const header = { alg: 'RS256', typ: 'JWT' };
    const unsignedToken = `${b64url(JSON.stringify(header))}.${b64url(JSON.stringify(payload))}`;
    const signature = crypto.sign('RSA-SHA256', Buffer.from(unsignedToken), PRIVATE_KEY);
    const assertion = `${unsignedToken}.${b64url(signature)}`;

    const response = await fetch(TOKEN_URI, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams({ 
        grant_type: 'urn:ietf:params:oauth:grant-type:jwt-bearer', 
        assertion 
      })
    });

    const data = await response.json();
    if (!response.ok) throw new Error(`Google API Error: ${JSON.stringify(data)}`);

    return {
      statusCode: 200,
      headers: { "Content-Type": "text/plain" },
      body: data.access_token
    };
  } catch (err) {
    console.error(`[HANDLER_ERROR] ${err.message}`);
    return { statusCode: 500, body: JSON.stringify({ error: err.message }) };
  }
};
```

---

### Lambda #2: `your-docusign-authenticator` (DocuSign Auth)
This function generates a JWT assertion using our DocuSign Integration Key and impersonates our DocuSign API user to fetch an access token supporting the `signature impersonation` scope.

```javascript
import { KMSClient, DecryptCommand } from "@aws-sdk/client-kms";
import crypto from 'node:crypto';

const kmsClient = new KMSClient();

async function getSecret(envName) {
  const encryptedValue = process.env[envName];
  if (!encryptedValue) throw new Error(`Missing environment variable: ${envName}`);

  const command = new DecryptCommand({
    CiphertextBlob: Buffer.from(encryptedValue, 'base64'),
    EncryptionContext: { LambdaFunctionName: process.env.AWS_LAMBDA_FUNCTION_NAME }
  });

  const { Plaintext } = await kmsClient.send(command);
  return Buffer.from(Plaintext).toString('utf-8');
}

const b64url = (input) => {
  const buf = typeof input === 'string' ? Buffer.from(input) : input;
  return buf.toString('base64').replace(/=/g, '').replace(/\+/g, '-').replace(/\//g, '_');
};

export const handler = async (event) => {
  try {
    const [EXPECTED_API_KEY, INTEGRATION_KEY, USER_ID, PRIVATE_KEY_RAW] = await Promise.all([
      getSecret('LAMBDA_SECRET'),
      getSecret('INTEGRATION_KEY'),
      getSecret('USER_ID'),
      getSecret('PRIVATE_KEY')
    ]);

    const receivedApiKey = event.headers['x-api-key'] || event.headers['X-API-KEY'];
    if (!EXPECTED_API_KEY || receivedApiKey !== EXPECTED_API_KEY) {
      return { statusCode: 401, body: JSON.stringify({ error: "Unauthorized" }) };
    }

    const cleanKeyBody = PRIVATE_KEY_RAW
      .replace(/-----(BEGIN|END) (RSA )?PRIVATE KEY-----/g, '')
      .replace(/\\n/g, '')
      .replace(/\s/g, '');

    const FORMATED_KEY = `-----BEGIN RSA PRIVATE KEY-----\n${cleanKeyBody.match(/.{1,64}/g).join('\n')}\n-----END RSA PRIVATE KEY-----`;
    const DS_AUTH_SERVER = "account.docusign.com"; 
    const now = Math.floor(Date.now() / 1000);
    
    const payload = {
      iss: INTEGRATION_KEY.trim(),
      sub: USER_ID.trim(),
      aud: DS_AUTH_SERVER,
      iat: now,
      exp: now + 3600,
      scope: 'signature impersonation'
    };

    const header = { alg: 'RS256', typ: 'JWT' };
    const unsignedToken = `${b64url(JSON.stringify(header))}.${b64url(JSON.stringify(payload))}`;
    const signature = crypto.sign('RSA-SHA256', Buffer.from(unsignedToken), FORMATED_KEY);
    const assertion = `${unsignedToken}.${b64url(signature)}`;

    const response = await fetch(`https://${DS_AUTH_SERVER}/oauth/token`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams({ grant_type: 'urn:ietf:params:oauth:grant-type:jwt-bearer', assertion })
    });

    const data = await response.json();
    if (!response.ok) return { statusCode: response.status, body: JSON.stringify({ success: false, docusign_error: data }) };

    return {
      statusCode: 200,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ success: true, access_token: data.access_token, expires_in: data.expires_in })
    };
  } catch (err) {
    console.error(`[FATAL_ERROR]`, err);
    return { statusCode: 500, body: JSON.stringify({ success: false, error: err.message }) };
  }
};
```

---

### Lambda #3: `your-doc-converter` (Exporting Google Doc to PDF Base64)
To dynamically upload the newly generated Google Doc into DocuSign, I had to convert the document into a base64-encoded PDF binary stream. 

This Lambda requests a Google Bearer Token (impersonating our secure automation email), reaches out to Google Drive's v3 Export API, converts the file to PDF, and packages it as Base64.

```javascript
const https = require('https');
const crypto = require('crypto');

function base64url(str) {
  return Buffer.from(str).toString('base64').replace(/=/g, '').replace(/\+/g, '-').replace(/\//g, '_');
}

async function getGoogleAccessToken(clientEmail, privateKey, impersonatedUser) {
  const header = { alg: 'RS256', typ: 'JWT' };
  const iat = Math.floor(Date.now() / 1000);
  const exp = iat + 3600;
  
  const claimSet = {
    iss: clientEmail,
    sub: impersonatedUser, 
    scope: 'https://www.googleapis.com/auth/drive https://www.googleapis.com/auth/documents',
    aud: 'https://oauth2.googleapis.com/token',
    exp, iat
  };

  const encodedHeader = base64url(JSON.stringify(header));
  const encodedClaimSet = base64url(JSON.stringify(claimSet));
  const signatureInput = `${encodedHeader}.${encodedClaimSet}`;

  const sign = crypto.createSign('RSA-SHA256');
  sign.update(signatureInput);
  sign.end();
  const signature = sign.sign(privateKey, 'base64').replace(/=/g, '').replace(/\+/g, '-').replace(/\//g, '_');
  const jwt = `${signatureInput}.${signature}`;

  return new Promise((resolve, reject) => {
    const postData = `grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer&assertion=${jwt}`;
    const options = {
      hostname: 'oauth2.googleapis.com', path: '/token', method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded', 'Content-Length': Buffer.byteLength(postData) }
    };

    const req = https.request(options, (res) => {
      let data = '';
      res.on('data', chunk => data += chunk);
      res.on('end', () => {
        if (res.statusCode === 200) resolve(JSON.parse(data).access_token);
        else reject(new Error(`Token generation failed: ${data}`));
      });
    });
    req.on('error', reject);
    req.write(postData);
    req.end();
  });
}

async function exportPdfBase64(fileId, token) {
  return new Promise((resolve, reject) => {
    const options = {
      hostname: 'www.googleapis.com',
      path: `/drive/v3/files/${fileId}/export?mimeType=application/pdf`,
      method: 'GET',
      headers: { 'Authorization': `Bearer ${token}` }
    };

    const req = https.request(options, (res) => {
      if (res.statusCode !== 200) {
         let errData = '';
         res.on('data', c => errData += c);
         res.on('end', () => reject(new Error(`Drive API Error: ${res.statusCode} -${errData}`)));
         return;
      }
      const chunks = [];
      res.on('data', chunk => chunks.push(chunk));
      res.on('end', () => resolve(Buffer.concat(chunks).toString('base64')));
    });
    req.on('error', reject);
    req.end();
  });
}

exports.handler = async (event) => {
  try {
     const fileId = event.queryStringParameters && event.queryStringParameters.id;
     if (!fileId) return { statusCode: 400, body: JSON.stringify({error: "Parameter 'id' is required."}) };

     const clientEmail = process.env.GOOGLE_CLIENT_EMAIL;
     const privateKey = process.env.GOOGLE_PRIVATE_KEY.replace(/\\n/g, '\n');
     const userEmail = "your-automation-account@yourcompany.com";

     const token = await getGoogleAccessToken(clientEmail, privateKey, userEmail);
     const base64Pdf = await exportPdfBase64(fileId, token);

     return {
        statusCode: 200,
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ documentBase64: base64Pdf })
     };
  } catch (err) {
     return { statusCode: 500, body: JSON.stringify({error: err.message}) };
  }
};
```

---

### 5. The "Anti-Jumpscare" DocuSign Routing Order: Respecting Corporate Blood Pressure
If you send an unannounced, complex security risk document directly to a VP or C-Level, they will (rightfully) drop everything and panic. 

To prevent these corporate jumpscares, I designed a strict **Routing Order** inside my DocuSign payload:
* **Routing Order 1 (The Engineering Manager):** Must sign and validate the technical details first.
* **Routing Order 2 (The Director / BU VP):** Only receives the signature request *after* their manager has successfully signed.

This ensures that by the time a VP opens the document, they can see their trusted manager has already approved and aligned on the technical context. No surprise alarms, just aligned corporate governance.

```json
{
  "recipients": {
    "signers": [
      {
        "email": "manager@company.com",
        "name": "Manager Name",
        "recipientId": "1",
        "routingOrder": "1"
      },
      {
        "email": "director@company.com",
        "name": "Director Name",
        "recipientId": "2",
        "routingOrder": "2"
      }
    ]
  }
}
```

### 6. DocuSign Signal Receiver: Webhook Custom Fields & No-Overhead Jira Transitions
Instead of keeping the `CreateDoc` workflow hanging on a long sleep state waiting for humans to sign, I built a highly responsive, event-driven listener called **DocuSign Signal Receiver** inside Astroflow.

When the final signature is collected, DocuSign fires a secure webhook callback containing the complete envelope metadata directly to this receiver workflow. 

Instead of dealing with complex Jira REST API v3 queries, handling OAuth handshakes, or wrestling with Atlassian's picky Atlassian Document Format (ADF) comment schemas, I designed a lightweight, zero-overhead decoupling pattern:

1. **State Carrying via Envelope Custom Fields:** During the initial envelope creation phase, I embed the original Jira issue key (e.g., `VULN-XXXX`) as metadata inside a DocuSign custom field called `Gdv_Key`.
2. **The Dynamic Webhook Payload:** When the envelope is completed, the incoming webhook triggers the `DocuSign Signal Receiver` with a structured JSON payload:
   ```json
   {
     "result": {
       "status": "completed",
       "envelopeId": "your-envelope-id",
       "customFields": {
         "textCustomFields": [
           {
             "name": "Flow Type",
             "value": "Risk Acceptance"
           },
           {
             "name": "Gdv_Key",
             "value": "VULN-XXXX"
           }
         ]
       }
     }
   }
   ```
3. **The Zero-Body Trigger:** The receiver workflow extracts the ticket key directly from the metadata: `customFields.textCustomFields[1].value`. 
4. **Targeted Jira Execution:** Instead of formatting a POST request body, the Astroflow workflow dispatches a fast, empty HTTP query directly to a Jira Automation webhook, appending the issue key in the URL parameter:
   ```text
   GET https://your-jira-instance.atlassian.com/automation/webhooks/jira/a/your-project-uuid/your-webhook-uuid?issue=VULN-XXXX
   ```
5. **Native Automation Handshake:** Jira's native automation engine handles the rest—intercepting the query parameter, advancing the issue state to `RISK ACCEPTANCE` and logging comments natively on the correct ticket.

This integration is clean, lightning-fast, and completely bypasses REST API complexity.

### 7. Native Agreement Actions: Seamless Google Drive Archival
Instead of overloading my workflow engine (Astroflow) with downloading and re-uploading massive binary files, I opted for a much cleaner architectural approach using DocuSign's native capabilities.

I established a secure organization-level cloud connection between **DocuSign** and our corporate **Google Drive**. Inside DocuSign, I configured an **Agreement Action** rule that listens for the "Completed" state of our templates. 

The moment the final signature is applied:
1. DocuSign natively packages the signed document and its official signature certificate into a `.zip` file.
2. It automatically saves it directly into our designated secure audit folder in Google Drive. 

Zero overhead, zero custom code, and zero load on our local workflow environment.

---

## 🤖 The Daily Chaser Flow & Bot Aegis Gateway

I realized that even with perfect automation, humans are still humans. People forget to open their emails, and DocuSign alerts often get lost in spam or daily clutter. Managers are busy keeping the business running, so they shouldn't have to spend mental energy keeping track of compliance paperwork.

To rescue our stakeholders from their inbox black holes, I built a secondary **Daily Chaser** automation within Astroflow:

1. **The Scheduler:** A cron workflow runs once a day, calling the DocuSign API to fetch all envelopes that are still pending.
2. **The Parser:** For each pending envelope, the workflow extracts the current signer's email and formats a customized, friendly nudge message.
3. **The Notification Gateway (Bot Aegis):** Astroflow dispatches this message payload via webhook to **Bot Aegis**, my custom Google Chat application. Aegis then interacts with the Google Chat API to deliver a direct, polite message to the user:
   > *"Hey there! 🛡️ Quick heads-up: You have a pending Risk Acceptance signature for the vulnerability **[CVE-XXXX-XXXX]**. Clicking [here] takes 30 seconds and keeps our systems compliant! Thanks!"*

By separating the notification gateway (Aegis) from the scheduler logic (Astroflow), I created a highly modular and extensible warning system.

---

## 📈 The Business Impact

| Metric | Before This Automation | After This Automation |
| :--- | :--- | :--- |
| **Document Generation** | 20 - 30 minutes (manual) | **< 3 seconds** (automated) |
| **Follow-up effort** | Hours of manual pinging | **0 hours** (handled by Astroflow + Bot Aegis) |
| **SLA Resolution Time** | Weeks (waiting on signatures) | **Within minutes** of final signature |
| **Audit Compliance** | Prone to human/copy-paste typos | **100% accurate** mapped data |

---

## 🧠 Key Takeaways & Lessons Learned

1. **API design must account for corporate culture:** The DocuSign Routing Order wasn’t just a technical configuration; it was a cultural success because it respected the established hierarchy and prevented friction between teams.
2. **Keep integrations simple when possible:** Leveraging Jira's native Automation webhooks with query parameters saved me from writing hundreds of lines of complex REST API wrappers and dealing with Atlassian Document Format (ADF) schemas. Always look for the path of least resistance when connecting enterprise tools.
3. **Keep workflows highly modular:** Decoupling the creation phase (`CreateDoc`) from the asynchronous webhook listener (`DocuSign Signal Receiver`) prevents massive failures, simplifies debugging, and isolates API rate limits.
4. **Automate the follow-up:** The real bottleneck of any document pipeline is human delay. Building a proactive notification system using a daily cron workflow and a custom Google Chat Bot (Aegis) was just as critical to my success as the document generation engine itself.

---

## 📝 Current Deployment & Future Roadmap (TODO)
> **Current Status:** Staging Ecosystem 🧪

Currently, these secure Lambda microservices are deployed and fully validated inside a dedicated staging environment. 

### My Next Steps (On my security engineering checklist):
* 🔒 **Hardened Account Migration:** Migrate these functions out of staging and deploy them into a locked-down production AWS account.
* 🛡️ **Tighten KMS & IAM Policies:** Further restrict AWS KMS decryption contexts so that *only* the specific production Lambda execution roles have decrypt capabilities, keeping key material completely isolated from any human operator.

---

### 💬 Let's Connect!
Developed with 💙 by me. If you have questions about setting up this automation process or any insights, feel free to connect on [LinkedIn](https://www.linkedin.com/in/samuelchiodi/)!
