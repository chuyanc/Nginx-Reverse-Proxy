This is the documentation of how to deploy Neo4j on a single VM instance and how to load dow30 & sp500 data inside.

## Task Description
### Goal
Deploy Neo4j to a single VM instance, and load data from JSON files to the deployed Neo4j.

### Definition of Done
1. Launch a VM instance. Install the neo4j binary and start the DB server.
2. Load JSON graph data into it. 

## Solution
### Deploy Neo4j to a single VM instance
#### 1. Add dependencies & repository
1.1 Update your package repository information:
   
   ```
   $ sudo apt update
   ```
1.2 Install the required Neo4j dependencies:

   ```
   $ sudo apt install wget curl nano software-properties-common dirmngr apt-transport-https gnupg gnupg2 ca-certificates lsb-release unzip -y
   ```
1.3 Import the Neo4j GPG key to verify the Neo4j APT repository packages:

   ```
   $ wget -O - https://debian.neo4j.com/neotechnology.gpg.key | sudo apt-key add -
   ```
1.4 Add the Neo4j APT repository to your system:
   
   For Debian 11 / Ubuntu 22.04:

   ```
   $ echo 'deb https://debian.neo4j.com stable 4.4' | sudo tee -a /etc/apt/sources.list.d/neo4j.list
   ```

   For Debian 10 / Ubuntu 20.04:
   
   ```
   $ echo 'deb https://debian.neo4j.com stable 4.0' | sudo tee -a /etc/apt/sources.list.d/neo4j.list
   ```
1.5 Update the package list again to include the Neo4j repository:

   ```
   $ sudo apt update
   ```
#### 2. Install and start Neo4j
2.1 Install Neo4j:

  ```
  $ sudo apt install neo4j -y
  ```
2.2 Start Neo4j service

  ```
  $ sudo systemctl start neo4j
  ```
2.3 Enable Neo4j to start on boot:

  ```
  $ sudo systemctl enable neo4j
  ```
2.4 Check whether the Neo4j instance is up:

  ```
  $ sudo systemctl status neo4j
  ```
#### 3. Configuration for connections beyond localhost
3.1 Open Neo4j configuration file at ```/etc/neo4j/neo4j.conf```, and edit the following connection lines as:

```
# Bolt connector
dbms.connector.bolt.enabled=true
dbms.connector.bolt.tls_level=DISABLED
dbms.connector.bolt.listen_address=:7687
# dbms.connector.bolt.advertised_address=:7687

# HTTP Connector. There can be zero or one HTTP connectors
dbms.connectors.default_listen_address=0.0.0.0
dbms.connector.http.enabled=true
dbms.connector.http.listen_address=:7474
# dbms.connector.http.advertised_address=:7474

# HTTPS Connector. There can be zero or one HTTPS connectors.
dbms.connector.https.enabled=false
# dbms.connector.https.listen_address=:7473
# dbms.connector.https.advertised_address=:7473
```

3.2 Then use ```$ sudo systemctl restart neo4j``` to restart the server.

3.3 After allowing all the http traffics to the VM instance, we can directly visit the Neo4j browser via http://public-ip:7474, which means Neo4j is configured successfully.

### Load JSON data to the Neo4j instance
The graph JSON data can be accessed here: 

dow 30: https://github.com/WRDSTech/competition-graph-frontend/blob/main/src/assets/data/dow30_relation_backend.json

sp 500: https://github.com/WRDSTech/competition-graph-frontend/blob/main/src/assets/data/sp500_relation_backend.json

#### 1. Install APOC library
1.1 Download APOC JAR file: Go to https://github.com/neo4j-contrib/neo4j-apoc-procedures and download the APOC JAR file which corresponds to your current Neo4j version.

1.2 Download GDS JAR file: Go to https://github.com/neo4j/graph-data-science and download the GDS JAR file which corresponds to your current Neo4j version.

1.2 Install the APOC library: Copy the downloaded JAR files into the "plugins" directory of your Neo4j installation (usually ```/var/lib/neo4j/plugins```).

1.3 Open Neo4j configuration file at ```/etc/neo4j/neo4j.conf```, and add the following line at the end of file to allow APOC procedures:

```
dbms.security.procedures.unrestricted=apoc.*
dbms.security.procedures.unrestricted=gds.*
```
1.4 Then use ```$ sudo systemctl restart neo4j``` to restart the server and install APOC library.

#### 2. Login to the Neo4j server and check APOC validation
2.1 In order to login, we can either use the user interface on http://public-ip:7474 or use command line to enter:

```
$ cypher-shell -u "" -p "" -a bolt://localhost:7687
```
The default username and password will both be 'neo4j', you'll be asked to change password then.

2.2 After entering the shell to enter Cypher query, first type in ```CALL apoc.help("apoc.version");``` to check if we successfully installed APOC.

#### 3. Load JSON data
Upon this step, we can use APOC library to directly load JSON data to Neo4j.

Use dow 30 data as an example, we have to write Cypher query to load node data (nodes), then relationship data (links).

To load nodes:

```
CALL apoc.load.json("https://raw.githubusercontent.com/WRDSTech/competition-graph-frontend/main/src/assets/data/dow30_relation_backend.json") YIELD value
             UNWIND value.nodes AS nodeData
             MERGE (node:Node {id: nodeData.id})
             SET node.name = nodeData.name, node.graph = 'dow30';
```

To load links:

```
CALL apoc.load.json("https://raw.githubusercontent.com/WRDSTech/competition-graph-frontend/main/src/assets/data/dow30_relation_backend.json") YIELD value
             UNWIND value.links AS linkData
             MATCH (sourceNode:Node {id: linkData.source})
             MATCH (targetNode:Node {id: linkData.target})
             MERGE (sourceNode)-[:RELATIONSHIP_TYPE {id: linkData.id, category: linkData.category}]->(targetNode);
```

The same steps also apply to sp 500 dataset:

```
CALL apoc.load.json("https://raw.githubusercontent.com/WRDSTech/competition-graph-frontend/main/src/assets/data/sp500_relation_backend.json") YIELD value
             UNWIND value.nodes AS nodeData
             MERGE (node:Node {id: nodeData.id})
             SET node.name = nodeData.name, node.graph = 'sp500';
```
```
CALL apoc.load.json("https://raw.githubusercontent.com/WRDSTech/competition-graph-frontend/main/src/assets/data/sp500_relation_backend.json") YIELD value
             UNWIND value.links AS linkData
             MATCH (sourceNode:Node {id: linkData.source})
             MATCH (targetNode:Node {id: linkData.target})
             MERGE (sourceNode)-[:RELATIONSHIP_TYPE {id: linkData.id, category: linkData.category}]->(targetNode);
```



