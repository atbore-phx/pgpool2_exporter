This is a fork of pgpool/pgpool2_exporter, which utilizes gcloud SQL and the pg_stat_replication table to retrieve replication metrics.

When pgpool is deployed in a cloud environment like gcloud (as implemented in this fork),
replicas are often scaled up or down. If a node remains unused or goes down, the metric `pool_nodes` will hang,
potentially causing the Prometheus scraping timeout to be reached.

The original version defines `pool_nodes` as follows:

```golang
		"pool_nodes": {
			"hostname":          {LABEL, "Backend hostname"},
			"port":              {LABEL, "Backend port"},
			"role":              {LABEL, "Role (primary or standby)"},
			"status":            {GAUGE, "Backend node Status (1 for up or waiting, 0 for down or unused)"},
			"select_cnt":        {COUNTER, "SELECT statement counts issued to each backend"},
			"replication_delay": {GAUGE, "Replication delay"},
    }
```

`pool_nodes` is replaced as follows:

- `hostname`, `port`, `role`, `status`, `select_cnt` are considered redundant since they are already present in another metric called `pool_backend_stats`.

- `replication_delay` is substituted with three new metrics, namely: `flush_lag`, `write_lag`, `replay_lag`,
  which are obtained by inspecting the `pg_stat_replication` table on the primary PostgreSQL instance.

  the new `pool_backend_stats`:

```golang
"pool_backend_stats": {
			"hostname":   {LABEL, "Backend hostname"},
			"port":       {LABEL, "Backend port"},
			"role":       {LABEL, "Role (primary or standby)"},
			"status":     {GAUGE, "Backend node Status (1 for up or waiting, 0 for down or unused)"},
			"select_cnt": {COUNTER, "SELECT statement counts issued to each backend"},
			"insert_cnt": {COUNTER, "INSERT statement counts issued to each backend"},
			"update_cnt": {COUNTER, "UPDATE statement counts issued to each backend"},
			"delete_cnt": {COUNTER, "DELETE statement counts issued to each backend"},
			"ddl_cnt":    {COUNTER, "DDL statement counts issued to each backend"},
			"other_cnt":  {COUNTER, "other statement counts issued to each backend"},
			"panic_cnt":  {COUNTER, "Panic message counts returned from backend"},
			"fatal_cnt":  {COUNTER, "Fatal message counts returned from backend)"},
			"error_cnt":  {COUNTER, "Error message counts returned from backend"},
			"flush_lag":  {GAUGE, "flush delay"},
			"write_lag":  {GAUGE, "write delay"},
			"replay_lag": {GAUGE, "reply delay"},
		},
```

# Pgpool-II Exporter

