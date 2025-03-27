Absolutely! Here's a clean and modern **ES6 AWS SDK v3** method where you can pass the parameters:

- `regions`: array of AWS region strings  
- `engine`: RDS engine type (e.g. `"aurora-postgresql"`)  
- `clusterIdentifier`: optional Aurora cluster identifier

The method will fetch Aurora instance endpoints and classify them as `write`, `read`, or `global`.

---

### ✅ ES6 Function with Parameters (Self-Executing)

```js
// auroraEndpoints.mjs
import {
  RDSClient,
  DescribeDBClustersCommand,
  DescribeDBInstancesCommand,
} from '@aws-sdk/client-rds';

/**
 * Fetch Aurora PostgreSQL endpoints based on params.
 * @param {Object} params
 * @param {string[]} params.regions - AWS regions to query
 * @param {string} params.engine - Engine type (e.g. "aurora-postgresql")
 * @param {string|null} params.clusterIdentifier - Optional cluster ID, if null scan all clusters
 * @returns {Promise<Object[]>}
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
```

---

### ✅ Self-Executing Call (Runs When File is Executed)

```js
// You can replace these values or pass them from CLI or ENV
const params = {
  regions: ['us-east-1', 'us-west-2'],
  engine: 'aurora-postgresql',
  clusterIdentifier: null // Or specify a string like 'your-cluster-id'
};

const main = async () => {
  const results = await getAuroraEndpoints(params);
  console.log('\n✅ Aurora PostgreSQL Endpoints:\n');
  console.table(results);
};

main(); // self-executing
```

---

### ✅ How to Run

1. Save as `auroraEndpoints.mjs`
2. Install AWS SDK v3:
   ```bash
   npm install @aws-sdk/client-rds
   ```
3. Run it:
   ```bash
   node auroraEndpoints.mjs
   ```

---

### 🧪 Example Output

```bash
✅ Aurora PostgreSQL Endpoints:

┌─────────┬──────────────────────┬──────────────┬────────────────────┬──────┬────────────────────┬────────────┐
│ (index) │      clusterId       │      id      │      endpoint       │ port │   instanceClass    │   region   │
├─────────┼──────────────────────┼──────────────┼────────────────────┼──────┼────────────────────┼────────────┤
│    0    │ 'aurora-cluster-1'   │ 'instance-1' │ 'writer.db.x.rds..' │ 5432 │ 'db.r6g.large'     │ 'us-east-1'│
│    1    │ 'aurora-cluster-1'   │ 'instance-2' │ 'reader.db.x.rds..' │ 5432 │ 'db.r6g.large'     │ 'us-west-2'│
└─────────┴──────────────────────┴──────────────┴────────────────────┴──────┴────────────────────┴────────────┘
```

---

Would you like to also return this grouped by `clusterId`, or save as a `.json` file for use with Sequelize config generation?
