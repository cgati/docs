---
title: Fault Tolerance & Recovery
summary: Use a local cluster to explore how CockroachDB remains available during, and recovers after, failure.
toc: false
---

This page demonstrates how CockroachDB remains available during, and recovers after, failure. Starting with a 3-node local cluster, you'll remove a node and see how the cluster continues uninterrupted. You'll then add data while the node is offline, rejoin the node, and see how it catches up with the rest of the cluster. Finally, you'll add a fourth node, remove a node again, and see how missing replicas eventually re-replicate to the new node.

<div id="toc"></div>

## Before You Begin

Make sure you have already:

- [Installed CockroachDB](install-cockroachdb.html) 
- [Started a 3-node local cluster](start-a-local-cluster.html) in insecure mode

## Step 1. Restart the local cluster

If your [local cluster](start-a-local-cluster.html) isn't up and running already, restart the nodes:

~~~ shell
# Restart node 1:
$ cockroach start --background

# Restart node 2:
$ cockroach start --background \
--store=node2 \
--port=26258 \
--http-port=8081 \
--join=localhost:26257

# Restart node 3:
$ cockroach start --background \
--store=node3 \
--port=26259 \
--http-port=8082 \
--join=localhost:26257
~~~

Open the built-in SQL shell on any node to verify that the cluster is live:

~~~ shell
$ cockroach sql
# Welcome to the cockroach SQL interface.
# All statements must be terminated by a semicolon.
# To exit: CTRL + D.
~~~

~~~ sql
> SHOW DATABASES;
~~~

~~~ 
+--------------------+
|      Database      |
+--------------------+
| information_schema |
| pg_catalog         |
| bank               |
| system             |
+--------------------+
(4 rows)
~~~

Use **CTRL + D**, **CTRL + C**, or `\q` to exit the SQL shell.

## Step 2. Remove a node temporarily

Use the [`cockroach quit`](stop-a-node.html) command to stop node 2 (port `26258`):

~~~ shell
$ cockroach quit --port=26258
~~~

~~~
initiating graceful shutdown of server
ok
~~~

## Step 3. Verify that the cluster remains available

Open the built-in SQL shell on node 1 (port `26257`) or node 3 (port `26259`):

~~~ shell
$ cockroach sql --port=26259
# Welcome to the cockroach SQL interface.
# All statements must be terminated by a semicolon.
# To exit: CTRL + D.
~~~

~~~ sql
> SHOW DATABASES;
~~~

~~~ 
+--------------------+
|      Database      |
+--------------------+
| information_schema |
| pg_catalog         |
| bank               |
| system             |
+--------------------+
(4 rows)
~~~

As you see, despite one node being offline, the cluster continues uninterrupted because a majority of replicas (2/3) remains available. If you were to remove another node, however, leaving only one node live, the cluster would be unresponsive until another node was brought back online.

Use **CTRL + D**, **CTRL + C**, or `\q` to exit the SQL shell.

## Step 4. Add data while the node is offline

Use the [`cockroach gen`](generate-cli-utilities-and-example-data.html) command to generate an example `startrek` database:

<div class="language-shell highlighter-rouge"><pre class="highlight"><code data-eventcategory="fault1-gen-data"><span class="gp noselect shellterminal"></span>cockroach gen example-data | cockroach sql
</code></pre>
</div>

~~~
CREATE DATABASE
SET
DROP TABLE
DROP TABLE
CREATE TABLE
INSERT 79
CREATE TABLE
INSERT 200
~~~

Once again, open the SQL shell on node 1 (port `26258`) or node 3 (port `26259`) and verify that the new `startrek` database was added with two tables, `episodes` and `quotes`:

~~~ sql
> SHOW DATABASES;
~~~

~~~
+--------------------+
|      Database      |
+--------------------+
| information_schema |
| pg_catalog         |
| bank               |
| startrek           |
| system             |
+--------------------+
(5 rows)
~~~

~~~ sql
> SHOW TABLES FROM startrek;
~~~

~~~
+----------+
|  Table   |
+----------+
| episodes |
| quotes   |
+----------+
(2 rows)
~~~

~~~ sql
> SELECT * FROM startrek.episodes LIMIT 10;
~~~

~~~
+----+--------+-----+--------------------------------+----------+
| id | season | num |             title              | stardate |
+----+--------+-----+--------------------------------+----------+
|  1 |      1 |   1 | The Man Trap                   |   1531.1 |
|  2 |      1 |   2 | Charlie X                      |   1533.6 |
|  3 |      1 |   3 | Where No Man Has Gone Before   |   1312.4 |
|  4 |      1 |   4 | The Naked Time                 |   1704.2 |
|  5 |      1 |   5 | The Enemy Within               |   1672.1 |
|  6 |      1 |   6 | Mudd's Women                   |   1329.8 |
|  7 |      1 |   7 | What Are Little Girls Made Of? |   2712.4 |
|  8 |      1 |   8 | Miri                           |   2713.5 |
|  9 |      1 |   9 | Dagger of the Mind             |   2715.1 |
| 10 |      1 |  10 | The Corbomite Maneuver         |   1512.2 |
+----+--------+-----+--------------------------------+----------+
(10 rows)
~~~

Use **CTRL + D**, **CTRL + C**, or `\q` to exit the SQL shell.

