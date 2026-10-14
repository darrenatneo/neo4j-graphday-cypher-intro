# Aura Graph Analytics (GDS)

This file gives some advice on how to use Aura Graph Data Science with the dataset.


We will be projecting a graph into the GDS [Graph Catalog](https://neo4j.com/docs/graph-data-science/current/management-ops/graph-catalog-ops/) using [Native Projection](https://neo4j.com/docs/graph-data-science/current/management-ops/projections/graph-project/) 

Aura Graph Analytics uses serverless architecture to host ephemeral sessions and dynamically created graph projections. Algorithms can be executed on these projections and sessions only conume Aura credits while in use.

We can explicitly create a new Aura GDS Session in the following way :

```cypher
CALL gds.session.getOrCreate(
  'test-session',
  '2GB',
  duration({minutes: 30})
)
YIELD name
RETURN name;
```

If you want to ensure you have no existing projections you can run the following Cypher to clear your Graph Catalog:

```cypher
CALL gds.graph.list() YIELD graphName AS toDrop
CALL gds.graph.drop(toDrop) YIELD graphName
RETURN "Dropped " + graphName;
```

We will project a graph named 'OperationalPoints' into the Graph Catalog. We will take the `OperationalPoint` Nodes and the `SECTION` Relationships to form a monopartite graph:

```cypher
CALL gds.graph.project(
    'OperationalPoints',
    'OperationalPoint',
    'SECTION',
  {
    relationshipProperties: ['sectionlength', 'traveltime'],
    undirectedRelationshipTypes: ['SECTION'],
    sessionId: 'd7ef8b94-e44e3e75'
  }
) YIELD nodeCount, relationshipCount, projectMillis
RETURN nodeCount;
```

### Path Finding

We can calculate the shortest path between two stations - for example, Malmö Central to Stockholm Central - using our `traveltime` relatonship weights and the [Dijkstra Source-Target Shortest Path](https://neo4j.com/docs/graph-data-science/current/algorithms/dijkstra-source-target/) algorithm from the GDS library.  Note that bad data in our dataset (such as `null` or `zero` relationship weights) can cause strange results when calculating weighted shortest paths.

```cypher
MATCH     
    (stockholm:OperationalPoint {name:'Stockholms central'}),
    (malmo:OperationalPoint {name:'Malmö central'})
CALL gds.shortestPath.dijkstra.stream('OperationalPoints', {
    sourceNode: malmo,
    targetNode: stockholm,
    relationshipWeightProperty: 'traveltime'
})
YIELD index, sourceNode, targetNode, totalCost, nodeIds, costs, path
RETURN *;
```

Do we get the same result if we use the `sectionlength` relationship property as our weight instead of `traveltime` when computing the shortest path?

```cypher
MATCH     
    (stockholm:OperationalPoint {name:'Stockholms central'}),
    (malmo:OperationalPoint {name:'Malmö central'})
CALL gds.shortestPath.dijkstra.stream('OperationalPoints', {
    sourceNode: malmo,
    targetNode: stockholm,
    relationshipWeightProperty: 'sectionlength'
})
YIELD index, sourceNode, targetNode, totalCost, nodeIds, costs, path
RETURN *;
```

### Community Detection

Now we use the [Leiden Community Detection](https://neo4j.com/docs/graph-data-science/current/algorithms/leiden/) algorithm in 'stream' mode to review `OperationalPoint`s that are _not_ well connected to the network:

```cypher
CALL gds.wcc.stream('OperationalPoints') YIELD nodeId, componentId
WITH collect(gds.util.asNode(nodeId).id) AS nodes, componentId
RETURN nodes, componentId 
ORDER BY size(nodes) ASC LIMIT 50;
```

We can also write the Leiden `communityId` properties to the database so we can query and visualise them later:

```cypher
CALL gds.leiden.write('OperationalPoints', {writeProperty: 'communityId'}) YIELD communityCount, nodeCount
RETURN communityCount, nodeCount;```

We should index our new `communityId` property, so that we can query with it in a performant way:

```cypher
CREATE INDEX index_OperationalPoint_communityId IF NOT EXISTS FOR (opn:OperationalPoint) ON (opn.communityId);
```

Let's find a specific `OperationalPoint` and view the other members of its community. You should see that it belongs to an isolated group of `OperationalPoint`s.

```cypher
MATCH (op:OperationalPoint {id: 'UKN4288'})
WITH op.communityId as component
MATCH path = (:OperationalPoint {communityId: component})-[:SECTION]->()
RETURN path
```

### Centrality

Using the [Degree Centrality](https://neo4j.com/docs/graph-data-science/current/algorithms/degree-centrality/) algorithm we can identify important nodes in the graph based on how many `SECTION` relationships they have.
Nodes with a high Degree Centrality score represent `OperationalPoint`s which are important transfer points in our network.

```cypher
CALL gds.degree.stream('OperationalPoints')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).id AS id, score
ORDER BY score DESC LIMIT 50;
```

We should write the Degree Centrality `degreeScore` properties to the database so we can query and visualise them later:

```cypher
CALL gds.degree.write('OperationalPoints', {writeProperty: 'degreeScore'}) YIELD centralityDistribution
RETURN centralityDistribution;
```

Using the [Betweenness Centrality](https://neo4j.com/docs/graph-data-science/current/algorithms/betweenness-centrality/) algorithm we can identify important nodes in the graph by another metric - those nodes which sit on the shortest path between the most other nodes.

These nodes represent `OperationalPoint`s which many journeys are likely to pass through, and may act as 'bridge' nodes between different parts of the network.

```cypher
CALL gds.betweenness.stream('OperationalPoints')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).id AS id, score
ORDER BY score DESC LIMIT 50;
```

We should also write the Betweenness Centrality scores back to the database so we can query and visualise them later:

```cypher
CALL gds.betweenness.write('OperationalPoints', {writeProperty: 'betweennessScore'}) 
YIELD computeMillis
RETURN computeMillis
```

Let's also index our new Degree Centrality `degreeScore`and Betweenness Centrality `betweennessScore` properties, so that we can query using them in a performant way:

```cypher
CREATE INDEX index_OperationalPoint_degreeScore IF NOT EXISTS FOR (opn:OperationalPoint) ON (opn.degreeScore);
CREATE INDEX index_OperationalPoint_betweennessScore IF NOT EXISTS FOR (opn:OperationalPoint) ON (opn.betweennessScore);
```

### Tidy Up

Finally, it's best practice to remove your graph projections from memory when you're finished with them:

```cypher
CALL gds.graph.drop('OperationalPoints')
```