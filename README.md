# watchdock
docker container to monitor availability of other containers and publish their state via an HTTP API

## tldr
simply add a label to a container to make it monitorable via watchdock. watchdock checks its container health and allows it to be published via an http endpoint, resulting in an 200 if all containers are healthy or 500 if not:
```
services:
  foo:
    labels:
      watchdock.v1.healthcheck.enabled: true
```

if the service has no health check configured, a different docker container state can be checked by changing the successState also via a label:
```
services:
  bar:
    labels:
      watchdock.v1.healthcheck.enabled: true
      watchdock.v1.healthcheck.successState: running
```

intermediate states, transitioning from "soft" to "hard" states are displayed via an 202 status code if a transition to success/healthy is evaluated. 

## why?
having a container stack running on a system and attaching it to an external monitoring tool can be quiet hard if the services are just processing in the background. If you want to use Load Balancer Healtchecks as done via Target Groups in AWS you always need an an endpoint that gives response about the overall healthiness of the system. Here watchdock is the sidecar container you could benefit from.

## how?
given this example docker compose file we create two containers, both having a healthcheck, one succeeding, one failing and two containers without any healthcheck, one that keeps running and one that just exits:
```
version: '3.8'

services:
  foo:
    image: busybox
    command: tail -f /dev/null  # Keeps the container running indefinitely
    labels:
      watchdock.v1.healthcheck.enabled: true
    healthcheck:
      test: ["CMD", "true"] # Succeeds healthcheck
      interval: 10s
      retries: 3
      start_period: 5s
      timeout: 5s

  bar:
    image: busybox
    command: tail -f /dev/null
    labels:
      watchdock.v1.healthcheck.enabled: true
    healthcheck:
      test: ["CMD", "false"] # Fails healthcheck
      interval: 10s
      retries: 3
      start_period: 5s
      timeout: 5s

  oof:
    image: busybox
    command: tail -f /dev/null  # Keeps the container running indefinitely
    labels:
      watchdock.v1.healthcheck.enabled: true
      watchdock.v1.healthcheck.successState: running

  rab:
    image: busybox
    command: /bin/true
    labels:
      watchdock.v1.healthcheck.enabled: true
      watchdock.v1.healthcheck.successState: running

  watchdock:
    restart: always
    group_add:
      - ${SYSTEM_DOCKER_SOCKET_GID:-994}                  # adds internal user to docker sockets user group to allow communication via file
    environment:
      APP_DOCKER_SOCKET: /var/run/docker.sock             # pass through of docker socket to query other containers
      APP_HEALTHCHECK_MODE: on-demand                     # or "periodic" if healthchecks should be evaluated on an interval and not ad-hoc
      APP_HEALTHCHECK_PERIODIC_INTERVAL=30                # only used in "periodic" mode. determines time inbetween checks
      APP_HEALTHCHECK_RETRIES=5                           # determines retries to fetch container state if internal failure occurs
      APP_PORT: 3000                                      # port on which http endpoint is published, only route being /healthcheck right now
      APP_HEALTHCHECK_CONSECUTIVE_FAILURES_THRESHOLD: 3   # number of consecutive failing requests per container to determine a transition from "soft" to "hard" state
      APP_HEALTHCHECK_CONSECUTIVE_SUCCESSES_THRESHOLD: 2  # same as above but handling transition to "hard" success state per container
    ports:
      - 8080:3000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro      # pass through of docker socket to query other containers in read-only mode
```

starting this setup with docker compose results in this state for all 4 containers:
```
NAME              IMAGE     COMMAND               SERVICE   CREATED         STATUS                      PORTS
watchdock-bar-1   busybox   "tail -f /dev/null"   bar       3 minutes ago   Up 35 seconds (unhealthy)
watchdock-foo-1   busybox   "tail -f /dev/null"   foo       3 minutes ago   Up 35 seconds (healthy)
watchdock-oof-1   busybox   "tail -f /dev/null"   oof       3 minutes ago   Up 35 seconds
watchdock-rab-1   busybox   "/bin/true"           rab       3 minutes ago   Exited (0) 34 seconds ago
```

querying the healtcheck endpoint of watchdock represents this too:
```
curl -s http://localhost:8080/healthcheck | jq .
{
  "status": 500,
  "message": "Healthcheck failed: Some containers are unhealthy",
  "containers": {
    "/files_bar_1": "unhealthy",
    "/files_rab_1": "unhealthy",
    "/files_foo_1": "healthy",
    "/files_oof_1": "healthy"
  },
  "healthMetrics": {
    "consecutiveFailures": {
      "/files_bar_1": 4,
      "/files_rab_1": 4,
      "/files_foo_1": 0,
      "/files_oof_1": 0
    },
    "consecutiveSuccesses": {
      "/files_bar_1": 0,
      "/files_rab_1": 0,
      "/files_foo_1": 4,
      "/files_oof_1": 4
    }
  }
}
```

