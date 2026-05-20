# GameHub — Understanding the System

## 1. When a user logs a new activity, how many database tables are written to?

Two tables are written:

### 1. `activities`
- Stores the new activity information.
- Example:
  - `user_id`
  - `game_id`
  - `action`

### 2. `notifications`
- Stores notification rows for friends/followers.
- Notifications are created so friends can see the activity in their feed.

---

## 2. What happens if we run:

```sql
DELETE FROM users WHERE id = 3;
```

### Result
A foreign key constraint error occurs.

### Reason
Other tables still reference the user.

Examples:
- `activities.user_id`
- `notifications.user_id`
- friendship tables

SQLite prevents deleting parent rows that are still being referenced.

### Correct approach
Delete child rows first, then delete the user.

---

## 3. User `nova` changes username to `nova_2`

### What do friends see?
Friends see:

```txt
nova_2
```

### Why?
Notifications usually store IDs, not usernames directly.

When feeds are displayed:
- the app joins `users`
- the latest username is fetched dynamically

So updated usernames appear automatically.

---

## 4. Full flow of `POST /activities`

### Steps

1. HTTP request received
2. JSON body parsed
3. Validate user and game
4. Create activity object
5. Insert into `activities`
6. Find friends/followers
7. Create notification objects
8. Insert into `notifications`
9. Commit database transaction
10. Return JSON response

---

## 5. Is the opt-out feature fully implemented?

### No.

### What is missing?
The teammate only:
- added `opted_out` column
- updated API route check

But they missed:
- notification generation logic elsewhere
- direct DB inserts
- other activity creation paths

### Real problem
Business logic is tightly coupled.

All activity-processing paths must respect `opted_out`.

---

## 6. How many rows are created when nova logs one activity?

### Formula

```txt
Total rows = 1 + number_of_friends
```

### Explanation
- `1` row inserted into `activities`
- `N` rows inserted into `notifications`

### Example
If nova has 3 friends:

```txt
1 activity row
3 notification rows

Total = 4 rows
```

---

## 7. Correct delete order for `maya_r`

### Delete order

1. `notifications`
2. `activities`
3. friendship/follow tables
4. `users`

### Why?
Foreign key constraints require child rows to be deleted before parent rows.

---

## 8. What happens if an activity with notifications is deleted?

### Result
A foreign key constraint error occurs.

### Reason

```txt
notifications.activity_id → activities.id
```

Notifications still depend on the activity row.

---

## 9. Fixing game genre and restarting app

### What else goes down?
The entire application temporarily goes down.

### Why?
Because the app is monolithic:
- API
- notifications
- activities
- game catalog

all run together in one process.

### Downtime
Until restart/deployment completes.

---

## 10. Does moving notification logic into another function solve the issue?

### No.

Moving code into another function only improves code organization.

It does NOT fix the architecture.

### Actual issue
The activity system and notification system are tightly coupled.

### Better architecture

```txt
Activity Service
        ↓
Publish Event
        ↓
Notification Service
```

### Benefits
- loose coupling
- independent deployment
- better scalability
- isolated failures
- asynchronous processing
