## Bootstrapping

## The Whole Picture

1. `StoreManager` creates `PartialStore` or `FullStore` for each model to synchronize in-memory data with the IndexedDB (1). Related functions: .
2. Generate metadata of the database to store the current user's data. Then connect to IndexedDB and determine if the database need to be migrated. Metadata would be written into the database as well (2.1) (2.2).
3. Identify the bootstrap type to decide the bootstrapping method (3). Related method `Database.requiredBootstrap`.
4. Bootstrap the database (4). Related function: `Database.bootstrap`.
5. For a full or partial bootstrap, retrieve the model from the server (5). Related function: `BootstrapHelper.fullBootstrap`
6. Store model data for persistence (6).
7. Load data that needs immediate hydration into memory. Initialize in-memory model objects, start observability, apply delta for local bootstrapping, load persisted transactions, save data into IndexedDB by flushing stores, monitor remote changes, and schedule the execution of persisted transactions (7).

## Create Stores

During the bootstrapping process, the initial step is to construct the `StoreManager`. In the `constructor` of `StoreManager`, it constructs `Store` instances for each model registered on the `ModelRegistry`. `Store` manages a model's corresponding table in IndexedDB.

Models have different `loadStrategy`, so LSE generates different stores for them. One is the `PartialStore`, which manages models using `partial` load strategy. The other is the `FullStore`, used for all other models. Both classes extend `Store`.

![[Pasted image 20241005161425.png]]

When a `Store` is created, it calculates the model's hash. The hash will be used a the name of the table in the database.

```jsx
this.storeName = $1(e + Me.propertyHashOfModel(e)); // bf9... for "Issue"
```

For example, `Comment` model has `storeName` `1a1def71af54d8ede0e6fb8a0b8e71b6`, and there will be a table with the same hash.

![[Pasted image 20241005162006.png]]

## Create Databases

Assuming you have one account logged in and a single workspace loaded, LSE will maintain two databases in IndexedDB, `linear_databases` and `linear_(hash)` (in my case `linear_07c3f5816b277c8158a9f6526b158ba6`). `linear_databases` contains metadata of other databases. `linear_07c3f5` stores my data in my private workspace..

Metadata is generated in the `databaseInfo` method and includes:

During the bootstrapping process, LSE will determine whether to create or migrate these databases and establish connections to them.

