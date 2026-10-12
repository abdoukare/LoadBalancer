# Go Load Balancer

A simple HTTP load balancer written in Go. It forwards requests to a list of backend servers using round-robin selection, retries failed requests, and periodically checks backend health.

## Requirements

- Go 1.20 or newer
- `curl` for testing
- One or more HTTP backend servers

Check your Go installation:

```bash
go version
```

## Build

From the project directory:

```bash
go test ./...
go build -o loadbalancer .
```

The first command verifies the project. There are currently no automated tests, but it confirms that the code compiles and passes Go checks.

## Quick Test With Python Backends

Open four separate terminals and start one backend in each terminal. First create a different response for each backend:

```bash
mkdir -p /tmp/backend{1,2,3,4}
echo "Backend 3031" > /tmp/backend1/index.html
echo "Backend 3032" > /tmp/backend2/index.html
echo "Backend 3033" > /tmp/backend3/index.html
echo "Backend 3034" > /tmp/backend4/index.html
```

Start the backend servers:

```bash
python3 -m http.server 3031 --directory /tmp/backend1
```

```bash
python3 -m http.server 3032 --directory /tmp/backend2
```

```bash
python3 -m http.server 3033 --directory /tmp/backend3
```

```bash
python3 -m http.server 3034 --directory /tmp/backend4
```

Keep these terminals running.

## Start the Load Balancer

Open another terminal in the project directory and run:

```bash
go run . -backends=http://localhost:3031,http://localhost:3032,http://localhost:3033,http://localhost:3034 -port=3030
```

Or run the compiled binary:

```bash
./loadbalancer -backends=http://localhost:3031,http://localhost:3032,http://localhost:3033,http://localhost:3034 -port=3030
```

The load balancer listens on port `3030`.

Keep this terminal running. Stop the server with `Ctrl+C`.

## Send Requests

Use a separate terminal. Do not enter these commands in the terminal where the load balancer is running.

Send one request:

```bash
curl -i http://localhost:3030
```

Send several requests and display each response separately:

```bash
for i in {1..8}; do
  curl -s http://localhost:3030
  echo
done
```

The responses should identify different backend ports, for example:

```text
Backend 3032
Backend 3033
Backend 3034
Backend 3031
```

The exact first backend depends on the round-robin counter.

## Check Backends Directly

Test each backend without going through the load balancer:

```bash
curl http://localhost:3031
curl http://localhost:3032
curl http://localhost:3033
curl http://localhost:3034
```

If a backend is not running, `curl` reports a connection error.

## Test Failure Handling

1. Stop one backend server with `Ctrl+C`, for example the server on port `3031`.
2. Wait for the health-check message in the load balancer terminal. Health checks run every two minutes.
3. Send requests again:

```bash
for i in {1..8}; do
  curl -s http://localhost:3030
  echo
done
```

The load balancer should continue using the available backends. Requests sent to a failed backend are retried and routed to another backend.

## Command-Line Options

```text
-backends  Comma-separated backend URLs. Required.
-port      Port used by the load balancer. Defaults to 3030.
```

Example with two backends:

```bash
go run . -backends=http://localhost:8081,http://localhost:8082 -port=3030
```

## Troubleshooting

### `address already in use`

Another process is already using the selected port. Use another port:

```bash
go run . -backends=http://localhost:3031 -port=4040
```

### `connection refused`

The backend server is not running on the configured port. Start it, or remove that URL from the `-backends` list.

### No response appears

Make sure the load balancer is running in one terminal and run `curl` from a second terminal.

### Stop all test servers

Press `Ctrl+C` in each terminal running a Python backend and in the terminal running the load balancer.
