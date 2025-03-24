## **Overview**

This guide provides a step-by-step approach to **entity resolution** using **Neo4j Graph Analytics for Snowflake**. In this quickstart, you will learn how to set up Neo4j Graph Analytics, construct graphs, run community detection algorithms like WCC, and store results in Snowflake.

### **What is Neo4j Graph Analytics for Snowflake?**  

Neo4j, the Graph Database & Analytics leader, helps organizations find hidden relationships and patterns across billions of data connections deeply, easily, and quickly. **Neo4j Graph Analytics for Snowflake** brings to the power of graph directly to Snowflake, allowing users to run 65+ ready-to-use algorithms on their snowflake data, all without leaving Snowflake! 


### **Dataset Overview**

The dataset used in this guide represents peer-to-peer (P2P) financial transactions where users transfer money between each other. Users may have multiple identifiers, including credit cards, devices, and IP addresses, enhancing the complexity and richness of the data. This structure makes it ideal for identifying clusters, influencers, and fraudulent behaviors.

### What you will learn

- How to prepare and project your data for graph analytics
- How to use community detection to identify fraud
- How to read and write directly from and to your snowflake tables

### **Prerequisites**

**Snowflake Account & Access**

1. Active Snowflake account with appropriate access to databases and schemas.

2. Neo4j Graph Analytics application installed from the Snowflake marketplace.
   ![alt text](image-3.png)

## Step 1 :Create a new worksheet and select database 

First, we need to select a worksheet and a database to work with.

![alt text](/Users/corydonbaylor/Documents/github/quickstart-drafts/image-1.png)
![alt text](image-2.png)

## **Step 2: Prepare Data**

Now that we have our worksheet and have selected our database, we need to create and aggregate the transaction data necessary for subsequent graph analytics.

```
CREATE OR REPLACE TABLE p2p_demo.public.P2P_AGG_TRANSACTIONS (
	SOURCENODEID NUMBER(38,0),
	TARGETNODEID NUMBER(38,0),
	TOTAL_AMOUNT FLOAT,
	TRANSACTION_COUNT FLOAT
) AS
SELECT sourceNodeId, targetNodeId, SUM(transaction_amount) AS total_amount, COUNT(*) AS transaction_count
FROM p2p_demo.public.P2P_TRANSACTIONS
GROUP BY sourceNodeId, targetNodeId;
SELECT * FROM p2p_demo.public.P2P_AGG_TRANSACTIONS;
*/

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

## Step 3 : Setup & Access Configuration

Before running graph algorithms, we need to make sure to configure roles, grant permissions, and initialize the Neo4j Graph Analytics application in Snowflake. You can find more details about granting permissions [here](https://app.snowflake.com/hiysshm/neo4j_emea_field/#/apps/application/DF_SNOW_NEO4J_GRAPH_ANALYTICS/security/readme?isPrivate=true).

<table style="width:100%; border-collapse: collapse;">
  <tr>
    <th style="width:75%; text-align: left; padding: 8px; border: 1px solid #ddd;">Permissions</th>
    <th style="width:25%; text-align: left; padding: 8px; border: 1px solid #ddd;">Explanation</th>
  </tr>
  <tr>
    <td style="padding: 8px; border: 1px solid #ddd;">
      <pre><code>CREATE ROLE IF NOT EXISTS gds_role;
GRANT APPLICATION ROLE neo4j_graph_analytics.app_user TO ROLE gds_role;</code></pre>
    </td>
    <td style="padding: 8px; border: 1px solid #ddd;">Creates a role for users to analyze data</td>
  </tr>
  <tr>
    <td style="padding: 8px; border: 1px solid #ddd;">
      <pre><code>CREATE ROLE IF NOT EXISTS gds_admin_role;
GRANT APPLICATION ROLE neo4j_graph_analytics.app_user TO ROLE gds_role;</code></pre>
    </td>
    <td style="padding: 8px; border: 1px solid #ddd;">Creates a role for managing the application</td>
  </tr>
  <tr>
    <td style="padding: 8px; border: 1px solid #ddd;">
      <pre><code>GRANT USAGE ON DATABASE p2p_demo TO APPLICATION neo4j_graph_analytics;
GRANT USAGE ON SCHEMA p2p_demo.public TO APPLICATION neo4j_graph_analytics;</code></pre>
    </td>
    <td style="padding: 8px; border: 1px solid #ddd;">The application needs read access to transaction data and write permissions for storing results.</td>
  </tr>
  <tr>
    <td style="padding: 8px; border: 1px solid #ddd;">
      <pre><code>GRANT SELECT ON ALL TABLES IN SCHEMA p2p_demo.public TO APPLICATION neo4j_graph_analytics;</code></pre>
    </td>
    <td style="padding: 8px; border: 1px solid #ddd;">Grants `SELECT` permission for the Snowflake application role so it can read tables. Without this, the Neo4j GDS application cannot load data from your tables to create a graph.</td>
  </tr>
  <tr>
    <td style="padding: 8px; border: 1px solid #ddd;">
      <pre><code>GRANT CREATE TABLE ON SCHEMA p2p_demo.public TO APPLICATION neo4j_graph_analytics;</code></pre>
    </td>
    <td style="padding: 8px; border: 1px solid #ddd;">Grants `CREATE TABLE` permission so GDS can store any analytics outputs (e.g., results of Louvain) in new tables. </td>
  </tr>
  <tr>
    <td style="padding: 8px; border: 1px solid #ddd;">
      <pre><code>GRANT ALL PRIVILEGES ON FUTURE TABLES IN SCHEMA p2p_demo.public TO ROLE gds_role;
GRANT ALL PRIVILEGES ON FUTURE TABLES IN SCHEMA p2p_demo.public TO ROLE accountadmin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA p2p_demo.public TO ROLE accountadmin;</code></pre>
    </td>
    <td style="padding: 8px; border: 1px solid #ddd;">Ensures that any new tables created in the future by the application are automatically accessible to the roles in question. This is optional but convenient for a frictionless workflow.</td>
  </tr>
  <tr>
    <td style="padding: 8px; border: 1px solid #ddd;">
      <pre><code>CALL neo4j_graph_analytics.gds.create_session('CPU_X64_M');</code></pre>
    </td>
    <td style="padding: 8px; border: 1px solid #ddd;">Creates a compute session for the engine on Snowflake using a specified compute model (CPU_X64_M). This step is required before running graph algorithms.</td>
  </tr>
  <tr>
    <td style="padding: 8px; border: 1px solid #ddd;">
      <pre><code>SELECT neo4j_graph_analytics.gds.graph_drop('transaction_graph', { 'failIfMissing': false });
USE DATABASE p2p_demo;</code></pre>
    </td>
    <td style="padding: 8px; border: 1px solid #ddd;">Drop any existing graph named transaction_graph so you can recreate it from scratch. Using { 'failIfMissing': false } to avoids errors if the graph does not exist yet.</td>
  </tr>
</table>


## Step 4 : Load Data from Snowflake

- <<MYSTERY DATA SET GOTTA ASK ZACH HOW IT WAS MADE!!!

```
USE DATABASE p2p_demo;
USE SCHEMA public;
SELECT * FROM p2p_trans_w_shared_card;
SELECT * FROM p2p_users;
```

Note the `HAS_FRAUD_FLAG` in the `p2p_users` table. 

## Step 5 : Create a Projection

Next we are going to create a projection using our transacitons with a shared card to establish relationships between our nodes.

```
-- Construct a graph projection from node and relationship tables
SELECT neo4j_graph_analytics.gds.graph_drop('entity_linking_graph', { 'failIfMissing': false });

