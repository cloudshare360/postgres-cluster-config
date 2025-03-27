Absolutely! Here's the **combined code** with everything in one file for simplicity — including:

- Fetching Aurora cluster metadata  
- Determining instance roles (reader, writer, global)  
- Returning a detailed list of endpoints  
- Executing the method with `console.table` output

---

### ✅ Combined File: `auroraEndpoints.js`

```js
const AWS = require('aws-sdk');

// Step 1: Fetch cluster metadata (writer endpoint + cluster members)
const fetchClusterInfo = async (clusterId, region) => {
  AWS.config.update({ region });
  const rds = new AWS.RDS();

  const { DBClusters } = await rds.describeDBClusters({ DBClusterIdentifier: clusterId }).promise();
  if (!DBClusters.length) throw new Error(`No cluster found with ID: ${clusterId}`);

  return DBClusters[0];
};

// Step 2: Fetch all DB instances (in region)
const fetchInstanceInfo = async (rds) => {
  const { DBInstances } = await rds.describeDBInstances().promise();
  return DBInstances;
};

// Step 3: Determine instance roles (write/read/global)
const mapInstanceRoles = (instances, cluster, region) => {
  const clusterMemberIds = cluster.DBClusterMembers.map(member => member.DBInstanceIdentifier);

  return instances
    .filter(instance => clusterMemberIds.includes(instance.DBInstanceIdentifier))
    .map(instance => {
      const memberMeta = cluster.DBClusterMembers.find(
        m => m.DBInstanceIdentifier === instance.DBInstanceIdentifier
      );

      const isWriter = memberMeta?.IsClusterWriter === true;
      let role = 'read';

      if (isWriter) role = 'write';
      else if (instance.Engine?.includes('global')) role = 'global';

      return {
        id: instance.DBInstanceIdentifier,
        endpoint: instance.Endpoint.Address,
        port: instance.Endpoint.Port,
        instanceClass: instance.DBInstanceClass,
        region,
        role,
      };
    });
};

// Step 4: Main function to get all Aurora endpoints
const getAllAuroraEndpoints = async (clusterId, region) => {
  const rds = new AWS.RDS({ region });

  const cluster = await fetchClusterInfo(clusterId, region);
  const instances = await fetchInstanceInfo(rds);
  const endpoints = mapInstanceRoles(instances, cluster, region);

  return endpoints;
};

// Run the method (if this file is executed directly)
const CLUSTER_ID = 'your-aurora-cluster-id'; // ✅ Replace with your actual Aurora cluster ID
const REGION = 'us-east-1'; // ✅ Replace with your AWS region

if (require.main === module) {
  (async () => {
    try {
      console.log(`🔍 Fetching Aurora endpoints for cluster: ${CLUSTER_ID}`);
      const endpoints = await getAllAuroraEndpoints(CLUSTER_ID, REGION);
      console.log('✅ Aurora Endpoints:\n');
      console.table(endpoints);
    } catch (err) {
      console.error('❌ Error fetching Aurora endpoints:', err.message);
    }
  })();
}

// Export if used as a module
module.exports = { getAllAuroraEndpoints };
```

---

### 🧪 How to Run

1. Save as `auroraEndpoints.js`
2. Replace `your-aurora-cluster-id` and `us-east-1` with real values
3. Run it:

```bash
node auroraEndpoints.js
```

---

### 📋 Output Example

```bash
✅ Aurora Endpoints:

┌─────────┬────────────────────────────┬──────┬────────────────────┬────────────┬────────────┐
│ (index) │         endpoint           │ port │   instanceClass    │    role    │   region   │
├─────────┼────────────────────────────┼──────┼────────────────────┼────────────┼────────────┤
│    0    │ 'writer.cluster.example...'│ 5432 │ 'db.r6g.large'     │ 'write'    │ 'us-east-1'│
│    1    │ 'reader1.cluster.example...'│ 5432 │ 'db.r6g.large'    │ 'read'     │ 'us-east-1'│
└─────────┴────────────────────────────┴──────┴────────────────────┴────────────┴────────────┘
```

---

Let me know if you want this to:
- Include **status** (`available`, `modifying`, etc.)
- Handle **multiple regions**
- Integrate with Sequelize right after

Happy to chain it all together!
