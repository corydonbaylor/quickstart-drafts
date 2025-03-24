# **Community Detection in Neo4j Graph Analytics on Snowflake**

## **Overview**
This guide provides a step-by-step approach to running **community detection graph algorithms** using **Neo4j Graph Analytics for Snowflake**. In this quickstart, you will learn how to set up Neo4j Graph Analytics, construct graphs, run community detection algorithms like Louvain, and store results in Snowflake.
### **What is Neo4j Graph Analytics for Snowflake?** CHANGE 

Neo4j, the Graph Database & Analytics leader, helps organizations find hidden relationships and patterns across billions of data connections deeply, easily, and quickly. **Neo4j Graph Analytics for Snowflake** brings to the power of graph directly to Snowflake, allowing users to run 65+ ready-to-use algorithms on their snowflake data, all without leaving Snowflake! 


### **Dataset Overview**
The dataset used in this guide represents peer-to-peer (P2P) financial transactions where users transfer money between each other. Users may have multiple identifiers, including credit cards, devices, and IP addresses, enhancing the complexity and richness of the data. This structure makes it ideal for identifying clusters, influencers, and fraudulent behaviors.
![alt text](image.png)

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

![alt text](image-1.png)
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
Before running graph algorithms, we need to make sure to configure roles, grant permissions, and initialize the Neo4j Graph Analytics application in Snowflake. You can find more details about granting permissions here.

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
Next we will retrieve raw data from Snowflake tables. Let's take a look at the results from a couple of tables:

- User Table (Nodes): Represents individual users (nodes) identified uniquely by user IDs.
- Transaction Table (Relationships): Contains transactions between users, forming relationships in the graph.

```
USE DATABASE p2p_demo;
USE SCHEMA public;
SELECT * FROM p2p_users;

SELECT * FROM p2p_agg_transactions;
```
## Step 5 : Construct a Graph Projection
Next, with just some simple transformations, we can prepare our tabular data to work with a graph model. In our case:

- Users → Nodes

- Transactions → Edges (Relationships)


```
SELECT neo4j_graph_analytics.gds.graph_project('transaction_graph',{
  'nodeTables': {
    'p2p_demo.public.p2p_users': 'Node'
  },
  'relationshipTables': {
    'p2p_demo.public.p2p_agg_transactions': {
      'type': 'REL',
      'sourceTable': 'p2p_demo.public.p2p_users',
      'targetTable': 'p2p_demo.public.p2p_users',
      'orientation': 'NATURAL'
    }
  }
});
```
## Step 6:  Run Louvain Community Detection
The louvain algorithm is a popular method for detecting communities in large networks by optimizing modularity. Modularity measures the density of links inside communities compared to links between communities. This makes Louvain particularly useful for applications such as customer segmentation, fraud detection, and social network analysis.

Now that we know how it works, let's run it with Neo4j Aura Graph Analytics directly in snowflake!

```
SELECT neo4j_graph_analytics.gds.louvain('transaction_graph', {'mutateProperty': 'community_id'});

-- Write  to table
--SELECT neo4j_graph_analytics.gds.write_nodeproperties('transaction_graph',
--           {'nodeProperties': ['community_id'], 'table': 'p2p_demo.public.p2p_louvain'});
SELECT neo4j_graph_analytics.gds.write_nodeproperties_to_table('transaction_graph', {
  'nodeLabels': ['Node'],
  'nodeProperties': ['community_id'],
  'tableSuffix': '_louvain'
});
```

The `{'mutateProperty': 'community_id'}` parameter tells us to store each node’s community identifier as community_id. What this does is that it group users into communities based on transaction activity.

## Step 7: Store and Analyze Community Results

Persist Louvain results in Snowflake and analyze community structures.

This helps us retreive all rows from your newly written table (e.g., p2p_users_louvain) so you can see which community each user belongs to.
```
SELECT * FROM p2p_users_louvain ORDER BY community_id;
```
This step aggregates the user communities by counting how many nodes exist in each. This query helps you identify the largest or most active communities.
```
SELECT community_id, count(nodeid) AS community_size
FROM p2p_users_louvain
GROUP BY community_id
ORDER BY community_size DESC LIMIT 10;
```

## **Summary**
Quick Links: 



































