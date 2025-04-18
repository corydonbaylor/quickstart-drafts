# Discovering Communities in P2P Fraud
P2P Fraud Losses are Skyrocketing. 8% of banking customers reported being victims of P2P Scams in the past year, and the average loss to these scams was $176.

Finding different communities within P2P transactions is the first step towards identifying and ultimately ending P2P fraud. 

**In This Demo You Will Learn**

- How to prepare and project your data for graph analytics
- How to use community detection to identify fraud
- How to read and write directly from and to your snowflake tables

**You Will Need**
- Active Snowflake account with appropriate access to databases and schemas.
- Neo4j Graph Analytics application installed from the Snowflake marketplace. Access the marketplace via the menu bar on the left hand side of your screen, as seen below:

## Loading the Data
Let's name our database `P2P_DEMO`. Using the CSVs found here, We are going to add two new tables:

- One called `P2P_TRANSACTIONS` based on the p2p_transactions.csv
- One called `P2P_USERS based` on p2p_users.csv

Follow the steps found [here](https://docs.snowflake.com/en/user-guide/data-load-web-ui) to load in your data.

## Setting Up
Before we run our algorithms, we need to set the proper permissions. But before we get started granting different roles, we need to ensure that you are using `accountadmin` to grant and create roles. Lets do that now:

```sql
-- you must be accountadmin to create role and grant permissions
use role accountadmin;
```

Next let's set up the necessary roles, permissions, and resource access to enable Graph Analytics to operate on data within the `p2p_demo.public schema`. It creates a consumer role (gds_role) for users and administrators, grants the GDS application access to read from and write to tables and views, and ensures that future tables are accessible. 

It also provides the application with access to the required compute pool and warehouse resources needed to run graph algorithms at scale.

```sql
USE SCHEMA p2p_demo.public;

-- Create a consumer role for users and admins of the GDS application
CREATE ROLE IF NOT EXISTS gds_role;
GRANT APPLICATION ROLE se_snow_neo4j_graph_analytics.app_user TO ROLE gds_role;
GRANT APPLICATION ROLE se_snow_neo4j_graph_analytics.app_admin TO ROLE gds_role;

-- Grant access to consumer data
GRANT USAGE ON DATABASE p2p_demo TO APPLICATION se_snow_neo4j_graph_analytics;
GRANT USAGE ON SCHEMA p2p_demo.public TO APPLICATION se_snow_neo4j_graph_analytics;

-- Required to read tabular data into a graph
GRANT SELECT ON ALL TABLES IN SCHEMA p2p_demo.public TO APPLICATION se_snow_neo4j_graph_analytics;

-- Required to write computation results into a table/view
GRANT CREATE TABLE ON SCHEMA p2p_demo.public TO APPLICATION se_snow_neo4j_graph_analytics;
GRANT CREATE VIEW ON SCHEMA p2p_demo.public TO APPLICATION se_snow_neo4j_graph_analytics;

-- Ensure the consumer role has access to created tables/views
GRANT ALL PRIVILEGES ON FUTURE TABLES IN SCHEMA p2p_demo.public TO ROLE gds_role;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA p2p_demo.public TO ROLE gds_role;
GRANT CREATE TABLE ON SCHEMA p2p_demo.public TO ROLE gds_role;
GRANT CREATE VIEW ON SCHEMA p2p_demo.public TO ROLE gds_role;

-- Compute and warehouse access
GRANT USAGE, MONITOR ON COMPUTE POOL NEO4J_GRAPH_DATA_SCIENCE_POOL_CPU_X64_XS TO APPLICATION se_snow_neo4j_graph_analytics;
GRANT USAGE ON WAREHOUSE GDSONSNOWFLAKE TO APPLICATION se_snow_neo4j_graph_analytics;
```

Now we will switch to the role we just created:

```sql
use role gds_role;
```



## Cleaning Our Data

In order to run Graph Analytics, we will need to create a new notebook and then select `P2P_DEMO` as the database:

We need our data to be in a particular format in order to work with Graph Analytics. In general it should be like so:

**For the table representing nodes:**

The first column should be called `nodeId`, which represents the ids for the each node in our graph

**For the table representing relationships:**

We need to have columns called `sourceNodeId` and `targetNodeId`. These will tell Graph Analytics the direction of the transaction, which in this case means:
- Who sent the money (sourceNodeId) and
- Who received it (targetNodeId)
- We also include a total_amount column that acts as the weights in the relationship

We are going to use aggregated transactions for our relationships. Let's create that table now:

Step 3: Prepare Data**

Now that we have our worksheet and have selected our database, we can easily start manipulating our data using SQL for our analysis. 

We are first going to create a new table with the aggregated transactions between users like so:

```sql
CREATE OR REPLACE TABLE p2p_demo.public.P2P_AGG_TRANSACTIONS (
	SOURCENODEID NUMBER(38,0),
	TARGETNODEID NUMBER(38,0),
	TOTAL_AMOUNT FLOAT
) AS
SELECT sourceNodeId, targetNodeId, SUM(transaction_amount) AS total_amount
FROM p2p_demo.public.P2P_TRANSACTIONS
GROUP BY sourceNodeId, targetNodeId;
SELECT * FROM p2p_demo.public.P2P_AGG_TRANSACTIONS;
```
We are also going to create a view that just has the unique `nodeId`s from the `p2p_demo` table and use that as the nodes when we project the graph in the next step:

```sql
CREATE OR REPLACE VIEW p2p_users_vw (nodeId) AS
    SELECT DISTINCT p2p_demo.public.p2p_users.NODEID as nodeid
    FROM p2p_users;
```

## Running your Algorithms
Now we are finally at the step where we create a projection, run our algorithms, and write back to snowflake.

To start we will create a session:

```sql
CALL se_snow_neo4j_graph_analytics.gds.create_session('CPU_X64_L');
```

If you find that you run into permissions issues, directly grant access to each table used in the projection to the snowpack application for the next step:

```sql
GRANT SELECT ON TABLE P2P_DEMO.PUBLIC.P2P_USERS_VW TO APPLICATION se_snow_neo4j_graph_analytics;

GRANT SELECT ON TABLE P2P_DEMO.PUBLIC.P2P_AGG_TRANSACTIONS TO APPLICATION se_snow_neo4j_graph_analytics;
```

Next we are going to make a **graph projection**. This takes the nodes and relationships defined in the tables we just created and reprsents as an in-memory against which we can run our algorithms.

```sql
SELECT se_snow_neo4j_graph_analytics.gds.graph_project('g', {
    'defaultTablePrefix': 'p2p_demo.public',
    'nodeTables' : ['p2p_users_vw'],
    'relationshipTables': {
        'P2P_AGG_TRANSACTIONS': {
        'sourceTable': 'p2p_users_vw',
        'targetTable': 'p2p_users_vw'
        }
    }
});
```

Next we run **louvain** to determine communities within our data. Louvain identifies communities by grouping together nodes that have more connections to each other than to nodes outside the group.

```sql
SELECT se_snow_neo4j_graph_analytics.gds.louvain('g', {'mutateProperty': 'community_id'});
```

And finally, after granting some more permissions, we write our node labels back to a snowflake data:

```sql
GRANT ALL PRIVILEGES ON TABLE P2P_DEMO.PUBLIC.P2P_USERS_VW_LOUVAIN TO APPLICATION se_snow_neo4j_graph_analytics;

SELECT se_snow_neo4j_graph_analytics.gds.write_nodeproperties_to_table('g', {
  'nodeLabels': ['p2p_users_vw'],
  'nodeProperties': ['community_id'],
  'tableSuffix': '_louvain'
});
```

Finally, we can take a look at our results:

```sql
SELECT p.NODEID, p.FRAUD_TRANSFER_FLAG, lv.COMMUNITY_ID
    FROM p2p_demo.public.P2P_USERS_VW_LOUVAIN AS lv JOIN p2p_demo.public.p2p_users AS p
        ON lv.NODEID = p.NODEID
    ORDER BY lv.COMMUNITY_ID, p.NODEID;
```





