Prometheus exporter for [Pgpool-II](https://pgpool.net) metrics.

Supported Pgpool-II 3.6 and later.

## Building and running

### Build

```
$ make
```

### Running

Running using environment variables:

```
$ export DATA_SOURCE_NAME="postgresql://<user>:<password>@<hostname>:<port>/<dbname>?sslmode=<sslmode>"
OR
# If your password contains special characters, use the environment variables below:
export DATA_SOURCE_USER="<user>"
export DATA_SOURCE_PASS="<password>"
export DATA_SOURCE_URI="<hostname>:<port>/<dbname>?sslmode=<sslmode>"

$ ./pgpool2_exporter <flags>
```

To see all available configuration flags:

```
$ ./pgpool2_exporter --help
```

### Flags

- `help`
  Show context-sensitive help (also try --help-long and --help-man).

- `version`
  Print version information.

- `web.listen-address`
  Address on which to expose metrics and web interface. (default ":9719").

- `web.telemetry-path`
  Path under which to expose metrics. (default "/metrics")

- `log.level`
  Set logging level: one of debug, info, warn, error.

- `log.format`
  Set the log format: one of logfmt, json.

### Docker

This package is available for Docker. The following environment variables configure the docker container:

- `POSTGRES_USERNAME`
  PostgreSQL user name. Default is `postgres`.

- `POSTGRES_PASSWORD`
  PostgreSQL user password. Default is `postgres`.

- `POSTGRES_DATABASE`
  Database name. Default is `postgres`.

- `PGPOOL_SERVICE`
  Pgpool-II hostname. Default is `localhost`.

- `PGPOOL_SERVICE_PORT`
  Pgpool-II port number. Default is `9999`.

- `SSLMODE`
  Whether or not to use SSL. Default is `disable`. Valid values: disable, require, verify-ca, verify-full.

```
docker run --name pgpool2_exporter \
  --net=host --rm \
  -e POSTGRES_USERNAME=<username> \
  -e POSTGRES_PASSWORD=<password> \
  -e POSTGRES_DATABASE=<database> \
  -e PGPOOL_SERVICE=<hostname> \
  -e PGPOOL_SERVICE_PORT=<port> \
  -e SSLMODE=<sslmode> \
  pgpool/pgpool2_exporter:latest
```

### Metrics

| name                                                | Pgpool-II Version | Description                                                            |
| :-------------------------------------------------- | :---------------- | :--------------------------------------------------------------------- |
| pgpool2_frontend_total                              | 3.6+              | Number of total child processes                                        |
| pgpool2_frontend_used                               | 3.6+              | Number of used child processes                                         |
| pgpool2_frontend_used_ratio                         | 3.6+              | Ratio of used child processes to total child processes (0.0 to 1.0)    |
| pgpool2_pool_nodes_status                           | 3.6+              | Backend node Status (1 for up or waiting, 0 for down or unused)        |
| pgpool2_pool_nodes_replication_delay                | 3.6+              | Replication delay                                                      |
| pgpool2_pool_nodes_select_cnt                       | 3.6+              | SELECT query counts issued to each backend                             |
| pgpool2_pool_cache_cache_hit_ratio                  | 3.6+              | Query cache hit ratio                                                  |
| pgpool2_pool_cache_num_cache_entries                | 3.6+              | Number of used cache entries                                           |
| pgpool2_pool_cache_num_hash_entries                 | 3.6+              | Number of total hash entries                                           |
| pgpool2_pool_cache_used_hash_entries                | 3.6+              | Number of used hash entries                                            |
| pgpool2_pool_backend_stats_select_cnt               | 4.2+              | SELECT statement counts issued to each backend                         |
| pgpool2_pool_backend_stats_insert_cnt               | 4.2+              | INSERT statement counts issued to each backend                         |
| pgpool2_pool_backend_stats_update_cnt               | 4.2+              | UPDATE statement counts issued to each backend                         |
| pgpool2_pool_backend_stats_delete_cnt               | 4.2+              | DELETE statement counts issued to each backend                         |
| pgpool2_pool_backend_stats_ddl_cnt                  | 4.2+              | DDL statement counts issued to each backend                            |
| pgpool2_pool_backend_stats_other_cnt                | 4.2+              | other statement counts issued to each backend                          |
| pgpool2_pool_backend_stats_panic_cnt                | 4.2+              | Panic message counts returned from backend                             |
| pgpool2_pool_backend_stats_fatal_cnt                | 4.2+              | Fatal message counts returned from backend                             |
| pgpool2_pool_backend_stats_error_cnt                | 4.2+              | Error message counts returned from backend                             |
| pgpool2_pool_health_check_stats_total_count         | 4.2+              | Number of health check count in total                                  |
| pgpool2_pool_health_check_stats_success_count       | 4.2+              | Number of successful health check count in total                       |
| pgpool2_pool_health_check_stats_fail_count          | 4.2+              | Number of failed health check count in total                           |
| pgpool2_pool_health_check_stats_skip_count          | 4.2+              | Number of skipped health check count in total                          |
| pgpool2_pool_health_check_stats_retry_count         | 4.2+              | Number of retried health check count in total                          |
| pgpool2_pool_health_check_stats_average_retry_count | 4.2+              | Number of average retried health check count in a health check session |
| pgpool2_pool_health_check_stats_max_retry_count     | 4.2+              | Number of maximum retried health check count in a health check session |
| pgpool2_pool_health_check_stats_max_duration        | 4.2+              | Maximum health check duration in Millie seconds                        |
| pgpool2_pool_health_check_stats_min_duration        | 4.2+              | Minimum health check duration in Millie seconds                        |
| pgpool2_pool_health_check_stats_average_duration    | 4.2+              | Average health check duration in Millie seconds                        |
