
# Fraud Detection Program Documentation

This document provides a detailed explanation of each step in the fraud detection program. The program identifies clusters of clients that share Personally Identifiable Information (PII) and flags clients as potential second-party fraudsters based on their centrality in the similarity graph.

---

## Step 1: Check Neo4j Version

- **Purpose**: Verifies the Neo4j version and edition to ensure compatibility.
- **Process**:
  - Executes the Cypher query: 
    ```cypher
    CALL dbms.components() YIELD name, versions, edition
    RETURN name, versions, edition
    ```
  - Displays the kernel version, edition, and other component details.
- **Output**: Confirms that Neo4j is up and running.

---

## Step 2: Create SHARED_PII Relationships

- Purpose: Identifies relationships where clients share the same email, phone number, or SSN.
- Process:
  - Uses the following Cypher query:
    ```cypher
    MATCH (c1:Client)-[:HAS_EMAIL|HAS_PHONE|HAS_SSN]->(p)<-[:HAS_EMAIL|HAS_PHONE|HAS_SSN]-(c2:Client)
    WHERE c1.id < c2.id
    MERGE (c1)-[:SHARED_PII]->(c2)
    ```
  - Creates `SHARED_PII` relationships between clients who share any PII.
- Output: Confirms the relationships were successfully created.

---

## Step 3: Verify Schema Visualization

- Purpose: Visualizes the schema for confirmation of the data model.
- Process:
  - Executes:
    ```cypher
    CALL db.schema.visualization() YIELD nodes, relationships
    RETURN nodes, relationships
    ```
  - Verifies that all relationships, nodes, and constraints are in place.
- Output: Provides a schema visualization.

---

## Step 4: Create In-Memory Projection `clientClusters`

- Purpose: Projects the graph into memory for processing with the Neo4j Graph Data Science (GDS) library.
- Process:
  - Drops any existing graph with the same name to avoid conflicts.
  - Creates the projection:
    ```cypher
    CALL gds.graph.project(
        'clientClusters',
        'Client',
        'SHARED_PII'
    )
    YIELD graphName, nodeCount, relationshipCount
    RETURN graphName, nodeCount, relationshipCount
    ```
- Output: Confirms that the in-memory projection `clientClusters` was successfully created with the correct number of nodes and relationships.

---

## Step 5: Identify Clusters Using Weakly Connected Components (WCC)

- Purpose: Groups clients into clusters based on their connections in the `clientClusters` graph.
- Process:
  - Executes the Weakly Connected Components (WCC) algorithm:
    ```cypher
    CALL gds.wcc.stream('clientClusters')
    YIELD nodeId, componentId
    RETURN gds.util.asNode(nodeId).id AS clientId,
           componentId AS clusterId
    ```
  - Each `componentId` represents a unique cluster.
- Output: Displays the clusters and the clients in each.

---

## Step 6: Mark Clients in Clusters as Potential Fraud Rings

- Purpose: Flags clusters containing at least two clients as potential fraud rings.
- Process:
  - Identifies clusters with more than one client and marks each client in such clusters with the property `SecondPartyFraudster`.
  - Cypher query:
    ```cypher
    CALL gds.wcc.stream('clientClusters')
    YIELD nodeId, componentId
    WITH gds.util.asNode(nodeId) AS client, componentId AS clusterId
    WITH clusterId, collect(client.id) AS clients
    WITH clusterId, clients, size(clients) AS clusterSize
    WHERE clusterSize >= 2
    UNWIND clients AS clientId
    MATCH (c:Client) WHERE c.id = clientId
    SET c.SecondPartyFraudster = true
    ```
- Output: Confirms that clients in large clusters were marked.

---

## Step 7: Create Bipartite Graph Using Cypher Projection

