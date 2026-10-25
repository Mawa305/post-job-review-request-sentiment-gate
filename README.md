# Post-Job Review Request with Sentiment Gate

An n8n workflow for an HVAC business that asks customers for feedback after a job. When a job is marked complete, the customer gets an email with two buttons: "Great!" and "Not Great". Happy customers are sent a link to leave a public Google review. Unhappy customers do not get the review request, and the owner gets a Telegram alert so they can follow up directly.

## Architecture

```
Part 1: Job complete (POST /job-complete)
Job Complete Trigger → Log New Review Request (Supabase: reviews, status = pending)
   → Send Feedback Request Email (Gmail: "How was your service today?")
        ├── success → Respond Job Queued
        └── error   → Alert: Feedback Email Failed (Telegram) → Respond Job Queued

Part 2: Customer clicks a button (GET /review-response)
Customer Response Trigger → Update Sentiment (Supabase: sentiment, status = responded)
   ├── success → Sentiment Gate
   │               ├── positive → Send Review Request Email (Gmail: Google review link)
   │               │               ├── success → Confirm to Customer
   │               │               └── error   → Alert: Review Email Failed → Confirm to Customer
   │               └── negative → Alert: Negative Feedback (Telegram) → Confirm to Customer
   └── error   → Alert: Sentiment Update Failed (Telegram) → Confirm to Customer
```

## Features

* Logs each completed job to Supabase before sending anything
* Sends the customer a feedback email with a "Great!" and a "Not Great" button
* Positive feedback gets a thank-you email with a Google review link
* Negative feedback sends no review request. The owner gets a Telegram alert with the customer name, job ID, and technician
* Every webhook call gets a response, so the caller never waits on an unanswered request
* Email and database failures are sent to Telegram, with the customer still shown a confirmation page

## Tech Stack

* n8n 2.x (workflow automation, webhooks)
* Supabase (reviews table)
* Gmail API (customer emails)
* Telegram Bot API (owner alerts)

## Setup

1. Create a Supabase table named `reviews` with these columns:

```
id | job_id | customer_name | email | technician_name | status | sentiment
```

2. Import `post_job_review_request_sentiment_gate.json` into n8n.
3. Add credentials: Supabase, Gmail OAuth2, and Telegram. Assign them to the matching nodes.
4. Replace these placeholders:
   - `YOUR_N8N_DOMAIN`: your n8n domain, in the two "Great!" and "Not Great" links inside Send Feedback Request Email
   - `YOUR_GOOGLE_REVIEW_LINK`: your business's Google review link, in Send Review Request Email
   - `YOUR_TELEGRAM_CHAT_ID`: your chat ID, in all Telegram nodes
5. Activate the workflow.
6. Send a POST request to `/job-complete` with a test job, then open the feedback email and click a button.

## Sample Request

```
POST https://YOUR_N8N_DOMAIN/webhook/job-complete
Content-Type: application/json

{
  "job_id": "JOB-1001",
  "customer_name": "Ali Raza",
  "email": "ali.test@example.com",
  "technician_name": "Usman"
}
```

Expected result: a `reviews` row with status `pending`, a feedback email to the customer, and a `{"status": "queued"}` response.

## Security Note

The button links are plain GET requests with `job_id` and `sentiment` in the URL. Anyone who knows a job ID could submit a sentiment for it. For a production setup, add a random token to each link and check it in the Update Sentiment step.

## Project Notes

* Built as part of my AI Automation portfolio.
* Sentiment is chosen by the customer with a button, not by an AI model, so the workflow has no LLM cost and no risk of misreading feedback.
* The "Confirm to Customer" page says only "Thank you for your feedback!" for every outcome, so a customer never sees an internal error.