1. `name` - The name of the database. Its hash is related to `userId`, `version` and `userVersion`. So apparently, if you have different user identity, you would have multi databases.
2. `schemaHash` - This metadata is used for database migration. The next section gives a simple introduction of how it is calculated.
3. `schemaVersion` - This is a local incremental counter used to determine if a database migration is needed. If the new `schemaHash` differs from what is stored in indexDB, the counter will increment. The updated version is then passed to [`IndexDB.open`](http://indexdb.open/) as its second parameter to check if the database requires a migration.

`linear_07c3f5` contains tables each dedicated to a specific type of model. Additionally, there are two special tables: `_meta` and `_transaction`. The `_meta` table stores persistence information for each model and other important information like `lastSyncId`, while the `_transaction` table contains mutations that have not yet been sent to the server.

## Determined Bootstrapping Type

`Database.requiredBootstrap` would determine which type of bootstrapping should be performed. There are three types of bootstrapping:

1. `full`: LSE would load everything from the server.
2. `local`: The application should load from the local database.
3. `partial` : The application should .

We would only cover full bootstrapping in this blog.

### `lastSyncId`

## Full Bootstrapping

- `Database.bootstrap`
- `BootstrapHelper.fullBootstrap`

When Linear does a full bootstrapping, it would send a request like this:

```
https://client-api.linear.app/sync/bootstrap?type=full&onlyModels=WorkflowState,IssueDraft,Initiative,ProjectMilestone,ProjectStatus,TextDraft,ProjectUpdate,IssueLabel,ExternalUser,CustomView,ViewPreferences,Roadmap,RoadmapToProject,Facet,Project,Document,Organization,Template,Team,Cycle,Favorite,CalendarEvent,User,Company,IssueImport,IssueRelation,TeamKey,UserSettings,PushSubscription,Activity,ApiKey,EmailIntakeAddress,Emoji,EntityExternalLink,GitAutomationTargetBranch,GitAutomationState,Integration,IntegrationsSettings,IntegrationTemplate,NotificationSubscription,OauthClientApproval,Notification,OauthClient,OrganizationDomain,OrganizationInvite,ProjectLink,ProjectUpdateInteraction,InitiativeToProject,Subscription,TeamMembership,TimeSchedule,TriageResponsibility,Webhook,WorkflowCronJobDefinition,WorkflowDefinition,ProjectRelation,DiaryEntry,Reminder
```

Note that there are two parameters:

1. `type`. It's either `full` or `partial`.
2. `onlyModels`. Names of the models that are going to load.

The response is a stream of objects:

```jsx
{"id":"8ce3d5fe-07c2-481c-bb68-cd22dd94e7de","createdAt":"2024-07-03T11:37:04.865Z","updatedAt":"2024-07-03T11:37:04.865Z","userId":"4e8622c7-0a24-412d-bf38-156e073ab384","issueId":"01a3c1cf-7dd5-4a13-b3ab-a9d064a3e31c","events":[{"type":"issue_deleted","issueId":"01a3c1cf-7dd5-4a13-b3ab-a9d064a3e31c","issueTitle":"Load data from remote sync engine."}],"__class":"Activity"}
{"id":"ec9ec347-4f90-465c-b8bc-e41dae4e11f2","createdAt":"2024-07-03T11:37:06.944Z","updatedAt":"2024-07-03T11:37:06.944Z","userId":"4e8622c7-0a24-412d-bf38-156e073ab384","issueId":"39946254-511c-4226-914f-d1669c9e5914","events":[{"type":"issue_deleted","issueId":"39946254-511c-4226-914f-d1669c9e5914","issueTitle":"Reverse engineering Linear's Sync Engine"}],"__class":"Activity"}
// many lines omitted here
_metadata_={"method":"mongo","lastSyncId":2326713666,"subscribedSyncGroups":["89388c30-9823-4b14-8140-4e0650fbb9eb","4e8622c7-0a24-412d-bf38-156e073ab384","AD619ACC-AAAA-4D84-AD23-61DDCA8319A0","CDA201A7-AAAA-45C5-888B-3CE8B747D26B"],"databaseVersion":948,"returnedModelsCount":{"Activity":6,"Cycle":2,"DocumentContent":5,"Favorite":1,"GitAutomationState":3,"Integration":1,"Issue":3,"IssueLabel":4,"NotificationSubscription":2,"Organization":1,"Project":2,"ProjectStatus":5,"Team":1,"TeamKey":1,"TeamMembership":1,"User":1,"UserSettings":1,"WorkflowState":7,"Initiative":1,"SyncAction":0}}
```

Each line (expect for the last) would be an instance of a model. Take an object for example:

```json
{
  "id": "556c8983-ca05-41a8-baa6-60b6e5d771c8",
  "createdAt": "2024-01-22T01:02:41.099Z",
  "updatedAt": "2024-05-16T08:23:31.724Z",
  "number": 1, // created sequence?
  "title": "Welcome to Linear ðŸ‘‹", // text encoding problem
  "priority": 1,
  "boardOrder": 0,
  "sortOrder": -84.71,
  "startedAt": "2024-05-16T08:16:57.239Z",
  "labelIds": ["30889eaf-fac5-4d4d-8085-a4c3bd80e588"],
  "teamId": "89388c30-9823-4b14-8140-4e0650fbb9eb",
  "projectId": "3e7ada3c-f833-4b9c-b325-6db37285fa11",
  "projectMilestoneId": "397b95c4-3ee2-47b0-bad1-d6b1c7003616",
  "subscriberIds": ["4e8622c7-0a24-412d-bf38-156e073ab384"],
  "previousIdentifiers": [],
  "assigneeId": "4e8622c7-0a24-412d-bf38-156e073ab384",
  "stateId": "030a7891-2ba5-4f5b-9597-b750950cd866",
  "reactionData": [],
  "__class": "Issue"
}
```

It is an `Issue`. The `id` is the primary key in the issue table. Additionally, they use UUIDs to generate IDs and establish references among models using these IDs. Also note that LSE employs fractional indexing to handle sorting.

The last line of the response is a metadata object. The `lastSyncId` `databaseVersion` would later be saved into metadata.

```json
{
  "method": "mongo",
  "lastSyncId": 2326713666,
  "subscribedSyncGroups": [
    "89388c30-9823-4b14-8140-4e0650fbb9eb",
    "4e8622c7-0a24-412d-bf38-156e073ab384",
    "AD619ACC-AAAA-4D84-AD23-61DDCA8319A0",
    "CDA201A7-AAAA-45C5-888B-3CE8B747D26B"
  ],
  "databaseVersion": 948,
  "returnedModelsCount": {
    "Activity": 6,
    "Cycle": 2,
    "DocumentContent": 5,
    "Favorite": 1,
    "GitAutomationState": 3,
    "Integration": 1,
    "Issue": 3,
    "IssueLabel": 4,
    "NotificationSubscription": 2,
    "Organization": 1,
    "Project": 2,
    "ProjectStatus": 5,
    "Team": 1,
    "TeamKey": 1,
    "TeamMembership": 1,
    "User": 1,
    "UserSettings": 1,
    "WorkflowState": 7,
    "Initiative": 1,
    "SyncAction": 0
  }
}
```

## Init In-Memory Data and Database

### Object Pool

## Lazy Loading & Hydration

LSE doesn't load everything into memory at once. When you lick on issues or projects, there might be network requests to load additional data from the server (sometimes from IndexedDB). This process is known as hydration.

Classes with `hydrate` method could be hydrated, including `Model` `LazyReferenceCollection` `LazyReference` `RequestCollection` and `LazyBackReference` .

Hydration for lazy reference collections are much more complex than

`processPartialIndexInfoForModel` generates an array that displays which models have back references to a given model. This process is performed recursively up to three layers deep.

```jsx
[
  {
    path: "issueId",
    referencedModelName: "Issue",
  },
  {
    path: "issue.teamId",
    referencedModelName: "Team",
  },
  {
    path: "issue.cycleId",
    referencedModelName: "Cycle",
  },
  {
    path: "issue.projectId",
    referencedModelName: "Project",
  },
  {
    path: "issue.projectMilestoneId",
    referencedModelName: "ProjectMilestone",
  },
  {
    path: "issue.creatorId",
    referencedModelName: "User",
  },
  {
    path: "issue.assigneeId",
    referencedModelName: "User",
  },
  {
    path: "issue.subscriberIds",
    referencedModelName: "User",
  },
  {
    path: "issue.stateId",
    referencedModelName: "WorkflowState",
  },
];
```

![[Pasted image 20241005165418.png]]

- [ ] this part has not finished yet

`resolveCoveringPartialIndexValues`

```jsx
[
  "issueId-c775f0f8-7c85-4da6-998a-939eccd337ed",
  "369af3b8-7d07-426f-aaad-773eccd97202",
  "issue.subscriberIds-e86b9ddf-819e-4e77-8323-55dd488cb17c",
  "issue.stateId-1595ecd3-0185-4b65-b172-141040c346aa",
];
```

### BatchUpdater

## Takeaway

# Part III: Syncing

In the final part, we would talk about how LSE sync between clients and the server. What exactly happens when we change the assignee of an issue? How does LSE manage undo/redo functionality, networking, error handling, offline caching, and many other details within these two lines?

```jsx
issue.assignee = user;
issue.save();
```

## The Whole Picture

First, let's review the entire process, then we can delve into the details.

![[Pasted image 20241009161445.png]]

1. When a property is assigned a value, decorators will bookkeeper what property has been changed and what the new value and the old value are (1), and perhaps also update back references.
2. Once `save()` is called, the model call `SyncedStore` , subsequently `SyncedClient` and `TransactionQueue` to generate a n `Transaction` (2).
3. The generated transaction is pushed into a queue (sometimes sent immediately) and saved into the `__transactions` table in IndexedDB (3). `TransactionQueue` schedule a timer to repeatedly sending these pending transactions to the server.
4. Later, it is sent to the backend along with other transactions in a batch, a process known as transaction execution (4).
5. If the transaction is accepted by the backend, it is removed from the `__transactions` table (5) (6).
6. Subsequently, the backend sends a delta to `SyncClient` (7), informing clients of changes made on the server. `SyncClient` rebases transactions that haven't been accepted by the server, modifies in-memory models (8), and updates the tables in IndexedDB (9).

## Find Out What Has Been Changed

- `M1`
- `as#propertyChanged`
- `as#markPropertyChanged`
- `as#referencedPropertyChanged`
- `as#updateReferencedModel`

As we have discussed in section "Observability", LSE uses a decorator `M1` to make a property observable. It also plays a critical part in generating transactions. When a property of a model is assigned with a new value, the setter would intercept the assignment and call `propertyChanged` to record the name of that property, the old value and the new value. `markPropertyChanged` would then be called to serialize the old value and store it in `modifiedProperties`.

<aside>
💡 In addition to 
</aside>

## Generate Transactions

There are various types of transactions:

| Minimized name | Possible original names | Description                                                             |
| -------------- | ----------------------- | ----------------------------------------------------------------------- |
| `M3` `Zo`      | `BaseTransaction`       | The base class of all transactions.                                     |
| `Hu`           | `CreationTransaction`   | The transaction to add an model.                                        |
| `zu`           | `UpdateTransaction`     | The transaction to update properties on an existing model.              |
| `g3`           | `DeleteTransaction`     | The transaction to delete a model. E.g. deleting a comment of an issue. |
| `m3`           | `ArchiveTransaction`    | The transaction to archive a model. E.g. deleting an issue.             |
| `y3`           | `UnarchiveTransaction`  | The transaction to unarchive a model.                                   |

`uce` (`TransactionQueue`) provides methods to create and execute transactions. But how does LSE determine which type of transaction it should generate, and what is the content of a transaction?

When the model object's `save` method is called, it invokes the `save` method of `SyncedStore`. If the model exists in `SyncClient`, an `UpdateTransaction` is generated. During the construction of `UpdateTransaction`, the model’s `changeSnapshot` function is called. Ultimately, an object is generated to represent the changes:

```jsx
{
  assigneeId: {
    original: this.modifiedProperties[n],
    updatedFrom: this.modifiedProperties[n],
    updated: i,
    unoptimizedUpdated: s
  }
}
```

## Transaction Queue

`TransactionQueue` use four arrays to manage transactions.

1. `createdTransactions` When a transaction is queued, it first goes to `createdTransactions`. A `commitCreatedTransactions` scheduler periodically moves all transactions in this array to the end of `queuedTransactions`.
2. `queueTransactions` These transactions are pushed to a queue, waiting to be executed. Transactions would all be saved into `__transactions` when are are move to `queuedTransactions` .
3. `executingTransactions` These transactions have been sent to the server but have not been accepted yet. The `TransactionQueue` has a `dequeueTransaction` scheduler that periodically moves transactions from `queuedTransactions` to `executingTransactions` in a FIFO manner.
4. `completedButUnsyncedTransactions` These transactions have been sent to the server and accepted, but the corresponding delta has not been received. When a set of transactions from `executingTransactions` is executed, they are removed from `executingTransactions` and pushed to `completedButUnsyncedTransactions`. Note that `completedButUnsyncedTransactions` are removed from `__transactions` in IndexedDB.
5. `persistedTransactionsEnqueue` When the database bootstraps, transactions saved in `__transactions` would be loaded into this array. After remote updates have been processed, they are moved to `queueTransactions` and waiting to be executed. During processing remove updates, these transactions may need to rebase on deltas.

At any given moment, transactions in `completedButUnsyncedTransactions` must be created earlier than those in `executingTransactions`, and the same applies to any other two arrays. This is important because the sequence plays a role when rebasing transactions.

## Generating GraphQL Mutating Queries

Before sending a batch of transactions to the server, it is converted into GraphQL mutating queries. Each transaction calls the `prepare` function to generate parameters (there are a lot of details but I am not going to cover in this post). These parameters are then used in `TransactionExecutor.execute` to create the query path and parameters.

```jsx
mutation IssueUpdate($issueUpdateInput: IssueUpdateInput!) {
  issueUpdate(id: "a8e26eed-7ad4-43c6-a505-cc6a42b98117", input: $issueUpdateInput
) { lastSyncId } }
```

```jsx
{
    "issueUpdateInput": {
        "assigneeId": "e86b9ddf-819e-4e77-8323-55dd488cb17c"
    }
}
```

## Delta, Actions and Rebasing

Delta from the server would be executed in the `applyDelta` method. After we modified the assignee, the delta would look like this:

A delta includes one or more actions, each identified by an `action` type. The current types are:

1. `I` for insertion
2. `U` for updating
3. `A` for archiving
4. `D` for deleting
5. `C` remains unknown
6. `G` remains unknown
7. `S` remains unknown
8. `V` remains unknown

For the updating action, LSE broadcasts all properties of the model, not just the changed ones. The model’s `updateFromData` would be called to update all properties.

```jsx
[
  {
    id: 2361610825,
    modelName: "Issue",
    modelId: "a8e26eed-7ad4-43c6-a505-cc6a42b98117",
    action: "U",
    data: {
      id: "a8e26eed-7ad4-43c6-a505-cc6a42b98117",
      title: "Connect to Slack",
      number: 3,
      teamId: "369af3b8-7d07-426f-aaad-773eccd97202",
      stateId: "28d78a58-9fc1-4bf1-b1a3-8887bdbebca4",
      labelIds: [],
      priority: 3,
      createdAt: "2024-05-29T03:08:15.383Z",
      sortOrder: -12246.37,
      updatedAt: "2024-07-13T06:25:40.612Z",
      assigneeId: "e86b9ddf-819e-4e77-8323-55dd488cb17c",
      boardOrder: 0,
      reactionData: [],
      subscriberIds: ["e86b9ddf-819e-4e77-8323-55dd488cb17c"],
      prioritySortOrder: -12246.37,
      previousIdentifiers: [],
    },
    __class: "SyncAction",
  },
  {
    id: 2361610826,
    modelName: "IssueHistory",
    modelId: "ac1c69bb-a37e-4148-9a35-94413dde172d",
    action: "I",
    data: {
      id: "ac1c69bb-a37e-4148-9a35-94413dde172d",
      actorId: "e86b9ddf-819e-4e77-8323-55dd488cb17c",
      issueId: "a8e26eed-7ad4-43c6-a505-cc6a42b98117",
      createdAt: "2024-07-13T06:25:40.581Z",
      updatedAt: "2024-07-13T06:25:40.581Z",
      toAssigneeId: "e86b9ddf-819e-4e77-8323-55dd488cb17c",
    },
    __class: "SyncAction",
  },
  {
    id: 2361610854,
    modelName: "Activity",
    modelId: "1321dc17-cceb-4708-8485-2406d7efdfc5",
    action: "I",
    data: {
      id: "1321dc17-cceb-4708-8485-2406d7efdfc5",
      events: [
        {
          type: "issue_updated",
          issueId: "a8e26eed-7ad4-43c6-a505-cc6a42b98117",
          issueTitle: "Connect to Slack",
          changedColumns: ["subscriberIds", "assigneeId"],
        },
      ],
      userId: "e86b9ddf-819e-4e77-8323-55dd488cb17c",
      issueId: "a8e26eed-7ad4-43c6-a505-cc6a42b98117",
      createdAt: "2024-07-13T06:25:41.924Z",
      updatedAt: "2024-07-13T06:25:41.924Z",
    },
    __class: "SyncAction",
  },
];
```

When applying a delta, it might conflict with local changes (transactions). For updating actions and transactions, LSE needs to rebase transactions onto the model modified by the action. This starts with the `TransactionQueue.rebaseTransactions` method.

LSE rebase all transactions in a specific order: `completedButUnsyncedTransactions`, `executingTransactions`, `queuedTransactions`, and `persistedTransactionsEnqueue`, as mentioned earlier. In terms of rebasing, it:

1. Adjusts the original transaction values to those specified by the action. For example, if I change the title of an issue from "A" to "B", and my colleagues change it from “A” to “C”, during rebasing on my device, my transaction would update from `A -> B` to `C -> B`.
2. Updates the model’s properties again since they have been altered by the action. This inevitably leads to some back-and-forth value assignments.

As Tuomas mentioned in his talk, this follows a last-writer-wins strategy.

### Side effects on the server

Issue history.

If you made any changes offline and then reload the page, it may seem that the changes are lost. However, they are not. The transactions are stored in the database and will be sent to the server once you are back online. **Linear does not apply these transactions locally as they are designed to be executed only on the backend.** In other words, the frontend and backend use different methods to update data. The backend needs to check authentication, generate history records, send emails, and so on. Linear refers to these as 'side effects'.

<aside>
💡 Linear has to do some side effects on the server.
</aside>

## Error handling

## Undo & redo

Undo and redo operations in LSE are based on transactions. Each transaction type has a specific `undoTransaction` method. This method executes the undo logic and returns another transaction for redo purposes. For example, the `undoTransaction` method of `UpdateTransaction` reassigns the model's property with its previous values and returns another `UpdateTransaction` to the `UndoQueue`. It is important to note that when a transaction performs its undo logic, a new transaction is created and added to the `queuedTransactions` to ensure synchronization.

But how does the `UndoManager` determine which transactions should be appended to the undo redo stack? It turns out that Linear UI is responsible for identifying the differences.

```jsx
n.title !== d &&
  o.undoQueue.addOperation(
    s.jsxs(s.Fragment, {
      children: [
        "update title of issue ",
        s.jsx(Le, {
          model: n,
        }),
      ],
    }),
    () => {
      //
      (n.title = d), n.save();
    }
  );
```

When an edit is made, the UI calls `UndoQueue.addOperation` to have `UndoQueue` subscribe to the next `transactionQueuedSignal` and create an undo item. This signal is triggered when transactions are added to `queuedTransactions`. The subscription ends when a callback finishes execution. What is in that callback? It is `save()`, our old friend! So when we perform an undo, although the signal is triggered, since `UndoQueue` is not listening to it, no extra undo item will be created.

## Add, Delete and Archive

For `DeleteTransaction`, `ArchiveTransaction`, and `UnarchiveTransaction`, there are specific methods on models to generate these transactions. We would not take a deeper investigation into them.

## Takeaway

A quick summary of this part:

1. **Client-side operations will never directly alter the tables in the local database!** Only data synced from the server would do that. You can consider
2. Unlike OT, which only sends an acknowledgment signal to the mutator client, **LSE consistently sends all properties of modified models to all clients, regardless of whether one of them is the mutator**. It really simplifies things.
3. LSE use simple **last-writer-win strategy to handle conflicts**, and only handle conflicts for updating actions.

# Conclusion

# Lookup

| Minimized names                                                                                                                                         | Possible original names                                                     | A short description                                                                                                                                                            |
| ------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Me` `rr`                                                                                                                                               | `ModelRegistry`                                                             |                                                                                                                                                                                |
| `w`                                                                                                                                                     | `ApplicationStore`                                                          |                                                                                                                                                                                |
| `Ln`                                                                                                                                                    | `makeObservable`                                                            | `MobX` API to make an object observable.                                                                                                                                       |
| `sg` `hf` `km`                                                                                                                                          | `SyncedStore`                                                               | It is the `store` in models. It provides lots of methods to get or manipulate models.                                                                                          |
| `Hr`                                                                                                                                                    | `SyncedClient`                                                              |                                                                                                                                                                                |
| `Dt`                                                                                                                                                    | `Database`                                                                  |                                                                                                                                                                                |
| `jn`                                                                                                                                                    | `DatabaseManager`                                                           |                                                                                                                                                                                |
| `cce`                                                                                                                                                   | `StoreManager`                                                              | The in-memory store manager. It manages lots of object stores. Each store maps to a model, and maps to a table in the database.                                                |
| `TE`                                                                                                                                                    | `FullStore`                                                                 | A store which content should be loaded all-at-once.                                                                                                                            |
| `Jm` `p3`                                                                                                                                               | `PartialStore`                                                              | A store which content can be partially loaded. It extends `FullStore` .                                                                                                        |
| `uce`                                                                                                                                                   | `TransactionQueue`                                                          |                                                                                                                                                                                |
| `A8` `sd`                                                                                                                                               | `GraphQLClient`                                                             | A GraphQL client. It would be used by many classes.                                                                                                                            |
| `w`                                                                                                                                                     | `Property`                                                                  | This refers to a self-owned property of a model. For example, `title` is a property of `Issue`.                                                                                |
| `_g`                                                                                                                                                    | `EphemeralProperty`                                                         | This is also a self-owned property of a model but is not stored in the database. For example, `lastUserInteraction` is an ephemeral property of `User`.                        |
|                                                                                                                                                         |                                                                             | For example, `documentContentId` of `Issue` refers to the document model's ID of the issue's description.                                                                      |
| `pe`                                                                                                                                                    | `Reference`                                                                 |                                                                                                                                                                                |
| `xe` `Nt`                                                                                                                                               | `ReferenceCollection` `LazyReferenceCollection`                             | This is used for 1 to many relationships. For example, `issues` is a lazy reference collection of `IssueLabel`. And `notifications` is a reference collection of `Initiative`. |
| `pe` `hr` `Ue` `g5` `dt` `kl`                                                                                                                           | `WithBackReference` `LazyWithBackReference` `Reference` `LazyBackReference` | These 6 decorators are used to declare reference and back references with different options.                                                                                   |
| `ReferenceArray`                                                                                                                                        | `ii`                                                                        |                                                                                                                                                                                |
| Many to many relationships. For example, `labels` are reference array of `Issue`. (But `issues` are lazy reference collection of `Label`).              |                                                                             |                                                                                                                                                                                |
| `as`                                                                                                                                                    | `Model`                                                                     |                                                                                                                                                                                |
| The base class that each model will inherit. The class declares a `store` and `__mobx` . The are the critical parts of data fetching and observability. |                                                                             |                                                                                                                                                                                |
| The class provides lots of static methods to register models, properties and their metadata.                                                            |                                                                             |                                                                                                                                                                                |
| `Qc`                                                                                                                                                    | `UUID`                                                                      | A helper function to generate UUID.                                                                                                                                            |
