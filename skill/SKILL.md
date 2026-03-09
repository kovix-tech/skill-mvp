---
name: create-board-task
description: Use when a task should be created in the public task board.
---

# Create Board Task

This skill creates a new task in the public task board.

Board URL:
https://v0-task-board-app-seven.vercel.app

API endpoint:
POST https://v0-task-board-app-seven.vercel.app/api/cards

## Steps

1. Ask the user for:
   - title
   - bucketId

Valid buckets:
- pending
- review
- approved
- rejected

2. Call the API using the Bash tool:

curl -X POST https://v0-task-board-app-seven.vercel.app/api/cards \
-H "Content-Type: application/json" \
-d '{"title":"TASK_TITLE","bucketId":"BUCKET_ID"}'

3. Confirm that the card was created successfully.