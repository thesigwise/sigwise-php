# SigWise API SDK for PHP

Send events about your objects, get typed answers back.

Covers version 1.0.0 of the API. Full documentation, guides and the API reference:
<https://sigwise.ai/docs>.

## Get your API key and secret

1. Sign up at <https://sigwise.ai/register> (or sign in).
2. In the console, open **API keys** and click **New key**.
3. Copy both values it shows:
   - the **key ID** (for example `7Hx2Qp9LmZ`), which identifies the key;
   - the **signing secret**, which is **shown only once**. Store it like a password.

Pass them to the client, or set them in the environment, where the client
finds them on its own:

```bash
export SIGWISE_API_KEY=your_key_id
export SIGWISE_SECRET=your_secret
```

Lost the secret? **Rotate** the key on the same page to get a new one.

## Install

```bash
composer require sigwise/sdk
```

## Quick start

```php
<?php

require 'vendor/autoload.php';

// Null arguments fall back to SIGWISE_API_KEY and SIGWISE_SECRET.
$sigwise = new SigWise\Client('your_key_id', 'your_secret');

// Configure what you want to know about your objects.
$sigwise->signals->upsert('is_scammer', [
    'type' => 'noul',
    'instructions' => 'Decide if this user is likely a scammer.',
    'criteria' => ['true' => 'clear scam signals', 'false' => 'legitimate behaviour'],
]);

// Send events. Analysis runs in the background.
$sigwise->events->ingest('user-42', [
    'object_type' => 'user',
    'events' => [['type' => 'message', 'content' => 'is this still available? can I pay by wire?']],
]);

// Read the latest answers.
$object = $sigwise->objects->get('user-42');
print_r($object['analysis']);

// Or score inline and gate the content before publishing it.
$verdict = $sigwise->events->ingest('user-42', [
    'wait' => true,
    'signals' => ['is_scammer'],
    'events' => [['type' => 'message', 'content' => 'pay me by wire and I double it']],
]);
if (($verdict['answers'][0]['noul'] ?? 0) > 0.9) {
    // block it
}
```

## Authentication

The client sends the key ID with every request and signs a short-lived HS256
token with the secret, bound to the request's method and path. The secret
itself is never sent, so keep it on your server. Without explicit arguments
the client reads `SIGWISE_API_KEY` and `SIGWISE_SECRET` from the environment.

## Configuration

```php
$sigwise = new SigWise\Client('your_key_id', 'your_secret', [
    'timeout' => 10,     // seconds, per attempt
    'max_retries' => 3,  // idempotent requests only
]);
```

The client talks to the production API (`https://api.sigwise.ai`). Optionally, point it at
another deployment with the `base_url` option or the `SIGWISE_BASE_URL` environment variable.

Idempotent requests (`GET`, `PUT`, `DELETE`) are retried with exponential
backoff after a network error, a `429` or a `5xx`. Other requests are never
retried, so an event is never ingested twice.

## Errors

An error response raises an error carrying the HTTP status, the stable
machine-readable `code` (`not_found`, `payment_required`, …) and the message.

```php
use SigWise\SigWiseException;
use SigWise\ConnectionException;

try {
    $sigwise->objects->get('user-42');
} catch (SigWiseException $e) {
    echo $e->status, ' ', $e->errorCode, ' ', $e->getMessage(); // 404 not_found no data for this object
} catch (ConnectionException $e) {
    // network error or timeout
}
```

## Webhooks

Verify every delivery before trusting it. The helper checks the
`X-Webhook-Signature` HMAC against the raw body and rejects timestamps older
than five minutes.

```php
use SigWise\Webhook;
use SigWise\WebhookVerificationException;

try {
    // Pass the raw request body, exactly as received.
    $event = Webhook::constructEvent(
        file_get_contents('php://input'),
        $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'] ?? null,
        $_SERVER['HTTP_X_WEBHOOK_TIMESTAMP'] ?? null,
        getenv('SIGWISE_WEBHOOK_SECRET'),
    );
} catch (WebhookVerificationException $e) {
    http_response_code(400);
    exit;
}
if ($event['event'] === 'analysis.completed') {
    error_log($event['object_id']);
}
```

## Reference

### me

The authenticated principal and tenant settings.

- `$sigwise->me->get(array $options = [])`  
  `GET /v1/me`: Get the current principal

### overview

An object is anything you want answers about: a user, a listing, an order.

- `$sigwise->overview->get(array $options = [])`  
  `GET /v1/overview`: Get tenant overview

### objects

An object is anything you want answers about: a user, a listing, an order.

- `$sigwise->objects->analyzeAll(array $options = [])`  
  `POST /v1/analyze`: Re-analyze every object
- `$sigwise->objects->list(array $params = [], array $options = [])`  
  `GET /v1/objects`: List objects
- `$sigwise->objects->get(string $objectId, array $options = [])`  
  `GET /v1/objects/{object_id}`: Get an object's analysis
- `$sigwise->objects->getState(string $objectId, array $options = [])`  
  `GET /v1/objects/{object_id}/state`: Get an object's compacted history
- `$sigwise->objects->analyze(string $objectId, array $options = [])`  
  `POST /v1/objects/{object_id}/analyze`: Re-analyze an object

### playground

Events and messages are the evidence an object's answers are computed from.

- `$sigwise->playground->run(array $body, array $options = [])`  
  `POST /v1/playground`: Try signals on sample events

### events

Events and messages are the evidence an object's answers are computed from.

