# Notice: Repository Deprecation
This repository is deprecated and no longer actively maintained. It contains outdated code examples or practices that do not align with current MongoDB best practices. While the repository remains accessible for reference purposes, we strongly discourage its use in production environments.
Users should be aware that this repository will not receive any further updates, bug fixes, or security patches. This code may expose you to security vulnerabilities, compatibility issues with current MongoDB versions, and potential performance problems. Any implementation based on this repository is at the user's own risk.
For up-to-date resources, please refer to the [MongoDB Developer Center](https://mongodb.com/developer).

# PANW quick restore at DB Level 🧹
Step by step guide to quick DB level restore 

Example Commands
Backup a Specific Database:
```
mongodump --db yourDatabase --out /path/to/backup/
```
Restore a Specific Database:
```
mongorestore --db yourDatabase --drop /path/to/backup/yourDatabase
```
Using Change Streams for Minimizing Downtime
To minimize downtime while restoring a specific database, you can use Change Streams to capture changes during the restore process and replay them afterward, similar to the previous explanation. Here's how you can integrate this with a per-database restore:

Start Change Stream for the Specific Database:
```
const { MongoClient } = require('mongodb');

async function startChangeStream() {
  const client = new MongoClient('mongodb://localhost:27017');
  await client.connect();

  const db = client.db('yourDatabase');
  const changeStream = db.watch();

  let changes = [];
  changeStream.on('change', (next) => {
    // Save change events to a temporary storage for replaying later
    changes.push(next);
  });

  // Save the changeStream reference to stop it later if needed
  return { client, changeStream, changes };
}
```
Perform the Snapshot Restore for the Specific Database:
```
mongorestore --db yourDatabase --drop /path/to/backup/yourDatabase
```
Apply the Captured Changes:
```
async function applyChanges(changes) {
  const client = new MongoClient('mongodb://localhost:27017');
  await client.connect();

  const db = client.db('yourDatabase');

  for (const change of changes) {
    // Apply each change based on its type (insert, update, delete)
    switch (change.operationType) {
      case 'insert':
        await db.collection(change.ns.coll).insertOne(change.fullDocument);
        break;
      case 'update':
        await db.collection(change.ns.coll).updateOne(
          { _id: change.documentKey._id },
          { $set: change.updateDescription.updatedFields }
        );
        break;
      case 'delete':
        await db.collection(change.ns.coll).deleteOne({ _id: change.documentKey._id });
        break;
    }
  }
}
```
Cleanup:
```
async function cleanup(client, changeStream) {
  await changeStream.close();
  await client.close();
}
```
Full Workflow Example:
Start Change Stream and Capture Changes:
```
const { client, changeStream, changes } = await startChangeStream();
```
Restore the Specific Database:
```
mongorestore --db yourDatabase --drop /path/to/backup/yourDatabase
```
Apply Captured Changes:
```
await applyChanges(changes);
```
```
await cleanup(client, changeStream);
```
