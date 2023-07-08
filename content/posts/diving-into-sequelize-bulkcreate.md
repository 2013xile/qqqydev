---
title: Diving into bulkCreate method in Sequelize
date: "2023-07-08"
categories:
  - "NodeJS"
  - "Backend"
---

# Background

During my recent development work on a data synchronization feature using `bulkCreate` method in [Sequelize](https://sequelize.org/api/v6/class/src/model.js~model#static-method-bulkCreate), I encountered an issue with incorrect returning IDs when using the `updateOnDuplicate: true` option. This led me to suspect that Sequelize generates the returning IDs through inference. To confirm my hypothesis, I decided to investigate the source code of Sequelize.

# How Sequelize generate returning IDs in `bulkCreate`

Despite the extensive codebase of Sequelize, locating the `bulkCreate` method was straightforward. A global search within the codebase led me directly to the `src/model.js` file, where I found the relevant comment confirming my initial assumption.

```JavaScript
/*
  * The success handler is passed an array of instances, but please notice that these may not completely represent the state of the rows in the DB. This is because MySQL
  * and SQLite do not make it easy to obtain back automatically generated IDs and other default values in a way that can be mapped to multiple records.
  * To obtain Instances for the newly created values, you will need to query for them again.
  */
```

Next, I proceeded to investigate this method by following the code snippets below:

```JavaScript
// src/model.js
const results = await model.queryInterface.bulkInsert(model.getTableName(options), records, options, fieldMappedAttributes);

static get queryInterface() {
  return this.sequelize.getQueryInterface();
}
```

```JavaScript
// src/sequelize.js
this.queryInterface = this.dialect.queryInterface;
```

```JavaScript
// src/dialects/sqlite/index.js
// Just take SQLite as an example.
this.queryInterface = new SQLiteQueryInterface(
  sequelize,
  this.queryGenerator
);

// src/dialects/sqlite/query-interface.js
class SQLiteQueryInterface extends QueryInterface { 
  // ...
}
```

```JavaScript
// src/dialects/abstract/query-generator.js
async bulkInsert(tableName, records, options, attributes) {
  options = { ...options };
  options.type = QueryTypes.INSERT;

  const results = await this.sequelize.query(
    this.queryGenerator.bulkInsertQuery(tableName, records, options, attributes),
    options
  );

  return results[0];
}
```

```JavaScript
// src/sequelize.js
async query(sql, options) {
  // ...
  const query = new this.dialect.Query(connection, this, options);
  // ...
  return await query.run(sql, bindParameters);
  // ...
}
```

```JavaScript
// src/dialects/sqlite/query.js
async run(sql, parameters) {
  // ...
  resolve(query._handleQueryResponse(this, columnTypes, executionError, results, errForStack.stack));
  // ...
}
```

And finally I got what I was aiming for: 

```JavaScript
// src/dialects/sqlite/query.js
_handleQueryResponse(metaData, columnTypes, err, results, errStack) {
  // ...
  // add the inserted row id to the instance
  if (this.isInsertQuery(results, metaData) || this.isUpsertQuery()) {
    this.handleInsertQuery(results, metaData);
    if (!this.instance) {
      // handle bulkCreate AI primary key
      if (
        metaData.constructor.name === 'Statement'
        && this.model
        && this.model.autoIncrementAttribute
        && this.model.autoIncrementAttribute === this.model.primaryKeyAttribute
        && this.model.rawAttributes[this.model.primaryKeyAttribute]
      ) {
        const startId = metaData[this.getInsertIdField()] - metaData.changes + 1;
        result = [];
        for (let i = startId; i < startId + metaData.changes; i++) {
          result.push({ [this.model.rawAttributes[this.model.primaryKeyAttribute].field]: i });
        }
      } else {
        result = metaData[this.getInsertIdField()];
      }
    }
  }
  // ...
}
```

Upon the `_handleQueryResponse` method, it became apparent that Sequelize utilizes the last inserted ID and infers the remaining IDs when using `bulkCreate`. Consequently, when using `updateOnDuplicate: true` option, there is a possibility of receiving incorrect IDs in the return.

> **A handy tip for obtaining accurate IDs**  
Enhance the data table by introducing a `batch` field, assigning a batch number to each set of data being inserted. Subsequently, you can query the inserted IDs by using the corresponding batch number.

# Risk of using `updateOnDuplicate: true`

While utilizing the `updateOnDuplicate: true` option, I observed that even though duplicate rows are skipped, the `autoIncrement` key continues to increment. From the [documentation of MySQL](https://dev.mysql.com/doc/refman/8.0/en/insert-on-duplicate.html), it said: 

> The effects are not quite identical: For an InnoDB table where a is an auto-increment column, the INSERT statement increases the auto-increment value but the UPDATE does not.

The database follows a process of attempting to insert, incrementing the ID, and then the duplicate is detected. However, once the ID is auto-incremented, it cannot be rolled back.  

This situation presents a potential risk when utilizing a master-slave database architecture. As the ID is auto-incremented but duplicate is detected, the operation do not leave bin logs, there is a chance for key conficts to arise when the master goes down and the slave is promoted as the new master.
