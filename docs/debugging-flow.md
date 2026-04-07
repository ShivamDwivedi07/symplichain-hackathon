# Part 4: Debugging the Monday Photo Upload Outage

## Flow: App → EC2 (Django) → S3 → Celery → Bedrock → RDS

### Step 1 — Check Django logs on EC2

SSH into EC2, then:
tail -n 100 /var/log/gunicorn/error.log
grep "S3" /var/log/nginx/error.log

Question: Did the request reach Django? Did S3 upload succeed?

### Step 2 — Check Celery task queue (Flower dashboard)

Open: http://your-ec2-ip:5555 (Celery Flower)
Look for: tasks stuck in PENDING or FAILURE state

Question: Is the Celery task being created? Is it failing mid-way?

### Step 3 — Check AWS CloudWatch for Bedrock

Go to CloudWatch → Logs → /aws/bedrock
Look for: TimeoutException, ThrottlingException

Question: Is Bedrock timing out on the photo analysis?

### Step 4 — Check RDS connections

Go to CloudWatch → RDS → DatabaseConnections metric
Look for: connection count at the max limit (causes new writes to fail)

Question: Is the final result failing to save to the database?

### My Hypothesis for Monday

Monday = start of week = high traffic spike.
Most likely cause: Bedrock timeouts under load, causing Celery tasks
to pile up, eventually exhausting RDS connections.

Fix: Add Celery task timeout + retry with exponential backoff.
Add CloudWatch alarm for RDS connection count > 80%.