- Purpose: Creates a bipartite graph for similarity analysis.
- Process:
  - Projects clients and their PII (emails, phones, SSNs) into memory:
    ```cypher
    CALL gds.graph.project(
        'similarity',
        ['Client', 'Email', 'Phone', 'SSN'],
        ['HAS_EMAIL', 'HAS_PHONE', 'HAS_SSN']
    )
    YIELD graphName, nodeCount, relationshipCount
    RETURN graphName, nodeCount, relationshipCount
    ```
- Output: Confirms the creation of the bipartite `similarity` graph.

---

## Step 8: Compute Similarity Scores Using `nodeSimilarity` Algorithm

- Purpose: Calculates similarity between clients based on shared PII.
- Process:
  - Runs the Node Similarity algorithm:
    ```cypher
    CALL gds.nodeSimilarity.mutate(
        'similarity',
        {
            mutateProperty: 'jaccardScore',
            mutateRelationshipType: 'SIMILAR_TO',
            topK: 15
        }
    )
    YIELD nodesCompared, relationshipsWritten, similarityDistribution
    RETURN nodesCompared, relationshipsWritten, similarityDistribution
    ```
- Output: Confirms the relationships and scores for similar nodes.

---

## Step 9: Write `SIMILAR_TO` Relationships and Compute Degree Centrality

- Purpose: Assigns centrality scores based on the similarity graph.
- Process:
  - Runs the Degree Centrality algorithm:
    ```cypher
    CALL gds.degree.write(
        'similarity',
        {
            nodeLabels: ['Client'],
            relationshipTypes: ['SIMILAR_TO'],
            relationshipWeightProperty: 'jaccardScore',
            writeProperty: 'secondPartyFraudScore'
        }
    )
    YIELD nodePropertiesWritten
    RETURN nodePropertiesWritten
    ```
- Output: Confirms that centrality scores (`secondPartyFraudScore`) were computed and written.

---

## Step 10: Label High Centrality Clients as Potential Fraudsters

- **Purpose**: Identifies clients with high centrality scores as potential fraudsters using statistical thresholds.
- **Process**:
  - Mathematical Basis:
    1. Quartiles and IQR:
       \[
       IQR = Q3 - Q1
       \]
       \[
       	ext{Upper Threshold} = Q3 + 1.5 	imes IQR
       \]
       where \(Q1\) and \(Q3\) are the 25th and 75th percentiles of centrality scores.
    2. **Percentile Threshold**:
       \[
       	ext{Threshold} = 	ext{percentileCont}(	ext{score}, 95)
       \]
  - Implementation:
    - First, calculate IQR and thresholds:
      ```cypher
      MATCH (c:Client)
      WHERE c.secondPartyFraudScore IS NOT NULL
      WITH 
          percentileCont(c.secondPartyFraudScore, 0.25) AS Q1,
          percentileCont(c.secondPartyFraudScore, 0.75) AS Q3
      RETURN Q1, Q3, (Q3 - Q1) AS IQR, (Q3 + 1.5 * (Q3 - Q1)) AS upperThreshold
      ```
    - Label clients exceeding the thresholds:
      ```cypher
      WITH $threshold AS threshold
      MATCH (c:Client)
      WHERE c.secondPartyFraudScore > threshold
      SET c.SecondPartyFraudster = true
      RETURN count(c) AS fraudsterCount
      ```
- Output: Lists potential fraudsters and the applied threshold.

---

## Step 11: List Potential Fraudsters**

- **Purpose**: Retrieves all flagged fraudsters.
- **Process**:
  - Executes:
    ```cypher
    MATCH (c:Client)
    WHERE c.SecondPartyFraudster = true
    RETURN c.id AS ClientID, c.name AS Name
    ```
- **Output**: Displays the IDs and names of potential fraudsters.

---

## **Step 12: Cleanup - Close Neo4j Driver**

- **Purpose**: Ensures the Neo4j connection is closed after execution.
- **Process**:
  - Calls `driver.close()` in Python to clean up resources.
- **Output**: Confirms the driver was successfully closed.

---

This documentation provides a comprehensive understanding of the fraud detection program, ensuring that each step is clear and reproducible.
