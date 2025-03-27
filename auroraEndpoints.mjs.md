Absolutely! Here's your **combined Aurora endpoint fetcher**, fully converted to **ES6 module syntax** with modern JavaScript best practices:

---

### ✅ ES6 Version of `auroraEndpoints.mjs`

> Use `.mjs` extension, or set `"type": "module"` in your `package.json` to use `import`/`export`.

```js
// auroraEndpoints.mjs
import AWS from 'aws-sdk';

// Step 1: Fetch cluster metadata
const fetchClusterInfo = async (clusterId, region) => {
  AWS.config.update({ region });
  const rds = new AWS.RDS();

  const { DBClusters } = await rds.describeDBClusters({ DBClusterIdentifier: clusterId }).promise();
  if (!DBClusters.length) throw new Error(`No cluster found with ID: ${clusterId}`);

  return DBClusters[0];
};

// Step 2: Fetch all DB instances
const fetchInstanceInfo = async (rds) => {
  const { DBInstances } = await rds.describeDBInstances().promise();
  return DBInstances;
};

// Step 3: Map roles (write/read/global)
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

// Step 4: Main method
export const getAllAuroraEndpoints = async (clusterId, region) => {
  const rds = new AWS.RDS({ region });

  const cluster = await fetchClusterInfo(clusterId, region);
  const instances = await fetchInstanceInfo(rds);
  const endpoints = mapInstanceRoles(instances, cluster, region);

  return endpoints;
};

// Run directly: `node auroraEndpoints.mjs`
const CLUSTER_ID = 'your-aurora-cluster-id'; // ✅ Replace this
const REGION = 'us-east-1'; // ✅ Replace this

if (process.argv[1] === import.meta.url) {
  try {
    console.log(`🔍 Fetching Aurora endpoints for cluster: ${CLUSTER_ID}`);
    const endpoints = await getAllAuroraEndpoints(CLUSTER_ID, REGION);
    console.log('✅ Aurora Endpoints:\n');
    console.table(endpoints);
  } catch (err) {
    console.error('❌ Error fetching Aurora endpoints:', err.message);
  }
}
```

---

### ✅ How to Use

#### ➤ Option 1: Run directly
```bash
node auroraEndpoints.mjs
```

#### ➤ Option 2: Import in another ES6 file
```js
// anotherScript.mjs
import { getAllAuroraEndpoints } from './auroraEndpoints.mjs';

const endpoints = await getAllAuroraEndpoints('your-cluster-id', 'us-east-1');
console.table(endpoints);
```

---

### 💡 Tip: Enable ES6 Modules in CommonJS Projects
If you're using `.js` files, add this to your `package.json`:

```json
"type": "module"
```

---

Let me know if you'd like this wired up with a Sequelize config builder and instance initializer in ES6 style too!
