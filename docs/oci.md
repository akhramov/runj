*Placeholder for OCI changes*

# `create`

The `create` command is documented [in the
spec](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md#create)
as requiring two positional arguments: `<container-id> <path-to-bundle>`.
However, `runc` (the reference implementation of the spec) accepts only a single
positional argument (`<container-id>`) and instead uses either the current
working directory as the bundle or accepts it through the `-b`/`bundle` flag.
See [here](https://github.com/opencontainers/runc/blob/2cf8d240075dd322b9385100c9af4b149c973391/create.go#L12-L30).

## Non-terminal STDIO

The spec does not describe how container STDIO should be handled.  runc passes
the STDIO file descriptors for the `runc create` invocation to the container
process.  Programs that invoke runc (like containerd) configure STDIO for `runc
create` and provide input/collect output for the container through those
descriptors.

## Terminal STDIO

The spec includes a `terminal` field, but does not describe how a runtime should
expose the terminal.  runc expects an `AF_UNIX` socket to be provided as a
command-line argument in the `--console-socket` field; runc's init process is
responsible for allocating a `pty` and sending the control device's file
descriptor over the provided socket.

runc opens the socket file as part of `runc create`, then passes that socket
file as an extra file descriptor when starting the init process, indicating the
fd number with the `_LIBCONTAINER_CONSOLE` environment variable.  The init
process creates the pty (`console.NewPty`),  sends the control fd number to the
socket (`utils.SendFd`), and uses `dup3(2)` to override the standard I/O file
descriptors (0, 1, and 2) for the container process.

A runc client like `containerd-shim-runc-v2` is responsible for creating the
socket passed as `--console-socket` (with `pkg/process/init.go:Create`),
receiving the control device (with `socket.ReceiveMaster`), then copying bytes
to and from the device.


# `start`

runc's implementation of the start command exits immediately after starting
the container's process.  This does not appear to be specified in the spec.
