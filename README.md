# Clusterization practice

## Provisioning

For testing I will use three standalone linux containers running in docker (check docker-compose.yml):

```bash
docker compose up -d
```

Also it is mandatory to have ansible installed with community.docker plugin:

```bash
ansible-galaxy collection install community.docker
```

Check inventory after running compose

```bash
ansible-inventory -i inventory.docker.yml --list
```

Run playbooks, if on macOS set temporarily `export OS_ACTIVITY_MODE=disable`

```bash
ansible-playbook -i inventory.docker.yml <app>/playbook.yml
```

You can verify if instance is running

```bash
docker exec -it <node> redis-server --version
```

Also you can verify the instance type using the following command in a shell:

```bash
redis-cli -p 3010 ROLE
```

To verify autorecover you can shutdown one of the instances, but you should carefully configure timeouts as PC resources are limited, so high number of instances may not manage to vote for replica promotion within a specific timeframe

```bash
docker compose stop <node>
docker exec -it <alive-node> bin/sh
tail -f /var/log/redis_3010.log
```

You can test sharding process using Redis Insight

Cluster has the following architecture
 
![Architecture](/images/architecture.png)

## Redis clusterization

Native redis instrumentation can be leveraged in order to implement high availability and fault tolerance for caching

### Key aspects

- In merory, fast storage: key-value pairs, hash tables, ordered lists, unique sets, score sets, streams, pub/sub etc.
- Persistence: redis stores its data in RAM. The most recent versions of Redis can use hybrid approach for persistence. In order to persist data and ensure low data loss different techniques can be exposed:
    - RDB (Redis database) - periodic snapshots. Fast execution, but data can be lost beetween two iterations
    - AOF (append of file) - logs every write operation. Low execution speed, but high accuracy
- Use cases: caching, session management, real-time analytics, message queues, leaderboards, rate limiting etc.
- Redis provides a total of 16384 hash slots for keys, meaning after the key is sharded, it is delivered to the corresponding hash slot within the cluster. Nodes share the pool evenly, when new instance appears it should be redistributed
- Redis uses asynchronous replication
- In production it is mostly enough to use 1-2 replicas. More replicas should be organized in a tree-structure using the following command. Scale only for read heavy operations:
  ```redis
  REPLICAOF <parent-replica-host> <parent-replica-port>
  ```

### Clusterization

Minimum requirement of 3 nodes. Total number should be odd in order to prevent brain-split problem

### Important configurations

###### Isolation problem

```
min-replicas-to-write 1
min-replicas-max-lag <sec>
```

If master loses access to all of its replicas (minimum is 1), then after the specified amount of seconds it stops accepting new writes or reads

###### Node timeout

```
cluster-node-timeout <timeframe>
```

If node is unreachable within the specified timeframe it is considered dead. It is recomended to use 15 seconds in production