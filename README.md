# ENOWARS Specification
This repository bundles the specification for the communication between different parts of the ENOWARS infrastructure.

## ctf.json
The `ctf.json` serves as the central configuration file for the [EnoEngine](https://github.com/enowars/EnoEngine). It contains information about e.g. the round length, the registered teams and the services/checkers being used.

See [ctf.json](ctf_json.md)

## Checker API v2
The checker API specifies the HTTP-based protocol which is used by the engine (and any other tool interacting with the checkers) to send it tasks and receive the result of the task.

See [checker protocol](checker_protocol.md)

## Service & Checker tenets
The service & checker tenets specify the requirements a service, its vulnerabilities and associated checker have to satisfy.

See [service & checker tenets](service_checker_tenets.md)