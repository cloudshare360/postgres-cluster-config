Sure thing! Here's an updated version of the self-executing script that:

✅ Accepts `regions`, `engine`, and optional `clusterIdentifier`  
✅ Uses **AWS SDK v3 + ES6**  
✅ **Returns and logs the full output as a JSON object**

---

### ✅ Final Version – Output as JSON

```js
// auroraEndpoints.mjs
import {
  RDSClient,
  DescribeDBClustersCommand,
  DescribeDBInstancesCommand,
} from '@aws-sdk/client-rds';

/**
 * Fetch Aurora PostgreSQL endpoints.
 * @param {Object} params
 * @param {string[]} params.regions - AWS regions to search
 * @param {string} params.engine - Aurora engine type (e.g., "aurora-postgresql")
 * @param {string|null} params.clusterIdentifier - Optional cluster ID, or null to scan all
 * @returns {Promise<Object[]>} List of endpoint metadata
 */
const getAuroraEndpoints = async ({ regions, engine, clusterIdentifier = null }) => {
  const allEndpoints = [];

  for (const region of regions) {
    const client = new RDSClient({ region });

    try {
      const clusterCommandInput = clusterIdentifier
        ? { DBClusterIdentifier: clusterIdentifier }
        : {};

      const { DBClusters } = await client.send(
        new DescribeDBClustersCommand(clusterCommandInput)
      );

      for (const cluster of DBClusters) {
        const clusterMemberIds = cluster.DBClusterMembers.map(m => m.DBInstanceIdentifier);

        const { DBInstances } = await client.send(new DescribeDBInstancesCommand({}));
        const instances = DBInstances.filter(
          i =>
            clusterMemberIds.includes(i.DBInstanceIdentifier) &&
            i.Engine === engine
        );

        for (const instance of instances) {
          const memberMeta = cluster.DBClusterMembers.find(
            m => m.DBInstanceIdentifier === instance.DBInstanceIdentifier
          );

          const isWriter = memberMeta?.IsClusterWriter === true;

          let role = 'read';
          if (isWriter) role = 'write';
          else if (instance.Engine?.includes('global')) role = 'global';

          allEndpoints.push({
            clusterId: cluster.DBClusterIdentifier,
            id: instance.DBInstanceIdentifier,
            endpoint: instance.Endpoint?.Address,
            port: instance.Endpoint?.Port,
            instanceClass: instance.DBInstanceClass,
            region,
            role,
          });
        }
      }
    } catch (err) {
      console.warn(`⚠️ Error in region ${region}: ${err.message}`);
    }
  }

  return allEndpoints;
};

// 🟢 Self-executing main function
const main = async () => {
  const params = {
    regions: ['us-east-1', 'us-west-2'],
    engine: 'aurora-postgresql',
    clusterIdentifier: null, // or 'your-cluster-id'
  };

  const endpoints = await getAuroraEndpoints(params);

  // ✅ Log as JSON
  console.log('\n✅ Aurora PostgreSQL Endpoints (JSON):\n');
  console.log(JSON.stringify(endpoints, null, 2));
};

main();
```

---

### 🟢 How to Run

1. Save as `auroraEndpoints.mjs`  
2. Install the AWS SDK v3:

   ```bash
   npm install @aws-sdk/client-rds
   ```

3. Run:

   ```bash
   node auroraEndpoints.mjs
   ```

---

### 🧪 Example JSON Output

```json
[
  {
    "clusterId": "aurora-db-cluster",
    "id": "writer-instance-1",
    "endpoint": "writer.cluster-id.us-east-1.rds.amazonaws.com",
    "port": 5432,
    "instanceClass": "db.r6g.large",
    "region": "us-east-1",
    "role": "write"
  },
  {
    "clusterId": "aurora-db-cluster",
    "id": "reader-instance-1",
    "endpoint": "reader.cluster-id.us-west-2.rds.amazonaws.com",
    "port": 5432,
    "instanceClass": "db.r6g.large",
    "region": "us-west-2",
    "role": "read"
  }
]
```

---

Would you like to also **save this JSON to a file** (`endpoints.json`) or format it as a **Sequelize config** object next?
