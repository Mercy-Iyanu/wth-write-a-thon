# How to Make Payouts to Interledger Wallets Using Chimoney’s API

In this tutorial, you will learn how to integrate Chimoney’s `/payouts/interledger-wallet-address` endpoint into your existing project and send live payouts using your wallet-enabled Chimoney account.

---

## What You’ll Build

- Configure your environment
- Build a payout service with Chimoney’s API
- Handle real-world errors (like currency mismatches)
- Test your first ILP payout

---

## Prerequisites

Before you start, make sure you have completed the [Setup Guide](./setup.md). You should already have:
- A Chimoney developer account
- Your API key
- Your ILP wallet address
- Tested the `payouts to interledger wallet address` endpoint

Additionally;
- Basic knowledge of RESTful APIs
- Node.js and npm/yarn installed
- Axios for HTTP requests

With that out of the way, let us dive in.

---

## Project Structure

Here is what your folder might look like:

```bash
chimoney-payouts/
├── .env
├── src/
│   ├── config.js
│   ├── services/
│   │   └── chimoneyService.js
│   └── index.js
├── package.json
└── README.md
```

## Step 1: Configuration & Setup

### Install Dependencies

```bash
mkdir chimoney-payouts && cd chimoney-payouts
npm init -y
npm install axios dotenv
```

### Environment Variables

Create `.env`:

```js
CHIMONEY_API_KEY=your_api_key_here
CHIMONEY_ENV=sandbox # or "production"

ILP_WALLET_ADDRESS=https://ilp-sandbox.chimoney.com/your-id
```

---

### Configuration Module (`config.js`)

```js
import dotenv from "dotenv";
dotenv.config();

export const config = {
  apiKey: process.env.CHIMONEY_API_KEY,
  baseUrl:
    process.env.CHIMONEY_ENV === "production"
      ? "https://api-v2.chimoney.io/v0.2.4"
      : "https://api-v2-sandbox.chimoney.io/v0.2.4",
  headers: {
    Accept: "application/json",
    "Content-Type": "application/json",
    "X-API-KEY": process.env.CHIMONEY_API_KEY,
  },
};
```

---

## Step 2: Build the Chimoney Interledger Service
Now that your configuration is set up, it is time to write the code that will send a payout to an Interledger wallet using Chimoney’s API. Let us start by creating a reusable service class.

### Create `chimoneyService.js`:

```js
import axios from "axios";
import { config } from "./config.js";

class ChimoneyService {
  constructor() {
    this.client = axios.create({
      baseURL: config.baseUrl,
      headers: config.headers,
    });
  }

  async sendInterledgerPayout({ walletAddress, amount, narration, collectionPaymentIssueID = "" }) {
    try {
      const payload = {
        turnOffNotification: false,
        debitCurrency: "USD",
        interledgerWallets: [
          {
            interledgerWalletAddress: walletAddress,
            currency: "USD",
            amountToDeliver: amount,
            narration,
            collectionPaymentIssueID, // optional but great for tracking payouts
          },
        ],
      };

      const res = await this.client.post("/payouts/interledger-wallet-address", payload);
      return res.data;
    } catch (err) {
      if (err.response) {
        console.error("Chimoney API error:", err.response.data);
        throw err.response.data;
      }
      throw err;
    }
  }
}

export default ChimoneyService;
```

---

## Step 3: Usage Example (`index.js`)
In this step, you will see how to **bring everything together**.
We will import the Chimoney service, provide an Interledger wallet address, set an amount and narration, then trigger a payout. This example also shows how to capture success responses and gracefully handle errors.

```js
import ChimoneyService from "./chimoneyService.js";
import dotenv from "dotenv";
dotenv.config();

(async () => {
  const service = new ChimoneyService();

  const walletAddress = process.env.ILP_WALLET_ADDRESS; // e.g., https://ilp-sandbox.chimoney.com/your-id
  const amount = 10; // USD
  const narration = "Freelance payout for job #123";
  const collectionPaymentIssueID = "job-123-issue";

  try {
    const result = await service.sendInterledgerPayout({
      walletAddress,
      amount,
      narration,
      collectionPaymentIssueID,
    });
    console.log("Payout Success:", result);
  } catch (error) {
    const res = error.response;
    console.error("Payout Failed:", {
        status: res?.status,
        message: res?.data?.message || res?.data?.error || error.message
    });
    // handle error messages, e.g.:
    // if errorData.message includes "CAD is not enabled."
    // if errorData.error includes "valid Chimoney user ID"
  }
})();
```
---

## How It Works (Behind the Scenes)

1. You make a `POST` request to the `/payouts/interledger-wallet-address` endpoint
2. Chimoney authenticates your API key
3. Chimoney deducts the amount from your wallet
4. The ILP wallet address receives the payout (in supported currency)
5. A response is returned with `chiRef`, status, and optionally a `paymentLink`

---

## Handling Errors & Troubleshooting
Chimoney’s API may return various errors depending on your inputs or environment. Below are common issues and how to resolve them:

| **Code** | **Description**   | **Example Message**                                                | **How to Fix**                                                                 |
|----------|-------------------|--------------------------------------------------------------------|--------------------------------------------------------------------------------|
| **200**  | Success           | `"Payout to Chimoney wallets completed successfully."`             | No action needed.                                                              |
| **400**  | Validation error  | `"The request parameters are invalid."`                            | Double-check request body (wallet address format, required fields, etc.).      |
| **401**  | Unauthorized      | `"API key is not defined. Generate a new one from the developer portal."` | Ensure your API key is set in `.env` and passed in headers.                    |
| **403**  | Forbidden         | `"API Access not enabled for account Test. Email support@chimoney.io"` | Contact Chimoney support to enable API access for your account.                |
| **500**  | Server error      | `"An internal server error occurred."`                             | Retry after a short delay. If persistent, reach out to Chimoney support.       |

---

## You Did It

You just:
- Set up your environment
- Made a successful payout to an Interledger wallet
- Learned how to handle and debug errors


That’s it for the setup!

_Next up: Extend this into a backend job or admin panel!_

