 I believe this repo demonstrates issues we're seeing with Kubernetes readiness probes and our multi-region, multi-node CockroachDB cluster.

 ## Getting started

Step #1:
```
  $> docker-compose up crdb-a
  $> docker-compose exec crdb-a cockroach sql \
      --execute "CREATE USER hydra; CREATE DATABASE hydra; GRANT ALL ON DATABASE hydra TO hydra;" \
      --url "postgres://root@crdb-a:26257/hydra?sslmode=disable" \
      --insecure
```

Step #2:
```
  $> docker-compose run --rm hydra migrate sql -e --config /.hydra.yaml
```

Step #3:
```
  $> docker-compose up hydra
```

Step #4:

_(Do this in a new terminal window)_

```
  $> while sleep 30; do curl --max-time 30 http://localhost:4444/health/ready; done
```

Step #5:

_(Do this in a new terminal window too)_

Go through each CockroachDB service and stop it, starting with `crdb-a` and stop it with `$> docker-compose stop crdb-a` and observe the response from the ready check.

If the ready check continues to succeed then start that CockroachDB service back up and proceed to stop the next CockroachDB service.  Eventually hydra will begin to respond with `503 Service Unavailable` errors. Bringing the CockroachDB service back online will result in the ready checks succeeding again. But a Hydra pod should not pinned to a specific node in the cluster, there are no guarantees that nodes won't go down and be replaced at a different IP address.

During a rolling patch update (which can't be scheduled with Cockroach Cloud) _all_ nodes are eventually brought down, updated, and reintroduced to the cluster with different IP addresses. This leads to downtime because _all_ Hydra pods will eventually be marked "not ready" and there won't be any pods left to handle traffic.