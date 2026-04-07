# Part 1: Shared Gateway — Rate Limiting

## Problem

25 customers send >20 requests/second. External API only allows 3 req/sec.

## Architecture

- Each customer gets their own Celery queue (Customer_A_queue, Customer_B_queue, etc.)
- A Round-Robin dispatcher picks 1 request from each customer queue in turn
- Redis (ElastiCache) stores all queues
- Celery workers process at a global rate of 3 tasks/second using `rate_limit="3/s"`

## Rate Limiting Strategy: Token Bucket

- A token is added every 1/3 seconds (= 3 tokens/sec)
- Each request consumes 1 token
- If no tokens, the request waits in the queue (never dropped)

## Fairness

Instead of one shared queue, we use per-customer queues:

- Customer A sending 100 requests doesn't block Customer B
- The dispatcher picks one from A, one from B, one from C — round-robin

## Failure Handling: Exponential Backoff

If the API fails: retry after 1s → 2s → 4s → 8s (doubles each time)
This prevents hammering a failing API and making things worse.
