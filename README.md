# Humane Remote Attestation Framework for AWS Nitro Enclaves (Humane-RAFW-NE)

![SemVer](https://img.shields.io/badge/Humane--RAFW--NE-0.1.0-blue)
![MSRV](https://img.shields.io/badge/MSRV-1.90.0-blue)
[![License](https://img.shields.io/badge/License-MIT-red)](/LICENSE)

This repository demonstrates a minimal end-to-end flow for AWS Nitro Enclaves:
**ECDH key exchange → remote attestation → API call via secure channel**.

## Compatibility

### Server (Parent VM)

- **Cloud Platform**: AWS
- **Instance type**: any Nitro Enclaves capable EC2 instance
  - See [Parent instance requirements](https://docs.aws.amazon.com/enclaves/latest/user/nitro-enclave.html#nitro-enclave-reqs)
  - Both x86_64 and AArch64 are supported
- **AMI**: Ubuntu 24.04 / 25.10
  - If you use other Linux distributions, manually setup the parent VM following
    the [Nitro CLI documentation](https://github.com/aws/aws-nitro-enclaves-cli)

### Client

The client code is architecture-independent. Ideally, it should be usable in
any environment. It is currently only verified to work on Ubuntu (x86_64 /
AArch64) and macOS (AArch64).

## Tested environments

Tested on the Parent VM environments listed below. For ease of testing, the
Server (vsock proxy) runs on localhost (`127.0.0.1:8080`) on the Parent VM, and
the Client was also run on the same Parent VM. In a typical deployment, the
Client can run on a different machine and connect to the Proxy over the network.

### Parent VM (AWS EC2 instance)

#### Ubuntu 24.04 (x86_64)

- **Instance type**: `c5.xlarge`
  - vCPUs: 4
  - Memory: 8 GiB
  - CPU arch: x86_64
- **AMI**: Ubuntu Server 24.04 LTS
  - AMI ID: `ami-06e3c045d79fd65d9`
- **Storage**: 64 GiB gp3
- **Kernel**: 6.14.0-1018-aws
- **Nitro Enclaves**: Enabled
- **Nitro Enclaves CLI / driver**: v1.4.4

#### Ubuntu 24.04 (AArch64)

- **Instance type**: `m6g.xlarge`
  - vCPUs: 4
  - Memory: 16 GiB
  - CPU arch: AArch64
- **AMI**: Ubuntu Server 24.04 LTS
  - AMI ID: `ami-01da1dbf9ea3a6ee6`
- **Storage**: 64 GiB gp3
- **Kernel**: 6.14.0-1018-aws
- **Nitro Enclaves**: Enabled
- **Nitro Enclaves CLI / driver**: v1.4.4

### Nitro Enclave (inside the parent VM)

- **OS**: Debian 12
- **Allocated vCPUs**: 2
- **Allocated Memory**: 512 MiB

## Architecture

Humane-RAFW-NE consists of the following components:

- `enclave/` (+ `Dockerfile`): Enclave application (listens on vsock port)
- `proxy/`: untrusted HTTP → vsock proxy (listens on HTTP, forwards to vsock port)
- `client/`: Client app (POSTs JSON to the proxy, verifies attestation, then
  calls the "add two integers" API)

By default, the proxy listens on localhost `127.0.0.1:8080`. See [Configuration](#configuration)
to change this.

For detailed architecture documentation, see [Architecture](docs/architecture.md).

## Quick start

Clone the repository on the Parent VM (and also on the client machine if you
run the client elsewhere):

```bash
git clone https://github.com/acompany-develop/Humane-RAFW-NE
cd Humane-RAFW-NE
```

### Server side

#### 1. Parent VM setup

```bash
make setup-docker
make setup-nitro-cli
```

#### 2. Build the Enclave image

```bash
make build-enclave
```

When you run `make build-enclave`, reference PCR measurements are printed like
this (example):

```text
Enclave Image successfully created.
{
  "Measurements": {
    "HashAlgorithm": "Sha384 { ... }",
    "PCR0": "...",
    "PCR1": "...",
    "PCR2": "..."
  }
}
```

Distribute these reference measurements to the client.

#### 3. Build the vsock proxy

```bash
make build-proxy
```

#### 4. Start the Enclave

```bash
make run-enclave
```

#### 5. Start the vsock proxy

```bash
make run-proxy
```

### Client side

#### 1. Client setup

```bash
make setup-client
```

#### 2. Configure the client app

Download AWS Nitro Enclaves root certificate:

```bash
make download-root-ca
```

This creates `root.pem` in the repository root, which the client uses for
attestation certificate chain verification.

Copy the reference **PCR0/1/2** values into `"PCRs"` in `client-configs.json`.

#### 3. Build the client

```bash
make build-client
```

#### 4. Run the client

```bash
make run-client
```

After ECDH key exchange and attestation verification, the client calls the
Enclave's "add two integers" API and then closes the session.

### Cleanup

#### Terminate the Enclave

```bash
make terminate-enclave
```

#### Delete Docker image

```bash
docker rmi rafwne-enclave
```

## Configuration

### Proxy arguments

| Argument               | Description           | Default     |
| ---------------------- | --------------------- | ----------- |
| `--ip`                 | HTTP server bind IP   | `127.0.0.1` |
| `--port`               | HTTP server port      | `8080`      |
| `--cid`                | Enclave CID           | `16`        |
| `--vsock-port`         | vsock port            | `5000`      |
| `--vsock-buffer-size`  | Buffer size (bytes)   | `8192`      |

### Enclave arguments

| Argument              | Description         | Default |
| --------------------- | ------------------- | ------- |
| `--vsock-port`        | vsock port          | `5000`  |
| `--vsock-buffer-size` | Buffer size (bytes) | `8192`  |

### Client configuration (`client-configs.json`)

| Field                      | Description                                 | Default     |
| -------------------------- | ------------------------------------------- | ----------- |
| `"server-ip"`              | Proxy IP address                            | `127.0.0.1` |
| `"server-port"`            | Proxy port                                  | `8080`      |
| `"PCRs"`                   | Expected PCR0/1/2 values (hex)              | —           |
| `"print-attestation-json"` | Print attestation document as JSON if `true`| `true`      |

For `"PCRs"`, copy the reference measurements printed by `make build-enclave`.
Note that rebuilding the enclave image will change PCR values.

### Enclave allocator configuration (`/etc/nitro_enclaves/allocator.yaml`)

| Field        | Description                         | Default |
| ------------ | ----------------------------------- | ------- |
| `memory_mib` | Allocated memory for enclaves (MiB) | `512`   |
| `cpu_count`  | Reserved CPU count for enclaves     | `2`     |
| `cpu_pool`   | Reserved CPU IDs for enclaves       | —       |

#### Notes

- `cpu_count` conflicts with `cpu_pool`.
- Example `cpu_pool` values: `1,2,3,5-7`.

- To reflect changes, restart the Nitro Enclaves Allocator Service:

  ```bash
  sudo systemctl restart nitro-enclaves-allocator.service
  ```

### Parameters in `Makefile`

| Variable            | Description                    | Default     |
| ------------------- | ------------------------------ | ----------- |
| `ENCLAVE_CID`       | Enclave CID                    | `16`        |
| `ENCLAVE_MEMORY`    | Allocated memory for enclaves  | `512`       |
| `ENCLAVE_CPU_COUNT` | Reserved CPU count for enclaves| `2`         |
| `SERVER_IP`         | HTTP server bind IP            | `127.0.0.1` |
| `SERVER_PORT`       | HTTP server port               | `8080`      |