## Step 5. Rejoin the node to the cluster

Rejoin node 2 to the cluster, using the same command that you used in step 1:

~~~ shell
$ cockroach start --background \
--store=node2 \
--port=26258 \
--http-port=8081 \
--join=localhost:26257
~~~

~~~
CockroachDB node starting at {{site.data.strings.start_time}}
build:      beta-20161103-97-g581c2bc @ 2016/11/07 16:54:20 (go1.7.1)
admin:      http://localhost:8081
sql:        postgresql://root@localhost:26258?sslmode=disable
logs:       node2/logs
store[0]:   path=node2
status:     restarted pre-existing node
clusterID:  {5638ba53-fb77-4424-ada9-8a23fbce0ae9}
nodeID:     2
~~~

## Step 6. Verify that the rejoined node has caught up

Open the SQL shell on the rejoined node 2 (port `26258`) and check for the `startrek` data that was added while the node was offline:

~~~ shell
$ cockroach sql --port=26258
# Welcome to the cockroach SQL interface.
# All statements must be terminated by a semicolon.
# To exit: CTRL + D.
~~~

~~~ sql
> SELECT * FROM startrek.episodes LIMIT 10;
~~~

~~~
+----+--------+-----+--------------------------------+----------+
| id | season | num |             title              | stardate |
+----+--------+-----+--------------------------------+----------+
|  1 |      1 |   1 | The Man Trap                   |   1531.1 |
|  2 |      1 |   2 | Charlie X                      |   1533.6 |
|  3 |      1 |   3 | Where No Man Has Gone Before   |   1312.4 |
|  4 |      1 |   4 | The Naked Time                 |   1704.2 |
|  5 |      1 |   5 | The Enemy Within               |   1672.1 |
|  6 |      1 |   6 | Mudd's Women                   |   1329.8 |
|  7 |      1 |   7 | What Are Little Girls Made Of? |   2712.4 |
|  8 |      1 |   8 | Miri                           |   2713.5 |
|  9 |      1 |   9 | Dagger of the Mind             |   2715.1 |
| 10 |      1 |  10 | The Corbomite Maneuver         |   1512.2 |
+----+--------+-----+--------------------------------+----------+
(10 rows)
~~~

At first, while node 2 is catching up, it acts as a proxy to one of the other nodes with the data. This shows that even when a copy of the data is not local to the node, it has seamless access.

Soon enough, node 2 catches up entirely. To verify, open the Admin UI at `http://localhost:8080`, go to the **Nodes** tab, and you'll see that all three nodes are listed, and the replica count is identical for each. This means that all data in the cluster has been replicated 3 times; there's a copy of every piece of data on each node.

{{site.data.alerts.callout_success}}CockroachDB replicates data 3 times by default. You can customize the number and location of replicas for the entire cluster or for specific sets of data using <a href="configure-replication-zones.html">replication zones</a>.{{site.data.alerts.end}}

<img src="images/recovery1.png" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Step 7. Add another node

Now, to prepare the cluster for a permanent node failure, add a fourth node:

<div class="language-shell highlighter-rouge"><pre class="highlight"><code data-eventcategory="fault2-add-node"><span class="gp noselect shellterminal"></span>cockroach start --background <span class="se">\</span>
--store<span class="o">=</span>node4 <span class="se">\</span>
--port<span class="o">=</span>26260 <span class="se">\</span>
--http-port<span class="o">=</span>8083 <span class="se">\</span>
--join<span class="o">=</span>localhost:26257
</code></pre>
</div>

~~~
CockroachDB node starting at {{site.data.strings.start_time}}
build:      beta-20161103-97-g581c2bc @ 2016/11/07 16:54:20 (go1.7.1)
admin:      http://localhost:8083
sql:        postgresql://root@localhost:26260?sslmode=disable
logs:       node4/logs
store[0]:   path=node4
status:     initialized new node, joined pre-existing cluster
clusterID:  {5638ba53-fb77-4424-ada9-8a23fbce0ae9}
nodeID:     4
~~~

## Step 8. Remove a node permanently

Again use the [`cockroach quit`](stop-a-node.html) command to stop node 2 (port `26258`):

~~~ shell
$ cockroach quit --port=26258
~~~

~~~
initiating graceful shutdown of server
ok
~~~

## Step 9. Verify that the cluster re-replicates missing replicas

Go back to the **Nodes** tab of the Admin UI and you'll see 4 nodes listed. After about 1 minute, the dot next to node 2 will turn yellow, indicating that the node is not responding. 

<img src="images/recovery2.png" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

After about 10 minutes, the dot next to node 2 will turn red, indicating that the node is dead and not expected to come back. At this point, you should also see that the **Replicas** count for node 4 matches the count for node 1 and 3, the other live nodes. This indicates that all missing replicas (those that were on node 2) have been re-replicated to node 4.

<img src="images/recovery3.png" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

{{site.data.alerts.callout_info}}The <strong>Replicas</strong> count for node 2 should go down to 0, but there's currently a UI bug preventing this. Follow <a href="https://github.com/cockroachdb/cockroach/issues/5415">issue #5415</a> for more details.{{site.data.alerts.end}}

## What's Next?

Use a local cluster to explore these other core CockroachDB features:

- [Data Replication](demo-data-replication.html)
- [Scalability](demo-scalability.html)