- `$sigwise->events->ingest(string $objectId, array $body, array $options = [])`  
  `POST /v1/objects/{object_id}/events`: Ingest events
- `$sigwise->events->list(string $objectId, array $options = [])`  
  `GET /v1/objects/{object_id}/events`: List an object's events

### signals

A signal is a question you ask about every object, such as "is this a scammer?" (`noul`), "how trustworthy is this user?" (`score`) or "what is their buyer intent?" (`choice`).

- `$sigwise->signals->list(array $options = [])`  
  `GET /v1/signals`: List signals
- `$sigwise->signals->upsert(string $key, array $body, array $options = [])`  
  `PUT /v1/signals/{key}`: Create or update a signal
- `$sigwise->signals->delete(string $key, array $options = [])`  
  `DELETE /v1/signals/{key}`: Delete a signal
- `$sigwise->signals->getBackfill(string $key, array $options = [])`  
  `GET /v1/signals/{key}/backfill`: Get backfill estimate and progress
- `$sigwise->signals->startBackfill(string $key, array $options = [])`  
  `POST /v1/signals/{key}/backfill`: Start a backfill

### settings

The authenticated principal and tenant settings.

- `$sigwise->settings->get(array $options = [])`  
  `GET /v1/settings`: Get tenant settings
- `$sigwise->settings->update(array $body, array $options = [])`  
  `PATCH /v1/settings`: Update tenant settings

### webhooks

Webhook endpoints receive a signed `POST` for the event types they subscribe to: `analysis.completed` each time an analysis completes, and `rule.triggered` when a rule with a webhook action fires.

- `$sigwise->webhooks->list(array $options = [])`  
  `GET /v1/webhooks`: List webhook endpoints
- `$sigwise->webhooks->create(array $body, array $options = [])`  
  `POST /v1/webhooks`: Register a webhook endpoint
- `$sigwise->webhooks->get(string $endpointId, array $options = [])`  
  `GET /v1/webhooks/{endpoint_id}`: Get a webhook endpoint
- `$sigwise->webhooks->update(string $endpointId, array $body, array $options = [])`  
  `PATCH /v1/webhooks/{endpoint_id}`: Update a webhook endpoint
- `$sigwise->webhooks->delete(string $endpointId, array $options = [])`  
  `DELETE /v1/webhooks/{endpoint_id}`: Delete a webhook endpoint

### webhookDeliveries

Webhook endpoints receive a signed `POST` for the event types they subscribe to: `analysis.completed` each time an analysis completes, and `rule.triggered` when a rule with a webhook action fires.

- `$sigwise->webhookDeliveries->list(array $params = [], array $options = [])`  
  `GET /v1/webhook_deliveries`: List webhook deliveries
- `$sigwise->webhookDeliveries->replay(string $deliveryId, array $options = [])`  
  `POST /v1/webhook_deliveries/{delivery_id}/replay`: Replay a delivery

### rules

A rule fires an action (email, Slack or webhook) when an object's signal values cross a line you care about, such as `is_scammer >= 90`.

- `$sigwise->rules->list(array $options = [])`  
  `GET /v1/rules`: List rules
- `$sigwise->rules->create(array $body, array $options = [])`  
  `POST /v1/rules`: Create a rule
- `$sigwise->rules->get(string $ruleId, array $options = [])`  
  `GET /v1/rules/{rule_id}`: Get a rule
- `$sigwise->rules->update(string $ruleId, array $body, array $options = [])`  
  `PATCH /v1/rules/{rule_id}`: Update a rule
- `$sigwise->rules->delete(string $ruleId, array $options = [])`  
  `DELETE /v1/rules/{rule_id}`: Delete a rule

### ruleFirings

A rule fires an action (email, Slack or webhook) when an object's signal values cross a line you care about, such as `is_scammer >= 90`.

- `$sigwise->ruleFirings->list(array $params = [], array $options = [])`  
  `GET /v1/rule_firings`: List rule firings

### apiKeys

Create, rotate and revoke the keys your integrations sign requests with.

- `$sigwise->apiKeys->list(array $options = [])`  
  `GET /v1/api_keys`: List API keys
- `$sigwise->apiKeys->create(array $body, array $options = [])`  
  `POST /v1/api_keys`: Create an API key
- `$sigwise->apiKeys->update(string $keyId, array $body, array $options = [])`  
  `PATCH /v1/api_keys/{key_id}`: Update an API key
- `$sigwise->apiKeys->revoke(string $keyId, array $options = [])`  
  `DELETE /v1/api_keys/{key_id}`: Revoke an API key
- `$sigwise->apiKeys->rotate(string $keyId, array $options = [])`  
  `POST /v1/api_keys/{key_id}/rotate`: Rotate an API key's secret

### billing

Prepaid balance, the billing ledger, and card top-ups.

- `$sigwise->billing->getBalance(array $options = [])`  
  `GET /v1/billing`: Get the prepaid balance
- `$sigwise->billing->listLedger(array $params = [], array $options = [])`  
  `GET /v1/billing/ledger`: List ledger entries
- `$sigwise->billing->createCheckout(array $body, array $options = [])`  
  `POST /v1/billing/checkout`: Start a card top-up
- `$sigwise->billing->getCheckout(string $sessionId, array $options = [])`  
  `GET /v1/billing/checkout/{session_id}`: Get a top-up's status

### usage

Analysis volume, spend and token usage over a date range.

- `$sigwise->usage->get(array $params = [], array $options = [])`  
  `GET /v1/usage`: Get usage

