If you don't want to use `writeInstance.clusterId` for initializing the Sequelize instance, you can directly pass the `writeInstance.endpoint` for the connection as the `host` instead of relying on `clusterId`. Below is the updated code:

---

### Updated `dbInit` Method Without Using `writeInstance.clusterId`

```javascript
import { Sequelize } from 'sequelize';

export default async function dbInit(instanceArray) {
  try {
    console.log("Input instance array:", instanceArray);

    // Extract the write instance and read instances
    const writeInstance = instanceArray.find(instance => instance.role === 'write');
    const readInstances = instanceArray.filter(instance => instance.role === 'read');

    if (!writeInstance) {
      throw new Error('No write instance found in the provided instance array.');
    }

    // Create Sequelize cluster configuration
    const sequelizeConfig = {
      dialect: 'postgres',
      pool: {
        max: 10, // Adjust based on expected concurrent connections
        min: 0,
        idle: 10000, // Idle connection timeout in milliseconds
        acquire: 30000, // Maximum time (ms) to acquire a connection
        evict: 15000, // Remove stale connections
      },
      replication: {
        write: {
          host: writeInstance.endpoint, // Use endpoint of the write instance
          port: writeInstance.port,
          username: process.env.DB_USER || 'defaultUser', // Replace or fetch dynamically
          password: process.env.DB_PASSWORD || 'defaultPassword', // Replace or fetch dynamically
        },
        read: readInstances.map(reader => ({
          host: reader.endpoint,
          port: reader.port,
        })),
      },
      retry: {
        match: [
          /SequelizeConnectionError/,
          /SequelizeConnectionRefusedError/,
          /SequelizeHostNotFoundError/,
          /SequelizeHostNotReachableError/,
          /SequelizeInvalidConnectionError/,
          /SequelizeConnectionTimedOutError/,
        ],
        max: 5, // Maximum retry attempts
      },
      logging: console.log, // Enable logging for debugging
      dialectOptions: {
        ssl: {
          require: true,
          rejectUnauthorized: false, // Adjust as needed for your setup
        },
      },
    };

    console.log("Generated Sequelize config:", sequelizeConfig);

    // Initialize Sequelize instance
    const sequelize = new Sequelize({
      dialect: 'postgres',
      replication: sequelizeConfig.replication,
      pool: sequelizeConfig.pool,
      retry: sequelizeConfig.retry,
      logging: sequelizeConfig.logging,
      dialectOptions: sequelizeConfig.dialectOptions,
    });

    // Test connection
    await sequelize.authenticate();
    console.log('Database connection established successfully with Sequelize cluster configuration.');

    return sequelize;
  } catch (error) {
    console.error('Error initializing Sequelize cluster configuration:', error);
    throw error; // Re-throw the error for higher-level handling
  }
}
```

---

### Key Changes:
1. **Removed `writeInstance.clusterId`**:
   - Replaced `writeInstance.clusterId` with direct use of `writeInstance.endpoint`, which serves as the `host`.

2. **Updated `Sequelize` Initialization**:
   - Used `Sequelize` constructor directly with `replication`, `pool`, `retry`, and other configuration options.

3. **Environment Variables**:
   - Ensure you have `DB_USER` and `DB_PASSWORD` set in your environment for dynamic credential management.

4. **Direct Endpoint Usage**:
   - The `endpoint` of the write instance is used instead of `clusterId`.

---

### Input Example:
Provide the same input array as earlier:
```json
[
  {
    "clusterId": "aurpgs-fomfocucfl-dev-ffc-e-cluster",
    "id": "aurpgs-fomfocucfl-dev-e-ffc-01",
    "endpoint": "aurpgs-fomfocucfl-dev-e-ffc-01.ce70oon0iwlq.us-east-1.rds.amazonaws.com",
    "port": 5432,
    "instanceClass": "db.r6g.large",
    "region": "us-east-1",
    "role": "write"
  },
  {
    "clusterId": "aurpgs-fomfocucfl-dev-ffc-e-cluster",
    "id": "aurpgs-fomfocucfl-dev-e-ffc-03",
    "endpoint": "aurpgs-fomfocucfl-dev-e-ffc-03.ce70oon0iwlq.us-east-1.rds.amazonaws.com",
    "port": 5432,
    "instanceClass": "db.r6g.large",
    "region": "us-east-1",
    "role": "read"
  },
  {
    "clusterId": "aurpgs-fomfocucfl-dev-ffc-w-cluster",
    "id": "aurpgs-fomfocucfl-dev-w-ffc-01",
    "endpoint": "aurpgs-fomfocucfl-dev-w-ffc-01.cfsbdknpscok.us-west-2.rds.amazonaws.com",
    "port": 5432,
    "instanceClass": "db.r6g.large",
    "region": "us-west-2",
    "role": "read"
  }
]
```

---

### Expected Output:
When you invoke `dbInit(instanceArray)`, it will:
1. Dynamically create a Sequelize configuration using `writeInstance.endpoint` for the primary host.
2. Include all read instances under `replication.read`.
3. Test the connection successfully, logging a success message.

Let me know if this fits your needs or requires any further changes! 🚀
