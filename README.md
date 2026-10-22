# macOS PowerMetrics Exporter

A Prometheus exporter for macOS system metrics using `powermetrics` and `vm_stat` commands. This exporter provides detailed insights into CPU power consumption, GPU power usage, memory statistics, and system performance metrics.

## Features

- **CPU Metrics**: Power consumption, frequency, and residency (active/idle time)
- **GPU Metrics**: Power consumption and residency (active/idle time)
- **Memory Metrics**: Virtual memory statistics from `vm_stat`
- **Page Size**: Dynamic page size detection
- **Prometheus Format**: Native Prometheus metrics format
- **LaunchDaemon Support**: Automatic startup with macOS as root service
- **Modular Architecture**: Clean separation of concerns with internal packages

## Architecture

The application is structured with a modular architecture:

```
├── cmd/
│   └── main.go                    # Application entry point
├── internal/
│   ├── collector/
│   │   ├── powermetrics.go        # PowerMetrics collector
│   │   └── vmstat.go              # VM statistics collector
│   ├── config/
│   │   └── config.go              # Configuration management
│   └── server/
│       └── server.go              # HTTP server and metrics endpoint
└── test/e2e_test.go               # End-to-end tests
```

## Prerequisites

- macOS (tested on macOS 14.x+)
- Go 1.27.3+ (for building from source)
- Administrator access for LaunchDaemon installation
- Prometheus server (for metrics collection)

## Installation

### Building from Source

1. Clone the repository:
```bash
git clone <repository-url>
cd mac-powermetrics-exporter
```

2. Build the binary:
```bash
go build -o mac-powermetrics-exporter cmd/main.go
```

3. Install the binary:
```bash
sudo cp mac-powermetrics-exporter /usr/local/bin/
sudo chmod +x /usr/local/bin/mac-powermetrics-exporter
```

## Usage

### Manual Execution

Run the exporter manually (requires sudo):
```bash
sudo ./mac-powermetrics-exporter
```

Or run directly from source:
```bash
sudo go run cmd/main.go
```

The exporter will start an HTTP server on port 9127. Access metrics at:
```
http://localhost:9127/metrics
```

### LaunchDaemon Setup (Automatic Startup)

The exporter runs as a LaunchDaemon with root privileges to access `powermetrics` without additional sudo configuration.

1. Update the plist file to point to the correct binary location:
```bash
# Edit ninja.oppai.mac-powermetrics-exporter.plist if needed
# Ensure the ProgramArguments points to /usr/local/bin/mac-powermetrics-exporter
```

2. Copy the plist file to LaunchDaemons directory:
```bash
sudo cp ninja.oppai.mac-powermetrics-exporter.plist /Library/LaunchDaemons/
```

3. Set proper permissions:
```bash
sudo chown root:wheel /Library/LaunchDaemons/ninja.oppai.mac-powermetrics-exporter.plist
sudo chmod 644 /Library/LaunchDaemons/ninja.oppai.mac-powermetrics-exporter.plist
```

4. Load the LaunchDaemon:
```bash
sudo launchctl load /Library/LaunchDaemons/ninja.oppai.mac-powermetrics-exporter.plist
```

5. Verify the service is running:
```bash
sudo launchctl list | grep mac-powermetrics-exporter
```

6. To unload the service:
```bash
sudo launchctl unload /Library/LaunchDaemons/ninja.oppai.mac-powermetrics-exporter.plist
```

7. To start/stop the service manually:
```bash
# Start
sudo launchctl start ninja.oppai.mac-powermetrics-exporter

# Stop
sudo launchctl stop ninja.oppai.mac-powermetrics-exporter
```

## Development

### Running Tests

Run the end-to-end tests to verify functionality:
```bash
go test -v
```

Run tests for all packages:
```bash
go test -v ./...
```

### Project Structure

- **`cmd/main.go`**: Application entry point that initializes configuration and starts the server
- **`internal/config/`**: Configuration management with default values
- **`internal/collector/`**: Metric collectors for powermetrics and vm_stat
- **`internal/server/`**: HTTP server setup and Prometheus metrics endpoint
- **`e2e_test.go`**: End-to-end tests that verify the complete application functionality

## Metrics

### PowerMetrics (CPU/GPU)

| Metric Name | Type | Description | Labels |
|-------------|------|-------------|---------|
| `powermetrics_cpu_power_milliwatts` | Gauge | CPU power consumption in milliwatts | - |
| `powermetrics_gpu_power_milliwatts` | Gauge | GPU power consumption in milliwatts | - |
| `powermetrics_cpu_frequency_hertz` | Gauge | CPU frequency in Hertz | `core` |
| `powermetrics_cpu_temperature_celsius` | Gauge | CPU temperature in Celsius | `sensor_id` |
| `powermetrics_cpu_active_residency_percent` | Gauge | CPU active time percentage | `core` |
| `powermetrics_cpu_idle_residency_percent` | Gauge | CPU idle time percentage | `core` |
| `powermetrics_gpu_active_residency_percent` | Gauge | GPU active time percentage | - |
| `powermetrics_gpu_idle_residency_percent` | Gauge | GPU idle time percentage | - |

### VM Statistics (Memory)

