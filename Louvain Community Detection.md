# **Neo4j Graph Analytics on Snowflake: Quick Start Guide**

## **Overview**
This guide provides a step-by-step approach to running **Louvain Community detection graph algorithm** using **Neo4j Graph Analytics on Snowflake**. You will learn how to set up Neo4j Graph Analytics, construct graphs and store results in Snowflake.
### **What is Neo4j?**

 Neo4j, the Graph Database & Analytics leader, helps organizations find hidden relationships and patterns across billions of data connections deeply, easily, and quickly. Customers leverage the structure of their connected data to reveal new ways of solving their most pressing business problems, with Neo4j’s full graph stack and a vibrant community of developers, data scientists, and architects across hundreds of Fortune 500 companies. Visit neo4j.com.

### **What is Neo4j Graph Analytics?**




### **Dataset Overview**
The dataset used in this guide represents peer-to-peer (P2P) financial transactions where users transfer money between each other. Users may have multiple identifiers, including credit cards, devices, and IP addresses, enhancing the complexity and richness of the data. This structure makes it ideal for identifying clusters, influencers, and fraudulent behaviors.
![alt text](image.png)

### **What is the Louvain Community Detection Algorithm?**
The Louvain algorithm is a popular method for detecting communities in large networks by optimizing modularity. Modularity measures the density of links inside communities compared to links between communities. The algorithm iteratively assigns nodes into communities to maximize modularity, effectively identifying clusters of users that frequently interact with each other. This makes Louvain particularly useful for applications such as customer segmentation, fraud detection, and social network analysis.

### **Prerequisites**
**Snowflake Account & Access**
1. Active Snowflake account with appropriate access to databases and schemas.

2. Neo4j Graph Analytics application installed from the Snowflake marketplace.
   ![alt text](image-3.png)

## **Step 1 :Create a new worksheet and select database 
![alt text](image-1.png)
![alt text](image-2.png)

## **Step 2: Prepare Data**
In this step, we create and aggregate the transaction data necessary for subsequent graph analytics.
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
## **Step 3 : Setup & Access Configuration
Before running graph algorithms make sure you configure roles, grant permissions, and initialize the Neo4j Graph Analytics application in Snowflake.

Consumer Role: For users analyzing graph data.
```
CREATE ROLE IF NOT EXISTS gds_role;
GRANT APPLICATION ROLE neo4j_graph_analytics.app_user TO ROLE gds_role;
```

Admin Role: For managing the  application.
```
CREATE ROLE IF NOT EXISTS gds_admin_role;
GRANT APPLICATION ROLE neo4j_graph_analytics.app_admin TO ROLE gds_admin_role;
```

Grant Access to Data:
The application needs read access to transaction data and write permissions for storing results.
```
GRANT USAGE ON DATABASE p2p_demo TO APPLICATION neo4j_graph_analytics;
GRANT USAGE ON SCHEMA p2p_demo.public TO APPLICATION neo4j_graph_analytics;
```
Grants SELECT permission for the Snowflake application role so it can read tables. Without this, the Neo4j GDS application cannot load data from your tables to create a graph.
```
GRANT SELECT ON ALL TABLES IN SCHEMA p2p_demo.public TO APPLICATION neo4j_graph_analytics;
```
Grants CREATE TABLE permission so GDS can store any analytics outputs (e.g., results of Louvain or PageRank) in new tables.
```
GRANT CREATE TABLE ON SCHEMA p2p_demo.public TO APPLICATION neo4j_graph_analytics;
```
Ensures that any new tables created in the future by the application are automatically accessible to the roles in question. This is optional but convenient for a frictionless workflow.
```
GRANT ALL PRIVILEGES ON FUTURE TABLES IN SCHEMA p2p_demo.public TO ROLE gds_role;
GRANT ALL PRIVILEGES ON FUTURE TABLES IN SCHEMA p2p_demo.public TO ROLE accountadmin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA p2p_demo.public TO ROLE accountadmin;
```
Creates a compute session for the engine on Snowflake using a specified compute model (CPU_X64_M). This step is required before running graph algorithms.
```
CALL neo4j_graph_analytics.gds.create_session('CPU_X64_M');
```
Drop any existing graph named transaction_graph so you can recreate it from scratch. Using { 'failIfMissing': false } to avoids errors if the graph does not exist yet.
```
SELECT neo4j_graph_analytics.gds.graph_drop('transaction_graph', { 'failIfMissing': false });
USE DATABASE p2p_demo;
```

## Step 4 : Load Data from Snowflake
Retrieve raw data from Snowflake tables.
User Table (Nodes): Represents individual users (nodes) identified uniquely by user IDs.
Transaction Table (Relationships): Contains transactions between users, forming relationships in the graph.

```
USE DATABASE p2p_demo;
USE SCHEMA public;
SELECT * FROM p2p_users;

SELECT * FROM p2p_agg_transactions;
```
## Step 5 : Construct a Graph Projection
Transform tabular data into a graph model.
Define the Graph:

Users → Nodes

Transactions → Edges (Relationships)


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
Runs the Louvain algorithm to detect communities. The {'mutateProperty': 'community_id'} parameter tells us to store each node’s community identifier as community_id.
What this does is that it group users into communities based on transaction activity.

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



