-- Similar to calling the functions with simple or qualified names, we have to reference the tables wither with qualified names or with simple names while USE ing the database or schema.
SELECT neo4j_graph_analytics.gds.graph_project('entity_linking_graph', {
  'nodeTables': {
    'p2p_demo.public.p2p_users': 'Node'
  },
  'relationshipTables': {
    'p2p_demo.public.p2p_trans_w_shared_card': {
      'type': 'REL',
      'sourceTable': 'p2p_demo.public.p2p_users',
      'targetTable': 'p2p_demo.public.p2p_users',
      'orientation': 'NATURAL'
    }
  }
});

```

## Step 6: Running Weakly Connected Components

Before we actually run Weakly Connected Components (WCC), we should pause here and briefly describe how it works. WCC groups nodes together based on whether they are connected by relationships, ignoring relationship direction. Remember back to how we had some users manually flagged for potential fraud?

WCC will allow us to see any users who either directly or indirectly interacted with a flagged fraudster. Now that we know what we are doing, let's run WCC:

```
-- calculate wcc
SELECT neo4j_graph_analytics.gds.wcc('entity_linking_graph', {'mutateProperty': 'wcc_id'});
```

## Step 7: Writing Back to Snowflake

Next, we are going to write back to our snowflake tables. Notice how we specify `_wcc` as a table suffix. This creates a new table called `P2P_USERS_WCC`. 

```
-- Write  to table
SELECT DF_SNOW_NEO4J_GRAPH_ANALYTICS.gds.write_nodeproperties('entity_linking_graph', {
    'nodeLabels': ['Node'],
    'nodeProperties': ['wcc_id'], 
    'tableSuffix': '_wcc'}
);
```

We can look at the results like so:

```
SELECT * FROM P2P_USERS_WCC ORDER BY wcc_id;
```

## Step 8: Flagging Increased Fraud Risk

Finally, because we wrote back to snowflake, we can create different views of the data pretty easily. Let's look at the different communities created by WCC like so:

```
-- create a resolved entity view based on WCC
CREATE OR REPLACE VIEW resolved_p2p_users AS
SELECT p2p_users_wcc.wcc_id,
       count(*) AS user_count,
       TO_NUMBER(SUM(p2p_users.fraud_transfer_flag)>0) AS has_fraud_flag,
       ARRAY_AGG(p2p_users.nodeId) AS user_ids
FROM p2p_users JOIN p2p_users_wcc ON p2p_users.nodeId = p2p_users_wcc.nodeId
GROUP BY wcc_id ORDER BY user_count DESC;
```

~~ SHOW THE TABLE RESULTS HERE~~

If there is any users with a flag for fraud within a community, that means that this entire community may be at heightened risk either for experiencing fraud or for being duplicated accounts of the fraudsters themselves.