| Metric Name | Type | Description |
|-------------|------|-------------|
| `vmstat_page_size_bytes` | Gauge | System page size in bytes |
| `vmstat_pages_free_count` | Gauge | Number of free pages |
| `vmstat_pages_active_count` | Gauge | Number of active pages |
| `vmstat_pages_inactive_count` | Gauge | Number of inactive pages |
| `vmstat_pages_speculative_count` | Gauge | Number of speculative pages |
| `vmstat_pages_throttled_count` | Gauge | Number of throttled pages |
| `vmstat_pages_wired_count` | Gauge | Number of wired pages |
| `vmstat_pages_purgeable_count` | Gauge | Number of purgeable pages |
| `vmstat_pages_cow_faults_total` | Counter | Number of copy-on-write faults |
| `vmstat_pages_zero_filled_total` | Counter | Number of zero-filled pages |
| `vmstat_pages_reactivated_total` | Counter | Number of reactivated pages |
| `vmstat_pages_purged_total` | Counter | Number of purged pages |
| `vmstat_pages_file_backed_count` | Gauge | Number of file-backed pages |
| `vmstat_pages_anonymous_count` | Gauge | Number of anonymous pages |
| `vmstat_pages_compressor_count` | Gauge | Number of pages in compressor |
| `vmstat_pages_decompressed_total` | Counter | Number of decompressed pages |
| `vmstat_pages_compressed_total` | Counter | Number of compressed pages |
| `vmstat_page_ins_total` | Counter | Number of page-ins |
| `vmstat_page_outs_total` | Counter | Number of page-outs |
| `vmstat_faults_total` | Counter | Number of page faults |
| `vmstat_swap_ins_total` | Counter | Number of swap-ins |
| `vmstat_swap_outs_total` | Counter | Number of swap-outs |

## Prometheus Configuration

Add the following to your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'mac-powermetrics'
    static_configs:
      - targets: ['localhost:9127']
    scrape_interval: 30s
```

## Example Queries

### Power Consumption
```promql
# Total system power (CPU + GPU) in watts
(powermetrics_cpu_power_milliwatts + powermetrics_gpu_power_milliwatts) / 1000

# CPU power efficiency (performance per watt)
rate(powermetrics_cpu_active_residency_percent[5m]) / (powermetrics_cpu_power_milliwatts / 1000)
```

### Memory Usage
```promql
# Free memory in bytes
vmstat_pages_free_count * vmstat_page_size_bytes

# Memory utilization percentage
(1 - (vmstat_pages_free_count / (vmstat_pages_free_count + vmstat_pages_active_count + vmstat_pages_inactive_count))) * 100
```

### CPU Performance
```promql
# Average CPU utilization across all cores
avg(powermetrics_cpu_active_residency_percent)

# Highest CPU core utilization
max(powermetrics_cpu_active_residency_percent)
```

## Configuration

### Port Configuration

To change the default port (9127), modify the `internal/config/config.go` file:

```go
func New() *Config {
	return &Config{
		Port: ":YOUR_PORT",
	}
}
```

### Sampling Interval

The exporter uses a 1-second sampling interval for `powermetrics`. To modify this, change the `-i` parameter in the `powermetrics` command within the `internal/collector/powermetrics.go` file.

### Adding New Collectors

To add new metric collectors:

1. Create a new collector in `internal/collector/`
2. Implement the `prometheus.Collector` interface
3. Register the collector in `internal/server/server.go`

Example:
```go
// In internal/server/server.go
prometheus.MustRegister(collector.NewYourNewCollector())
```

## Troubleshooting

### Common Issues

1. **Permission Denied**: Ensure sudo permissions are configured correctly for `powermetrics`
2. **Command Not Found**: Verify `powermetrics` is available (should be on all modern macOS systems)
3. **High CPU Usage**: Consider increasing the sampling interval if the exporter consumes too many resources
4. **Build Errors**: Ensure Go modules are properly initialized with `go mod tidy`

### Logs

Check the LaunchDaemon logs:
```bash
tail -f /var/log/mac-powermetrics-exporter.out.log
tail -f /var/log/mac-powermetrics-exporter.err.log
```

### Testing

Test the exporter manually:
```bash
curl http://localhost:9127/metrics
```

Run the end-to-end tests:
```bash
go test -v -run TestE2E
```

### Debugging

For development and debugging:
```bash
# Run with verbose logging
sudo go run cmd/main.go

# Check if collectors are working
curl -s http://localhost:9127/metrics | grep -E "(powermetrics|vmstat)" | head -10
```

## Security Considerations

- The exporter runs as root via LaunchDaemon to access `powermetrics`
- LaunchDaemon provides better security isolation than user-level sudo access
- Restrict network access to the metrics endpoint (consider firewall rules)
- Monitor system logs for service activity
- The service automatically restarts if it crashes (KeepAlive=true)
- Internal packages are not exposed externally, following Go best practices

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests if applicable (especially for new collectors)
5. Run tests: `go test -v ./...`
6. Submit a pull request

### Code Structure Guidelines

- Keep collectors in `internal/collector/`
- Configuration changes go in `internal/config/`
- Server modifications in `internal/server/`
- Add end-to-end tests for new functionality
- Follow Go naming conventions and add appropriate documentation

## License

MIT

## Acknowledgments

- Built with [Prometheus Go client library](https://github.com/prometheus/client_golang)
- Uses macOS `powermetrics` and `vm_stat` system utilities
- Follows Go project layout standards
