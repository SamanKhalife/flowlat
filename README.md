# flowlat

flowlat is a network latency measurement tool that captures [TCP packets](https://en.wikipedia.org/wiki/Transmission_Control_Protocol), extracts key attributes, and calculates the [round-trip time (RTT)](https://www.cloudflare.com/de-de/learning/cdn/glossary/round-trip-time-rtt/) between SYN and SYN-ACK packets using [eBPF](https://ebpf.io/). The tool supports both IPv4 and IPv6 and is implemented in a combination of C and Go.

## Prerequisites

- **Linux Kernel with eBPF Support**: Ensure your Linux kernel version supports eBPF and has the necessary headers.
- **Go 1.21 or Higher**: Required for building the Go components.
- **Clang/LLVM**: Needed for compiling the eBPF C code.

### Installing Dependencies (If Needed)

```bash
sudo apt install linux-headers-$(uname -r) \
                 libbpfcc-dev \
                 libbpf-dev \
                 llvm \
                 clang \
                 gcc-multilib \
                 build-essential \
                 linux-tools-$(uname -r) \
                 linux-tools-common \
                 linux-tools-generic
```

```bash
sudo add-apt-repository ppa:longsleep/golang-backports
sudo apt update
sudo apt install golang-go
```

## Installation

1. Clone the Repository:

    ```bash
    git clone https://github.com/markpash/flowlat.git
    cd flowlat
    ```

2. Build the eBPF Program (skip this step `probe_bpfeb.go` and `probe_bpfel.go` are already present in the `internal/probe` directory):

    Use the [`bpf2go`](https://pkg.go.dev/github.com/cilium/ebpf/cmd/bpf2go) tool to compile the eBPF code:

    ```bash
    cd internal/probe
    go generate
    ```

   Note: `bpf2go` is a tool that compiles eBPF code written in C and automatically generates Go code so you can easily use the eBPF program in your Go project. It makes it simple to work with eBPF in Go without needing to handle the complex details of compiling and integrating the eBPF code.

   Note: The difference between `probe_bpfeb.go` and `probe_bpfel.go` is related to the [endianness](https://developer.mozilla.org/en-US/docs/Glossary/Endianness) of the target architecture (`bpfeb` for big-endian and `bpfel` for little-endian).

3. Lets cleans up your Go module's dependencies by adding any missing ones and removing any that aren't used:
    ```bash
    sudo go mod tidy
    ```


4. Build the Go Application:

    ```bash
    go build -o flowlat cmd/flowlat.go
    ```

## Usage

### Running flowlat

To run flowlat, specify the network interface to which you want to attach the probe:

```bash
sudo ./flowlat -iface eth0
```

- **`-iface`**: Specifies the network interface to attach the eBPF probe (default: `eth0`).

Note: Use `ifconfig -a` to list available network interfaces. Then, specify the desired interface in the `-iface` option.


### Example Output

flowlat captures SYN and SYN-ACK packets, calculates the RTT, and prints the result:

```bash
2001:db8::1:80 : RTT: 12ms
```

## Architecture Overview

### C Component (eBPF Program)

- **probe.c**: The eBPF program is implemented in C and captures packet attributes (source/destination IPs, ports, flags) using TC BPF hooks. The data is then sent to user space via a perf event array.

### Go Components

- **probe.go**: Manages the interaction with the eBPF program, including setting up filters and handling incoming data.
- **clsact.go**: Implements a `clsact` qdisc, used to apply the eBPF filters.
- **packet.go**: Handles TCP packet manipulation and serialization/deserialization.
- **tc.go**: Defines constants related to the TC subsystem in Linux.
- **packages.go**: Provides functions to create Ethernet and IP headers and build TCP packets.

## Development

### Adding New Features

1. Modify the C code in `probe.c` to capture additional packet attributes or modify existing logic.
2. Update the Go code to handle the new data or logic.
3. Rebuild the eBPF program and Go binary.

### Testing

Use `go test` to run unit tests for the Go components:

```bash
sudo go test ./...
```