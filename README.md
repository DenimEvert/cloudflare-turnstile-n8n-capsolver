# Automating Cloudflare Turnstile with n8n and CapSolver

Cloudflare Turnstile is a modern CAPTCHA alternative that verifies users silently in the background. This guide demonstrates how to integrate **[n8n](https://n8n.io/)** (a visual workflow automation tool) with **[CapSolver](https://www.capsolver.com/?utm_source=github)** (an AI-powered CAPTCHA solving service) to bypass Turnstile challenges. You'll learn to build reusable solver APIs and embed CapSolver into larger automation workflows for tasks like web scraping and account logins, all without writing traditional code.

## What You'll Build

*   **Solver APIs**: Reusable endpoints for other tools to call, specifically a Cloudflare Turnstile solver API.
*   **Direct-use Workflows**: CapSolver integrated as a step within broader automations, such as:
    *   A price & product scraper that solves Turnstile, fetches protected pages, and alerts on price changes.
    *   An account login automation that solves Turnstile before submitting credentials.

---

## What is Cloudflare Turnstile?

Cloudflare Turnstile is a CAPTCHA alternative that verifies visitors automatically using browser signals, eliminating the need for puzzles or checkboxes. It appears as a small widget embedded in forms (login, signup, checkout) and operates in three modes: **Managed**, **Non-Interactive**, and **Invisible**. From a solving perspective, the mode is irrelevant; you always need the same two parameters: `websiteURL` and `websiteKey`.

![](https://assets.capsolver.com/prod/posts/how-to-solve-cloudflare-turnstile-n8n/csE8JW3ryzyv-d2b5ca33bd970f64a6301fa75ae2eb22.png)

> **Important**: Turnstile is **not** the same as a Cloudflare Challenge. If you encounter a full-page "Performing security verification..." screen, that's a Cloudflare Challenge, which requires a [different solver](https://www.capsolver.com/blog/Cloudflare/how-to-identify-turnstile-challenge).

---

## Prerequisites

Before you begin, ensure you have the following:

1.  **An n8n instance**: Either [self-hosted](https://docs.n8n.io/hosting/) or [n8n Cloud](https://n8n.io/cloud/).
2.  **A CapSolver account**: [Sign up here](https://dashboard.capsolver.com/passport/register?utm_source=github) and obtain your API key.
3.  **The CapSolver n8n node**: This is an official integration and is already available in n8n (no installation needed).
4.  **CapSolver browser extension** (optional but recommended): Useful for identifying CAPTCHA parameters on target websites.

> **Note**: Ensure you have sufficient balance in your CapSolver account, as Turnstile solving tasks consume credits based on usage.

---

## Setting Up CapSolver in n8n

CapSolver is an **official integration** in n8n, meaning no community node installation is required. You can find it directly in the node panel when building your workflows.

To allow the CapSolver node to authenticate with your account, you need to **create a credential** in n8n.

### Step 1: Open the Credentials Page

Navigate to your n8n instance and go to **Settings** → **Credentials**. Here, you'll see all your configured credentials.

![n8n credentials page showing CapSolver account](https://assets.capsolver.com/prod/posts/how-to-solve-recaptcha-v2-n8n/Vzt876D18xKf-d2b5ca33bd970f64a6301fa75ae2eb22.png)

### Step 2: Create the CapSolver Credential

1.  Click **Create credential** (top right).
2.  Search for **"CapSolver"** and select **CapSolver API**.
3.  Enter your **API Key**, which you can copy directly from the [CapSolver Dashboard](https://dashboard.capsolver.com/?utm_source=github).
4.  Leave **Allowed HTTP Request Domains** set to `All` (default).
5.  Click **Save**.

n8n will automatically test the connection. A green **"Connection tested successfully"** banner will confirm your API key is valid.

![CapSolver credential configuration with successful connection test](https://assets.capsolver.com/prod/posts/how-to-solve-recaptcha-v2-n8n/zJmtkndK3tk9-d2b5ca33bd970f64a6301fa75ae2eb22.png)

> **Important**: Every CapSolver node in your workflows will reference this credential. You only need to create it once, and all your solver workflows will share the same credential.

Now you're ready to build your Turnstile solver workflow!

---

## Workflow: Cloudflare Turnstile Solver API

This workflow creates a POST API endpoint that accepts Turnstile parameters and returns a solved token.

![Cloudflare Turnstile](https://assets.capsolver.com/prod/posts/how-to-solve-cloudflare-turnstile-n8n/lurxPTpXw2kl-d2b5ca33bd970f64a6301fa75ae2eb22.png)

### How It Works

The workflow comprises four nodes:

1.  **Webhook**: Receives incoming POST requests with Turnstile parameters.
2.  **Cloudflare Turnstile**: Sends the challenge to CapSolver and awaits a solution.
3.  **CapSolver Error?**: An IF node that branches based on whether solving failed (`$json.error` is not empty).
4.  **Respond to Webhook**: Returns the solution on success, or `{"error": "..."}` on failure.

### Node Configuration

#### 1. Webhook Node

| Setting     | Value            |
| :---------- | :--------------- |
| HTTP Method | `POST`           |
| Path        | `solver-turnstile` |
| Respond     | `Response Node`  |

This configuration creates an endpoint at: `https://your-n8n-instance.com/webhook/solver-turnstile`

#### 2. CapSolver Cloudflare Turnstile Node

| Parameter       | Value                       | Description                                                                 |
| :-------------- | :-------------------------- | :-------------------------------------------------------------------------- |
| Operation       | `Cloudflare Turnstile`      | Must be set to Cloudflare Turnstile                                         |
| Type            | `AntiTurnstileTaskProxyLess` | This task type does not require a proxy                                     |
| Website URL     | `{{ $json.body.websiteURL }}` | The URL of the page containing the Turnstile widget                         |
| Website Key     | `{{ $json.body.websiteKey }}` | The Turnstile site key                                                      |
| metadata.action | *(Optional)*                | Some sites require a specific action string for the Turnstile challenge     |
| metadata.cdata  | *(Optional)*                | Custom data that some sites pass to the Turnstile widget for verification |

> Some sites may also require **metadata.action** and/or **metadata.cdata**. You can add these under the **Optional** section in the node. Remember to select your **CapSolver credentials** in this node.

#### 3. CapSolver Error? Node (IF)

| Setting     | Value                               |
| :---------- | :---------------------------------- |
| Condition   | `={{ $json.error }}` **is not empty** |
| True branch | Routes to the **Error** Respond to Webhook node |
| False branch | Routes to the **Success** Respond to Webhook node |

This setup explicitly defines the error path on the canvas. The CapSolver node continues on error (`onError: continueRegularOutput`), so failures are passed as `{ "error": "..." }` instead of crashing the workflow.

#### 4. Respond to Webhook Nodes

**Success branch** (false output of CapSolver Error?):

| Setting       | Value                          |
| :------------ | :----------------------------- |
| Respond With  | `JSON`                         |
| Response Body | `={{ JSON.stringify($json.data) }}` |

### Test It

Send a POST request to your webhook endpoint using `curl`:

```bash
curl -X POST https://your-n8n-instance.com/webhook/solver-turnstile \
  -H "Content-Type: application/json" \
  -d '{
    "websiteURL": "https://example.com/login",
    "websiteKey": "0x4AAAAAAADV8V8V8V8V8V8V"
  }'
```

**Expected Response:**

```json
{
  "taskId": "abc123...",
  "solution": {
    "token": "0.XXXXXXXXXXXXXXXX..."
  },
  "status": "ready"
}
```

### Import This Workflow

To import this workflow into n8n, copy the JSON content from `workflow_turnstile_solver_api.json` and use **Menu** → **Import from JSON** in your n8n instance.

[View Workflow JSON: `workflow_turnstile_solver_api.json`](./workflow_turnstile_solver_api.json)

---

## Workflow: Price & Product Scraper

This workflow demonstrates how to build a price and product scraper that can handle Cloudflare Turnstile protection. It includes both scheduled and webhook-triggered paths.

![Price & Product Scraper](https://assets.capsolver.com/prod/posts/how-to-solve-cloudflare-turnstile-n8n/QJmwiapvJOO0-09dd8c2662b96ce14928333f055c5580.png)

### How It Works

This workflow features two main paths:

*   **Scheduled Path**: Runs every 6 hours, solves Turnstile, fetches the product page, extracts and compares data, and triggers alerts if prices change.
*   **Webhook Path**: Initiates the scraping process via a webhook trigger, solves Turnstile, fetches the product page, extracts and compares data, and responds to the webhook.

Both paths use a **Set Target Config** node to centralize parameters like `websiteURL` and `websiteKey`.

### Node Configuration Highlights

*   **Schedule Trigger**: Configured to run every 6 hours.
*   **Set Target Config**: Defines `websiteURL` (e.g., `https://YOUR-TARGET-SITE.com/product-page`) and `websiteKey` (e.g., `YOUR_SITE_KEY_HERE`).
*   **Solve Turnstile**: Uses the CapSolver node to get a Turnstile token.
*   **Fetch Product Page**: An HTTP Request node that sends the `cf-turnstile-response` token in the header to access the protected page. Includes a `user-agent` header.
*   **Extract Data**: An HTML node to extract `price` and `productName` using CSS selectors (e.g., `.product-price, [data-price], .price` for price).
*   **Compare Data**: A Code node (JavaScript) to compare the current price with the previously stored price (in `$workflow.staticData`), determine if there's a change, and calculate the difference. It also updates `staticData` with the new price and timestamp.
*   **Data Changed?**: An IF node to check if `changed` is true.
*   **Build Alert / No Change**: Set nodes to prepare alert messages or status updates based on price changes.
*   **Respond to Webhook**: For the webhook path, this node returns the scraping results as JSON.

### Import This Workflow

To import this workflow into n8n, copy the JSON content from `workflow_price_scraper.json` and use **Menu** → **Import from JSON** in your n8n instance.

[View Workflow JSON: `workflow_price_scraper.json`](./workflow_price_scraper.json)

---

## Workflow: Account Login Automation

This workflow automates logging into a CAPTCHA-protected site. It also provides both scheduled and webhook-triggered paths, centralizing configuration in **Set Login Config** nodes.

### How It Works

Key behaviors for both scheduled and webhook paths:

*   **Trigger**: Can be `Every 24 Hours` (scheduled) or a `Webhook Trigger` (on-demand).
*   **Set Login Config**: Centralizes parameters like `websiteURL`, `websiteKey`, `successMarker` (a string to look for in the response body to confirm successful login), and `userAgent`.
*   **Solve Captcha**: Defaults to **Cloudflare Turnstile**. You can change the `Operation` in this node for other CAPTCHA types.
*   **Submit Login**: An HTTP Request node configured to send `email`, `password`, and the `cf-turnstile-response` token in the request body. You'll need to edit this node to match your site's specific field names.
*   **Login OK?**: An IF node that checks if `statusCode < 400` and if the `successMarker` is present in the response body.
*   **Mark Success / Mark Failed**: Set nodes to record the login outcome and timestamp.
*   **Respond to Webhook**: For the webhook path, this node returns the login result as JSON.

### Import This Workflow

To import this workflow into n8n, copy the JSON content from `workflow_account_login.json` and use **Menu** → **Import from JSON** in your n8n instance.

[View Workflow JSON: `workflow_account_login.json`](./workflow_account_login.json)

---

## Conclusion

This guide has shown you how to build a **Cloudflare Turnstile-solving API** and **production-ready scraping workflows** using n8n and CapSolver, all without traditional coding.

We covered:

*   An API solver endpoint for Cloudflare Turnstile using a webhook-based workflow.
*   Practical use-case examples, including web scraping and account login, demonstrating how to submit solved tokens and process protected data.
*   Methods for identifying Turnstile parameters by inspecting page sources.
*   Best practices for token handling, error management, and production use.

The crucial takeaway is that solving the Turnstile challenge is only half the battle; you must also **submit the token** to the target website to unlock the protected data.

> **Tip**: These workflows use Schedule + Webhook triggers, but you can swap the trigger node to any [n8n trigger](https://docs.n8n.io/integrations/builtin/core-nodes/) (e.g., manual, app event, form submission). After fetching data, leverage n8n's built-in nodes to save results to Google Sheets, databases, cloud storage, or send alerts via Telegram/Slack/Email.

---

> **Ready to get started?** [Sign up for CapSolver](https://www.capsolver.com/?utm_source=github) and use bonus code **n8n** for an extra 8% bonus on your first recharge!

![CapSolver bonus code banner](https://assets.capsolver.com/prod/posts/how-to-solve-recaptcha-v2-n8n/UXwrijPMSvAB-d2b5ca33bd970f64a6301fa75ae2eb22.png)

---

## Frequently Asked Questions

### What is Cloudflare Turnstile?

Cloudflare Turnstile is a CAPTCHA alternative that verifies visitors without requiring them to solve puzzles. It runs in the background using browser signals and behavioral analysis to determine if a visitor is human.

### How much does it cost to solve a Turnstile challenge?

Pricing varies based on usage. Check the [CapSolver pricing page](https://www.capsolver.com/pricing?utm_source=github) for current Turnstile rates.

### How long does it take to solve a Turnstile challenge?

Turnstile challenges are typically solved in **a few seconds** since no image challenges are involved.

### Can I use this workflow with n8n Cloud?

Yes! This workflow works with both self-hosted n8n and n8n Cloud. The CapSolver node is already available as an official integration; just add your API credentials.

### How do I find the Turnstile site key for a website?

Search the page source for `data-sitekey` in the HTML or look for `turnstile.render()` in the JavaScript. You can also open DevTools (`F12`) → **Network** tab and filter by `turnstile` to find the site key in requests. For a detailed guide, see the [CapSolver documentation](https://www.capsolver.com/blog/Extension/identify-any-captcha-and-parameters).

### CapSolver returned a token but the website still rejected it — why?

Several factors can cause this. First, **tokens expire quickly**; ensure you're submitting the token immediately. Second, **verify you're sending the token to the right place**: inspect the actual network request the browser makes when you submit the form (DevTools → Network tab) and confirm the field name (`cf-turnstile-response` is typical but not universal), request method, and endpoint all match what you've configured in n8n. Third, some sites require **metadata.action** or **metadata.cdata** parameters; use the CapSolver extension to check if any of these apply. If the token is still rejected, [contact CapSolver support](https://www.capsolver.com/contact) for site-specific help.
