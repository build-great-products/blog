# Tracking Deletion Rows using Tombstones

## Introduction

Hi, I’m George, one of the developers working on Rough.app. Our mission at Rough.app is to make SaaS product management accessible to everyone in a company. You can learn more about us at [Rough.app](https://rough.app).

Rough.app is a local-first web application that prioritizes performance, offline functionality, and seamless collaboration. This means our app needs to work efficiently even with spotty internet connections, enabling teams to work together on great products. To achieve this, we use [Replicache](https://replicache.dev/) for data synchronization.

In this blog post, I’ll discuss how we handle deletions in our database using tombstone tables and share the lessons we've learned along the way.

## Initial Approach: Soft Deletes

Initially, we handled deletions using a soft delete approach. We added a `deleted_at` column to each table, marking rows as deleted by setting a timestamp in this column. While straightforward, this method presented several challenges.

### Example

Projects Table:

|----|-----------|------------|
| id | name      | deleted_at |
|----|-----------|------------|
| 1  | Project A | NULL       |
| 2  | Project B | 2024-06-01 |
| 3  | Project C | NULL       |
|----|-----------|------------|

Tasks Table:

|----|--------|------------|------------|
| id | name   | project_id | deleted_at |
|----|--------|------------|------------|
| 1  | Task 1 | 1          | NULL       |
| 2  | Task 2 | 1          | NULL       |
| 3  | Task 3 | 2          | 2024-06-01 |
| 4  | Task 4 | 3          | NULL       |
|----|--------|------------|------------|

### Challenges

#### Extra Filtering

Queries had to explicitly exclude deleted items, adding complexity and increasing the risk of accidentally including them.

Example:

```sql
SELECT *
FROM tasks
WHERE deleted_at IS NULL;
```
 
#### Cascading Deletes

Soft deletes don’t trigger cascading deletes, leading to orphaned content that required manual cleanup.

For example, if we delete a project, the associated tasks remain "active" in the database:

```sql
UPDATE projects
SET deleted_at = NOW()
WHERE id = 1;
```

We also need to update any related tasks table to mark the associated tasks as deleted:

```sql
UPDATE tasks
SET deleted_at = NOW()
WHERE project_id = 1;
```

This isn't ideal, as it requires additional queries and manual intervention to maintain data integrity.

## New Approach: Tombstone Tables

To address these issues, we shifted to using a tombstone table to track deletions. This approach simplifies our queries and centralizes the management of deleted items.

**Implementation**:
- We created a tombstone table with columns for the table name, primary keys, deletion time, and replication version.
- When an item is deleted, we insert a corresponding row into the tombstone table within a transaction to ensure atomicity.
- This ensures that if the deletion or insertion fails, the transaction is rolled back, maintaining data consistency.

**Benefits**:
- **Simpler Queries**: We no longer need to filter out deleted items in our queries.
- **Standard Delete Queries**: We can use regular delete queries without worrying about leaving behind soft-deleted rows.
- **Centralized Tracking**: All deletions are tracked in one place, making it easier to manage and query.

**Challenges**:
- **Bulk Deletes**: Handling bulk deletes requires additional logic to insert tombstone entries for each deleted item.
- **Cascading Deletes**: Cascading deletes need to be managed manually or with triggers, adding to the implementation complexity.

## Error Handling

To ensure the integrity of our delete operations, we wrap them in transactions. This ensures that both the deletion and tombstone insertion either succeed together or fail together. If a deadlock occurs, we retry the transaction a few times. This approach guarantees that we don’t end up with inconsistent data states.

## Data Retention and Indexing

One important consideration with the tombstone approach is data retention. We need to define a policy for how long tombstone entries are kept to prevent the table from growing indefinitely. For instance, we might keep tombstone entries for a month and then perform a full re-sync if a client connects after this period. This helps maintain performance and manage storage.

**Indexing**:
- To optimize our queries, we’ve indexed the tombstone table on the workspace ID and version columns. This ensures efficient data retrieval, especially as the table grows.

## Future Improvements

Looking ahead, we aim to move to a RowVersion-based approach for even more efficient data synchronization. This method involves removing version numbers from rows and tracking in-memory when each client last pulled data. By diffing the sets of IDs between the client and server, we can determine which items have been deleted without needing a tombstone table.

Additionally, we plan to set up a public API for customers, including endpoints for data synchronization and tracking deleted items. This will provide a more efficient way for clients to sync data and track deletions without over-fetching.

## Conclusion

In this post, we explored our journey from using soft deletes to implementing a tombstone table for handling deletions in Rough.app. We discussed the benefits and challenges of each approach, as well as our strategies for error handling, data retention, and indexing. As we continue to refine our methods, we look forward to achieving even greater efficiency and providing robust data synchronization for our users.

If you have any thoughts or experiences with similar challenges, I’d love to hear from you!
