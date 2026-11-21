# Nauvim — Graphical user interface for Neovim

**Draft specification**

## 1 Scope

This document specifies the architecture and behaviour of Nauvim, a graphical user interface (GUI) for Neovim.

It covers local and remote session management, remote procedure call (RPC) communication with host Neovims, plugin interfaces, client operations, configuration, build-time features and dependencies. It also specifies the initial platform targets and user interaction model.

This document applies to the initial Flutter implementation and the components that are independent of the user interface (UI). Native input method editor (IME) integration is outside its scope.

The configuration interface is specified to the extent stated in Clause 8. Other configuration details remain provisional. Annex A provides informative examples. Annex B identifies matters requiring further specification. Annex C provides informative guidance on OpenSSH configuration inheritance.

## 2 Normative references

The following documents apply to the provisions that explicitly reference them. Their normative use is limited to the Neovim API contract in 5.2, the Neovim UI-option types and meanings in 6.3.1, the client-information interface in 7.2.1, the Neovim paste interface in 7.8, HTTP semantics, HTTP/2 protocol and URL/URI processing in 11.5.5, the Neovim clipboard-provider interface in 11.6, and the `UILeave` event's channel information in 11.7.3.

- Neovim, [Nvim API — API contract](https://neovim.io/doc/user/api/#api-contract), help tag `api-contract`.
- Neovim, [Nvim API — Client information](https://neovim.io/doc/user/api/), help tag `nvim_set_client_info()`.
- Neovim, [Nvim API — Paste](https://neovim.io/doc/user/api/#nvim_paste()), help tag `nvim_paste()`.
- Neovim, [Nvim UI protocol — UI options](https://neovim.io/doc/user/api-ui-events/#ui-option), help tag `ui-option`.
- Neovim, [Providers — Clipboard integration and paste](https://neovim.io/doc/user/provider/), help tags `provider-clipboard`, `g:clipboard`, `provider-paste` and `ui-paste`.
- Neovim, [Automatic commands — UILeave](https://neovim.io/doc/user/autocmd/#UILeave), help tag `UILeave`.
- IETF, [RFC 3986: Uniform Resource Identifier (URI): Generic Syntax](https://www.rfc-editor.org/rfc/rfc3986.html), January 2005.
- IETF, [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html), June 2022.
- IETF, [RFC 9113: HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html), June 2022.

Other supporting technical documents are listed in the Bibliography. The OpenSSH configuration guidance in 8.5.4 and Annex C uses `ssh_config(5)` [22] as an informative reference.

## 3 Terms and definitions

For the purposes of this document, the following terms and definitions apply.

Unless otherwise qualified, **session** denotes a Nauvim session and **attachment** denotes a UI attachment. **Local** and **remote** are relative to the client environment. Connections and channels are qualified by their protocol where necessary.

### 3.1 host Neovim

Neovim process launched with `--headless` that maintains editing state and provides the Neovim RPC application programming interface (API) and UI protocol

### 3.2 host environment

machine or execution environment in which a host Neovim runs, including its filesystem, processes and network environment

### 3.3 client Neovim

**UI client**

process that implements the Neovim UI protocol and provides a user interface for a host Neovim

Note 1 to entry: Nauvim is one client Neovim. Other implementations also exist.

### 3.4 client environment

machine or execution environment in which a client Neovim runs, including its filesystem, processes and network environment

Note 1 to entry: The host environment and client environment can be the same environment.

### 3.5 Nauvim engine

client-side component implemented by `nauvim-core` that manages sessions, communicates with host Neovims and handles Nauvim requests

### 3.6 host alias

name resolved by Nauvim to client-side configuration describing how to reach a host environment and start or locate a host Neovim

### 3.7 Nauvim session

live editing context managed by Nauvim, represented by a host Neovim and the information used to identify and reconnect to it

### 3.8 owning client environment

client environment from which Nauvim started the host Neovim associated with a Nauvim session

### 3.9 session identifier

**session ID**

identifier referring to the same Nauvim session across RPC connections and UI attachments

### 3.10 session multiplexing

management of multiple Nauvim sessions with switching of a client's attachment between them

### 3.11 session file

script containing commands for restoring selected Neovim editing state

Note 1 to entry: Neovim's `:mksession` command produces a session file. Such a file records state for restoration; it does not preserve a running process. See [4].

### 3.12 RPC transport

underlying communication mechanism carrying Neovim RPC messages

EXAMPLE Pipes for standard input and standard output, a Unix domain socket, a Transmission Control Protocol (TCP) connection or a stream carried through Secure Shell (SSH).

### 3.13 RPC connection

live, bidirectional MessagePack-RPC communication path between a client and a host Neovim

Note 1 to entry: An RPC connection can exist before a UI attaches. Nauvim's connection cleanup on detachment is specified in 6.3.

### 3.14 RPC endpoint

listening access point through which an RPC connection can be established, identified by a socket address

EXAMPLE A Unix domain socket path or a TCP address and port.

Note 1 to entry: Forwarding can cause the host-side endpoint and the address used by the client to differ.

### 3.15 Neovim RPC channel

Neovim input/output (I/O) channel operating in RPC mode and representing an RPC communication path within a Neovim process

### 3.16 channel identifier

**channel ID**

integer identifying a channel within a particular Neovim process

Note 1 to entry: A channel ID identifies a communication channel, independently of a session ID or socket address. Neovim channels are described in [3].

### 3.17 Neovim job

process started and managed through Neovim's job-control API

Note 1 to entry: A job ID also identifies the job's I/O channel. A channel accepted through a listening socket has no associated job. See [5].

### 3.18 SSH connection

authenticated, encrypted connection between an SSH client and an SSH server

### 3.19 SSH channel

logical, bidirectional communication path carried within an SSH connection

Note 1 to entry: SSH channels and Neovim channels belong to separate protocol layers.

### 3.20 SSH session

remote execution of a program through an SSH channel of type `session`

Note 1 to entry: The program can be a shell, command or subsystem. An SSH session can operate with or without a pseudo-terminal. See [6].

### 3.21 SSH multiplexing

carriage of multiple SSH channels over a shared SSH connection

### 3.22 disconnection

closure or loss of a connection at a specified protocol layer

### 3.23 reconnection

establishment of a new connection after a previous connection has ended

### 3.24 UI attachment

registration of a client as a UI of a host Neovim on a particular RPC channel

Note 1 to entry: `nvim_ui_attach()` establishes this registration and enables UI events on the channel. See [1] and [2].

### 3.25 attach

establish a UI attachment over an RPC connection

### 3.26 detach

end a UI attachment

Note 1 to entry: UI detachment and connection closure are distinct concepts. Nauvim performs both as specified in 6.3.

### 3.27 reattach

establish a new UI attachment to an existing Nauvim session after a previous attachment has ended

Note 1 to entry: Following Nauvim's detachment procedure in 6.3, reattachment requires a new RPC connection and, for a remote host, a new SSH connection.

### 3.28 RPC request

MessagePack-RPC message invoking a method and expecting a corresponding response containing a result or an error

### 3.29 RPC notification

MessagePack-RPC message invoking a method without expecting an RPC response

Note 1 to entry: The request and notification message types are described in [7].

### 3.30 Nauvim request

application-level request for the Nauvim engine to perform an operation in the client environment

Note 1 to entry: A Nauvim request can be carried by an RPC request or an RPC notification, according to the applicable API.

### 3.31 Nauvim plugin

Neovim plugin loaded by the host Neovim that depends on `nauvim-plugin` to request operations from the Nauvim engine

### 3.32 UI backend

concrete implementation of rendering and GUI functionality using a framework or toolkit

### 3.33 GUI window

window managed by Nauvim as part of the client application's graphical user interface

### 3.34 GUI tab

tab managed by Nauvim as part of the client application's graphical user interface

### 3.35 Neovim window

viewport displaying a Neovim buffer

### 3.36 Neovim tabpage

collection of Neovim windows

Note 1 to entry: Neovim windows and tabpages belong to the host Neovim's editing state. See [8].

### 3.37 client resource

object created or managed by Nauvim in the client environment

EXAMPLE A GUI window, GUI tab, local forward, copied file or directory, or client-side HTTP(S) server.

### 3.38 local forward

forwarding arrangement that accepts connections at a listening endpoint in the client environment and relays them to a destination reached from the host environment

Note 1 to entry: A local forward between TCP ports is also referred to as a local port forward. The `local-forward` interface also supports Unix domain sockets.

Note 2 to entry: For forwarding through SSH, each accepted connection uses a forwarding channel. Forwarding channels are independent of SSH session channels. See [6] and [11].

### 3.39 capability

feature or operation supported by a particular Nauvim implementation or UI backend

### 3.40 security policy

client-side rules governing permission for Nauvim requests and their access to client resources

### 3.41 resource handle

opaque integer value used to refer to an asynchronous operation and the client resources or effects associated with that operation

Note 1 to entry: A resource handle can refer to a resource whose creation is pending, an operation in progress, or a resource remaining after the operation has completed or failed.

Note 2 to entry: A resource handle is distinct from a Neovim channel ID, an RPC message identifier and an operating-system file descriptor.

Note 3 to entry: A handle value can be reused. Its meaning is determined by its current association in the addressed Nauvim process, as specified in 11.2.2.

### 3.42 request error

error returned to the host Neovim through the `err` result of a `nauvim-plugin` function that issues an RPC request

### 3.43 Nauvim error

error reported in the client environment through Nauvim's error-message interface

### 3.44 fairness

scheduling property under which asynchronous work that remains eligible for execution and has not been cancelled is eventually scheduled for execution

### 3.45 serving root

resolved filesystem location designated by `client_file` for the HTTP file-serving interface

Note 1 to entry: A serving root can be a directory or a regular file. Its permitted serving scope is specified in 11.5.5.3.

### 3.46 client identifier

**client ID**

unsigned 32-bit integer associated with a Nauvim process and used for client registration and addressing plugin RPC calls to that process

Note 1 to entry: A client ID is distinct from a session ID, a Neovim channel ID and a resource handle. Its lifecycle is specified in 7.2.1.

### 3.47 process temporary directory

directory in the client environment allocated uniquely to a Nauvim process and used as the base directory for the `client_file` input fields of `filesystem.copy` and `http.serve_file` and the `client_socket_path` input field of `local-forward.open_socket`

Note 1 to entry: Its lifecycle is specified in 6.6.5, and the path-interpretation rules are specified in 11.2.6.

### 3.48 UI implementation identifier

**UI implementation ID**

string identifying a concrete `nauvim` implementation for UI-specific configuration selection

Note 1 to entry: A UI implementation ID identifies an application implementation, rather than an individual running process. It is distinct from the client ID specified in 3.46.

## 4 Architecture

### 4.1 Repository allocation

The project shall comprise the following repositories:

- `nvim-rpc`;
- `nauvim-core`;
- `nauvim-plugin`;
- `nauvim`.

Their responsibilities shall be allocated as specified in 4.2 to 4.5.

### 4.2 `nvim-rpc`

`nvim-rpc` shall provide a general-purpose Rust abstraction of the Neovim RPC protocol.

Its principal entry points shall be the `nvim-rpc-plugin` and `nvim-rpc-ui` crates. The `nvim-rpc-ui` crate shall expose a `Backend` trait for UI backend implementations.

EXAMPLE GTK, Qt, Flutter and Tauri are frameworks or toolkits for which a backend implementation can be provided.

### 4.3 `nauvim-core`

`nauvim-core` shall be a single Rust crate built on `nvim-rpc`.

It shall provide the UI-independent Nauvim engine, including session management, SSH support with SSH multiplexing, and integration with `nauvim-plugin`. Build-time selection of optional functionality shall follow 9.1; the functionality corresponding to `nauvim-plugin::client` shall always be implemented.

For GUI requests, `nauvim-core` shall process the plugin RPC and arrange the requested operation through the UI backend, in accordance with 7.4.

### 4.4 `nauvim-plugin`

`nauvim-plugin` shall be a Neovim plugin implemented in Lua and loaded by the host Neovim.

It shall provide the Lua interface for transmitting Nauvim requests defined by `nauvim-core`. The module shall be accessible using:

```lua
local nauvim = require("nauvim-plugin")
```

### 4.5 `nauvim`

`nauvim` shall contain the concrete application implementation using `nauvim-core` and a UI backend.

It shall determine the concrete methods for creating GUI windows and GUI tabs and shall be responsible for their actual creation and management.

`nauvim` shall provide its UI implementation ID to `nauvim-core` before UI-specific options are selected under 8.10. The ID shall satisfy the string requirements for keys in `ui_opts` specified in 8.10.1. Each implementation shall document its ID.

Subject to those string requirements, the ID's format is unspecified. The ID should follow reverse domain name notation, with domain-name components written in reverse order. See [24], Section 3.

The initial default implementation shall use Flutter.

## 5 Supported environments and component compatibility

### 5.1 Platform targets

The initial platform targets shall be those listed in Table 1. Each target shall be supported as both a host environment and a client environment.

**Table 1 — Initial platform targets**

| Operating system | Architecture | Host environment | Client environment |
| --- | --- | --- | --- |
| GNU/Linux | x86-64 | Supported | Supported |
| GNU/Linux | aarch64 | Supported | Supported |
| macOS | Apple Silicon | Supported | Supported |

### 5.2 Neovim compatibility

Nauvim shall support host Neovim version `v0.12.6`. Behaviour with any Neovim version other than `v0.12.6` is unspecified.

Nauvim's interaction with Neovim shall be implemented in accordance with the Neovim API contract documented at `:h api-contract`.

NOTE Implementing against this contract is expected to allow Nauvim to work with Neovim versions `>= v0.12.6`. This expectation does not extend the specified support requirement beyond `v0.12.6`.

### 5.3 Core and plugin compatibility

The supported combination shall consist of the latest version of `nauvim-core` and the latest version of `nauvim-plugin`. Behaviour with any other combination of their versions is unspecified.

## 6 Session management

### 6.1 Host prerequisites and startup

Operation assumes that Neovim and the program used to launch it are available in the host environment.

Nauvim shall support starting a host Neovim in the client environment. When SSH support is enabled, it shall also support starting a host Neovim in a remote host environment through SSH.

Nauvim shall start the host Neovim with `--headless` and `--listen` followed by the resolved socket setting. The required placement of these arguments shall follow 8.7.2.

If host Neovim startup fails, Nauvim shall apply the connection-failure procedure in 6.9.

### 6.2 Session ownership and reuse

For a requested host, Nauvim shall determine whether an eligible host Neovim is already running.

A host Neovim shall be eligible for reuse only if it was started by Nauvim from the current client environment.

Under the current ownership-identification mechanism, Nauvim shall identify the owning client by checking the RPC socket path in the host environment.

The socket path shall distinguish the client Neovim for the purposes of session ownership. It should be derived from the client's hostname, user identity and any additional information needed to distinguish that client Neovim.

The socket naming scheme shall enable a subsequently started Nauvim process in the same client environment to locate sessions created by a previous process.

When an eligible host Neovim is available for the requested host, Nauvim shall attach to that host Neovim. Otherwise, Nauvim shall start a new host Neovim.

Nauvim shall neither attach to nor modify a Neovim process outside that ownership scope during session discovery and selection.

EXAMPLE A Neovim process started manually by a user, or by Nauvim from another client environment, is outside the ownership scope.

Exact path derivation and the selection rule for multiple eligible sessions are unspecified.

NOTE The socket-path ownership-identification mechanism is provisional and is subject to revision.

#### 6.2.1 Stale RPC socket checks

Before each initial attachment or reattachment, Nauvim shall check whether the selected host Neovim RPC socket in the host environment is stale. The check shall concern the socket selected for the requested session under the ownership rules in 6.2.

For a remote host, Nauvim shall establish the SSH connection before performing this check in the host environment.

The method used to determine whether the socket is stale is unspecified. If the socket is determined to be stale, Nauvim shall delete it before proceeding with attachment or reattachment.

If the staleness check or the required deletion fails, Nauvim shall abandon the attachment attempt, close the associated SSH connection if present, return to the welcome page and display the error. A failed or timed-out staleness check shall not by itself constitute a determination that the socket is stale.

#### 6.2.2 Timeouts

Nauvim shall enforce a timeout for the socket staleness check and for initial attachment and reattachment to a host Neovim. The specific timeout durations are unspecified.

Expiry of the staleness-check timeout shall be treated as a check failure under 6.2.1. Expiry of an attachment or reattachment timeout shall be treated as an attachment failure under 6.3.

### 6.3 Attachment and detachment

Nauvim shall support session multiplexing through attachment, explicit detachment and reattachment. This requirement applies to local sessions and, when SSH support is enabled, remote sessions.

Initial attachment and reattachment shall be subject to the socket-checking and timeout requirements in 6.2.1 and 6.2.2, the UI-option requirements in 6.3.1 and the client-registration requirements in 7.2.

When the attached host Neovim executes `:detach`, Nauvim shall complete the detachment procedure specified in this subclause.

Detachment cleanup shall apply to all client resources associated with the attachment being detached, including HTTP(S) servers and resources whose creation is pending. The task-stopping requirements below shall apply to asynchronous tasks associated with those resources.

As part of explicit detachment, Nauvim shall:

- send a stop signal to every task associated with the attachment being detached, including tasks that have not yet started;
- end the UI attachment;
- ensure that the RPC connection to the detached host Neovim is closed;
- for a remote host Neovim, close the associated SSH connection;
- close all active local forwards, including those opened through the interface in 11.3;
- close all GUI windows opened through `nauvim-plugin`;
- initiate shutdown of all associated HTTP(S) servers and cleanup of all other associated client resources;
- clean up copy operations and their client-side destinations in accordance with 11.4.5;
- remove generated descendants of the process temporary directory that belong to the detached attachment, while retaining that directory itself, in accordance with 6.6.5.

Sending stop signals shall be the minimum task-stopping requirement for detachment, including for tasks implementing HTTP(S) servers. Nauvim should ensure that those tasks terminate and should confirm their termination, but neither guaranteed termination nor confirmation of termination shall be a mandatory condition for completing detachment. The copy-operation requirements in 11.4.5 shall additionally require waiting for termination before deletion. The other resource-closing requirements in this subclause shall remain applicable.

Explicit detachment shall leave the host Neovim running and available for reattachment. It shall not terminate the Nauvim application.

On explicit detachment, Nauvim shall return to the welcome page specified in 6.4. This requirement shall also apply during session switching, with cleanup and connection timing as specified in 6.5.

When the attached host Neovim quits, including as a result of a quit command such as `:q`, Nauvim shall perform the same client-side cleanup and welcome-page transition as explicit detachment. The Nauvim application shall remain running.

Reattachment following this procedure shall establish a new RPC connection and, for a remote host, a new SSH connection.

If UI attachment or reattachment fails, including by timeout, Nauvim shall apply the connection-failure procedure in 6.9. This requirement shall also apply when the failed attachment is part of a session switch under 6.5.

#### 6.3.1 UI options at attachment

On every initial UI attachment and reattachment, `nauvim-core` shall enable all UI capability options that Neovim does not designate as legacy or deprecated.

Each UI option shall be supplied in accordance with its defined type and meaning. See [2].

NOTE UI options include capability switches and options that convey descriptive values. The requirement to enable UI capabilities does not mean assigning the Boolean value `true` to every option.

### 6.4 Welcome page and automatic connection

Nauvim shall provide a welcome page for selecting a connection destination.

From the welcome page, the user shall be able to start a new session or reattach to an eligible existing session.

After attachment, Nauvim shall display the host Neovim's UI.

At application startup, Nauvim shall use `hosts` together with `default_host` to select between automatic connection and waiting on the welcome page, as specified in 8.2.1. When that procedure requires user selection, the available choices and their presentation shall follow that subclause. The concrete host-selection interaction is otherwise unspecified.

Automatic connection under 8.2.1 shall be a startup action. The welcome-page transitions required for detachment, host Neovim termination and error recovery shall remain governed by 6.3, 6.8 and 6.9. A session switch shall proceed to its requested host under 6.5.

### 6.5 Plugin-initiated session switching

`nauvim-plugin` shall provide `nauvim-plugin::client.connect({ host: string })` as specified in 11.7.4. The `host` field shall be a host alias.

Subject to the no-effect rule in 7.3, the function shall send an RPC notification using the stored channel ID, with method name `nauvim-plugin/client.connect` and the parameter list `{ client_id, host }`. Immediately after successfully issuing the notification, `nauvim-plugin` shall execute detachment in the current host Neovim, for example through `:detach` or an equivalent operation. It shall not wait for a response or acknowledgment from Nauvim before initiating that detachment.

If issuing the notification fails, the function shall display an error message and perform no further operation for that call. In particular, it shall not initiate host-side detachment.

The notification received by `nauvim-core` shall be subject to 11.2.5. After accepting the notification, Nauvim shall retain the requested host as the destination for the session switch and perform the detachment cleanup required by 6.3 for the attachment being left.

Nauvim shall return to the welcome page and then begin connecting to the requested host without waiting for all detachment cleanup to finish. Before beginning that connection attempt, Nauvim shall complete state initialization and any other implementation-required prerequisite operations. Completion of the remaining cleanup shall not be a prerequisite for starting the new connection.

The new connection shall start or reuse the requested host Neovim in accordance with 6.1 and 6.2 and establish its attachment in accordance with 6.3.

Cleanup shall remain associated with the attachment being left. The copy-stopping and deletion order in 11.4.5 shall continue to apply within that cleanup; it shall not by itself require completion of all cleanup before the new connection attempt. Cleanup may continue while the new connection is being established or used, subject to the prerequisite operations specified above. Interference between copy destinations and cleanup operations shall be subject to 11.4.6.

Nauvim shall resolve the host alias using client-side configuration. The host Neovim shall delegate host-alias resolution and client-side connection management to Nauvim.

EXAMPLE Calling `require("nauvim-plugin.client").connect({ host = "another-host" })` requests a switch to the host selected by the client-side alias `"another-host"`.

NOTE The `host` field is a Nauvim host alias. Its interpretation is defined by this interface rather than inherited from Neovim's `:connect` command.

### 6.6 Nauvim process termination and reopening

Termination of the Nauvim process shall not terminate host Neovim processes associated with its managed sessions. This requirement applies to both local and remote sessions, including reattached sessions and termination by killing the Nauvim process.

A subsequently started Nauvim process in the same client environment shall be able to discover and reattach to eligible host Neovim processes that remain running. Session discovery and ownership identification shall follow 6.2.

#### 6.6.1 Shutdown triggers

Nauvim shall initiate and perform shutdown cleanup when either of the following occurs:

- all of its GUI windows have been closed;
- the Nauvim process receives `SIGTERM`.

The precise meaning of a GUI window being closed is UI-toolkit-dependent. It shall correspond to the ordinary user-facing meaning of closing a window.

#### 6.6.2 Cleanup responsibilities

Shutdown cleanup shall include:

- sending stop signals to running asynchronous tasks in accordance with 6.6.3;
- safely closing all RPC connections managed by Nauvim;
- safely closing all SSH connections managed by Nauvim;
- deleting plugin temporary files;
- deleting SSH-related temporary files;
- removing the process temporary directory and its remaining contents in accordance with 6.6.5;
- terminating Nauvim's child processes, subject to the host Neovim preservation requirement in this subclause;
- terminating the Nauvim process.

The requirement to terminate child processes shall not apply to host Neovim processes preserved under 6.6, including a local host Neovim that is a child process of Nauvim.

Cleanup shall preserve the ability to discover and reattach to those host Neovim processes in accordance with 6.2 and 6.6. In particular, cleanup shall not remove a host Neovim's RPC socket while it is needed for subsequent discovery and reattachment.

#### 6.6.3 Running asynchronous tasks

Nauvim shall send a stop signal to each running asynchronous task during shutdown cleanup.

Nauvim should ensure that tasks receiving a stop signal terminate, and should confirm their termination.

Sending the stop signal is mandatory. A guarantee of task termination and confirmation that a task has terminated are recommended, but are not mandatory conditions for shutdown cleanup or process termination.

#### 6.6.4 Graceful shutdown

During shutdown cleanup, Nauvim should perform graceful shutdown by ceasing to accept new RPC calls from host Neovim processes.

Nauvim should also cease accepting new connections through `accept(2)` on local ports and sockets used for WebView-related functionality and local forwarding.

#### 6.6.5 Process temporary directory

Each Nauvim process shall have its own temporary directory in the client environment. The directory shall be unique to that process and shall provide the common base directory for the interfaces specified in 11.2.6.

Detachment shall retain the process temporary directory itself. Descendant files and directories generated by operations associated with the detached attachment shall be removed in accordance with the applicable cleanup rules, including 11.4.5 for copy operations. The directory itself shall also be retained during session switching and connection recovery while the Nauvim process continues running.

During shutdown cleanup, Nauvim shall remove the process temporary directory and all remaining contents, subject to the deletion-failure rule in 6.6.6.

The directory's location, storage medium and filesystem, including whether it is backed by `tmpfs`, are implementation-dependent. Whether the operating system removes the directory during system shutdown or boot is also implementation-dependent.

#### 6.6.6 Shutdown deletion failures and scope of cleanup

If deletion fails during shutdown cleanup, Nauvim shall display an error message and proceed to process termination without retrying or resuming the failed deletion.

The file-deletion requirements specify the effects of cleanup when cleanup is executed. If termination prevents cleanup from executing, those requirements do not establish a guarantee that the remaining files will eventually be deleted. They neither prohibit deletion by other mechanisms nor require the implementation to provide such mechanisms.

### 6.7 Concurrent Nauvim instances

The defined operating configuration comprises at most one running Nauvim instance in each client environment.

Starting additional concurrent instances can succeed. Behaviour with multiple concurrent Nauvim instances in one client environment is undefined.

### 6.8 Unexpected connection interruption

#### 6.8.1 Connections in scope

The recovery procedure in 6.8.2 shall apply when an unexpected error interrupts either of the following:

- the RPC connection carrying Nauvim's UI attachment to a host Neovim;
- an SSH connection managed by Nauvim, other than a plugin-specific SSH connection.

Connection closure caused by the attached host Neovim quitting shall be handled under 6.3.

Interruption of a separate plugin-only RPC connection shall not trigger this procedure. An RPC connection carrying the UI attachment shall remain in scope even if it also carries plugin traffic.

An SSH connection serving the host session shall remain in scope even if it also carries plugin traffic.

The exclusions in this subclause concern the events that trigger recovery. They shall not exempt plugin-related resources from cleanup when an in-scope interruption has occurred.

#### 6.8.2 Recovery and cleanup

Following an interruption specified in 6.8.1, Nauvim shall perform connection-recovery cleanup, return to the welcome page and display the error. Nauvim shall remain running.

Nauvim shall not send a detachment request or notification to the host Neovim as part of this recovery procedure.

Connection-recovery cleanup shall apply the resource-cleanup requirements in 6.6.2 to resources associated with the affected connection or session. This includes safely closing associated RPC and SSH connections, deleting associated plugin and SSH-related temporary files, and terminating associated child processes subject to the host Neovim preservation requirement. The Nauvim process and resources needed to display and operate the welcome page shall be retained.

Connection recovery shall retain the process temporary directory itself in accordance with 6.6.5. Cleanup of associated descendant files shall not remove that directory.

The requirements and recommendations for stopping asynchronous tasks in 6.6.3 shall apply to the tasks associated with this cleanup. Stop signals are mandatory; guaranteed termination and confirmation of termination remain recommendations.

The graceful-shutdown recommendations in 6.6.4 shall also apply to the connections and listeners being cleaned up.

Recovery cleanup shall not terminate a surviving host Neovim or remove a non-stale RPC socket needed for subsequent discovery and reattachment. Stale-socket checking and deletion before a later attachment attempt shall follow 6.2.1.

### 6.9 Errors before UI attachment

If an error occurs during a connection attempt before UI attachment is complete, Nauvim shall display the error, return to the welcome page and perform cleanup as necessary. This rule shall apply to initial connections, reattachments and session switches, including failures of host resolution, authentication, host Neovim startup and UI attachment.

Where cleanup is needed, Nauvim shall apply the relevant resource-cleanup requirements of this document to the failed attempt. The Nauvim process and the resources required to operate the welcome page shall be retained. More specific requirements for a failure, including the socket-checking requirements in 6.2.1 and 6.2.2, shall remain applicable.

The non-fatal client-registration and host-side setup failures specified in 7.2.1 and 7.2.2 shall be exceptions to this procedure and shall be handled as specified in those subclauses.

## 7 Plugin interface and client operations

### 7.1 Execution model

A Nauvim plugin shall execute as a Neovim plugin loaded by the host Neovim.

The plugin can perform operations in the host environment through the facilities available to Neovim plugins. For Nauvim operations in the client environment, it shall transmit Nauvim requests through `nauvim-plugin`.

`nauvim-plugin` shall use Neovim RPC requests or notifications, as specified by the applicable Nauvim API.

The detailed API contracts in Clause 11 shall apply when the corresponding functionality is enabled.

### 7.2 RPC destination registration

Nauvim shall call `nvim_set_client_info` during each initial UI attachment and reattachment and initiate host-side setup as specified in 7.2.1 and 7.2.2.

`nauvim-plugin` shall use the single channel ID selected under 11.7 as the destination for RPC messages to the Nauvim engine and the associated stored client ID as their first argument. Calls made while either stored identifier is `nil` shall be subject to 7.3.

#### 7.2.1 Process client identifier and client information

Each Nauvim process shall have one client ID, generated randomly at process startup. The client ID shall be an unsigned 32-bit integer, from `0` to `4294967295` inclusive. It shall remain unchanged for the lifetime of the process, including across initial attachments, reattachments and connections to different hosts.

Behaviour is undefined if distinct Nauvim process instances are assigned the same client ID, whether their lifetimes overlap or occur at different times. Neither Nauvim nor `nauvim-plugin` shall perform any checks to determine whether a generated client ID collides with a client ID assigned to another process instance.

The generation method should produce a sufficiently uniform random distribution over the full unsigned 32-bit range, with the design objective of avoiding accidental collisions during an individual user's lifetime of Nauvim use. This is a strong recommendation; it does not establish a guarantee that collisions cannot occur.

Unpredictability is not required. The specific generation method is unspecified.

The checks of incoming RPC client-ID arguments specified in 11.2.5 shall remain required. Those checks compare an argument with the receiving process's own client ID; they do not determine whether distinct process instances have been assigned the same ID.

Nauvim shall call `nvim_set_client_info` over the RPC connection used for its UI attachment, with the values specified in Table 2. The parameter types and meanings shall follow Neovim's client-information interface. See [1].

**Table 2 — Client registration values**

| Parameter | Required value |
| --- | --- |
| `name` | `"nauvim-client-{client_id}"`, using the process client ID and the representation specified below |
| `version` | A dictionary containing the `major`, `minor` and `patch` version numbers of `nauvim-core` |
| `type` | `"ui"` |
| `methods` | An empty dictionary, except for entries explicitly required by this document for this parameter |
| `attributes` | A dictionary containing only `license`, whose value describes the licence of `nauvim-core` |

In the client name, `{client_id}` shall be replaced by the unsigned decimal representation of the client ID, using ASCII digits without leading zeros except for the value `0` itself.

If `nvim_set_client_info` fails, Nauvim shall ignore that failure and continue normal GUI and editor operation. The failure shall not by itself cause an error message, recovery or termination of UI attachment. The client ID and channel ID stored by `nauvim-plugin` shall both remain `nil`, and plugin calls shall follow 7.3.

NOTE 1 The client type `"ui"` identifies a GUI frontend in Neovim's client-information interface.

NOTE 2 Listing an RPC method elsewhere in this document does not itself require an entry in the `methods` parameter.

#### 7.2.2 Asynchronous host-side setup

After successfully calling `nvim_set_client_info`, Nauvim shall asynchronously invoke `require("nauvim-plugin.client").setup({ client_id = client_id })` in the host Neovim using `nvim_exec2` or an equivalent execution RPC. The setup invocation shall execute when Nauvim is attached as a UI. The fair-scheduling requirement in 11.2.2 shall apply.

The `client_id` field shall contain the process client ID specified in 7.2.1. Nauvim shall embed this field's value as an integer literal in the execution source it constructs in the client environment and sends to the host Neovim. This requirement shall also apply when an equivalent execution RPC is used. The host-side plugin shall construct the client name from this integer as specified in 11.7.1.

A failure to load or execute the host-side setup, including absence of the `nauvim-plugin.client` module or failure to select a matching client, shall be ignored by `nauvim-core` and `nauvim-plugin`. Neither component shall raise or report an error, display a message or initiate recovery solely because of that failure.

Setup failure shall not by itself prevent or terminate UI attachment or affect normal editor operation. Nauvim shall continue to function as a Neovim GUI without an available `nauvim-plugin` RPC destination. Plugin calls that require that destination shall follow 7.3. The stored identifiers following an unsuccessful setup invocation shall be governed by 11.7.1 and 11.7.2.

### 7.3 Operation without a selected RPC destination

`nauvim-plugin` shall remain usable when Nauvim is not attached.

For every `nauvim-plugin` function other than `nauvim-plugin::client.setup` and the clipboard-provider `paste` callbacks specified in 11.6.2, if the stored RPC channel ID or the stored client ID specified in 11.7.2 is `nil` when the function is called, the function shall perform no operation.

In this case, the function shall not issue an RPC request or notification, raise or return an error, or display any message. It shall return without waiting for `setup` to be called, and the call shall not be deferred for execution after setup.

Functions using the `ret, err` convention in 11.2.1 shall return `nil, nil`. Other functions subject to this subclause, including functions that normally send notifications and clipboard-provider `copy` callbacks, shall return `nil`.

This no-effect rule shall take precedence over method-specific validation and operation requirements. It shall apply to functions that normally issue RPC requests or notifications and to clipboard-provider `copy` callbacks. The clipboard-provider `paste` callbacks shall retain the behaviour specified in 11.6.2 regardless of whether an RPC destination is selected.

### 7.4 GUI operations

When HTTP plugin functionality is enabled, `nauvim-plugin` shall provide the `http.open_webview` interface specified in 11.5.3 and shall transmit the corresponding RPC request to `nauvim-core`.

`nauvim-core` shall process that request and arrange creation of the requested GUI window through the UI backend. `nauvim` shall determine the concrete creation method and shall be responsible for actual GUI window creation.

`nauvim` shall be responsible for creating and managing GUI windows and GUI tabs. GUI windows and GUI tabs shall be distinguished from Neovim windows and Neovim tabpages.

EXAMPLE 1 A plugin opens an overlay window above an existing GUI window to inspect a preview of HTML, TeX/LaTeX or Markdown content.

EXAMPLE 2 A plugin opens a WebView or displays an image in the client environment.

### 7.5 Local forwarding

When local-forwarding functionality is enabled, Nauvim shall provide the interface specified in 11.3 for forwarding between TCP ports and between Unix domain sockets.

The interface shall support local host environments and, when SSH support is enabled, remote host environments. The SSH channel types used for forwarding through SSH are specified in 11.3.

EXAMPLE A web server runs in a Docker container with its service published on host port `8080`. A Nauvim plugin requests forwarding from client port `8081` to host port `8080`. A browser in the client environment, such as Firefox, accesses the service at `http://localhost:8081/`.

### 7.6 Client-side security

Nauvim should restrict operations delegated by host Neovims using sandboxing, configurable security policies or both.

The restriction mechanism should apply to access to client resources, including GUI operations and local port forwarding.

### 7.7 Client-side files and HTTP(S) services

When the corresponding functionality is enabled, Nauvim shall provide the filesystem interface in 11.4 and the HTTP interface in 11.5.

The filesystem interface copies files from the host environment to the client environment. The HTTP interface serves client-side files and opens GUI windows that access HTTP(S) URLs.

The `client_file` input fields of `filesystem.copy` and `http.serve_file` shall share the process temporary directory as their path base, in accordance with 11.2.6.

EXAMPLE A Nauvim plugin requests a client-side HTTP(S) server to serve HTML and JavaScript assets for an interactive user interface. The server implementation can use `hyper` for HTTP handling.

### 7.8 Clipboard integration

When clipboard functionality is enabled, `nauvim-core` shall support copying text from the host Neovim to the clipboard backend in the client environment and pasting text obtained from that backend into the host Neovim.

For copying, `nauvim-plugin` shall provide the `clipboard.copy` function and clipboard provider specified in 11.6. On receiving a valid copy notification, `nauvim-core` shall pass its `text` argument to the selected clipboard backend. How `nauvim-core` uses the `register` argument is unspecified.

For GUI paste operations, `nauvim-core` shall obtain text from the selected clipboard backend and send that text to the host Neovim using `nvim_paste()` or an equivalent operation that delivers the text as paste input. See [1] and [18].

The GUI paste operation shall not obtain text through the clipboard provider's `paste` callbacks. Those callbacks shall have the behaviour specified in 11.6.2.

The concrete client-side interfaces between `nauvim`, `nauvim-core` and clipboard backends are unspecified.

Additional dependencies for clipboard integration shall be subject to the client and host allocation in 9.2. Client-side clipboard backend selection, availability validation and runtime error reporting shall follow 8.8.

## 8 Configuration

### 8.1 Configuration source and execution environment

Nauvim shall permit the user to specify the path of its Lua configuration file. The mechanism by which the user specifies that path is unspecified.

When the user does not specify a path, the default shall be `$XDG_CONFIG_HOME/nauvim/init.lua` in the client environment.

The configuration shall be evaluated by the Nauvim engine in the client environment.

Before executing the configuration file, `nauvim-core` shall obtain the client-side `HostInfo` specified in 8.6 and inject the `nauvim-client-info` module into the configuration's Lua environment. `require("nauvim-client-info")` shall return that `HostInfo` table. The acquisition and failure rules in 8.6.3 shall apply before execution of the configuration file begins.

#### 8.1.1 Lua module search path

When loading the Lua configuration file, `nauvim-core` shall configure the Lua module search path before executing the configuration's Lua code.

Let `config_file` be the non-nil, symbolic-link-resolved absolute path supplied by the client-side `HostInfo`. The module search root `ROOT` shall be the parent directory of that path, irrespective of the configuration file's name:

```text
ROOT = config_file.parent
```

`nauvim-core` shall prepend the following templates to `package.path`, in the order shown:

```text
ROOT/lua/?.lua;ROOT/lua/?/init.lua
```

The existing `package.path` entries shall be retained after these templates. The added templates shall have the highest priority within the Lua-file search performed through `package.path`.

For a module name such as `foo.bar`, these templates shall search `ROOT/lua/foo/bar.lua` and then `ROOT/lua/foo/bar/init.lua`, with dots in the module name interpreted as directory separators.

The search-path additions specified in this subclause shall be limited to these local Lua-module templates. `nauvim-core` shall not add search paths for Neovim-specific libraries or other Neovim runtime directories.

NOTE 1 Lua module-file search paths contain filename templates separated by semicolons. See [9].

NOTE 2 These templates follow Neovim's local Lua-module layout and search order. See [21].

#### 8.1.2 Configuration-loading failures

If loading the Nauvim Lua configuration fails, Nauvim shall display the error and continue using the default configuration values. This rule shall apply to failures in reading, parsing, executing or validating that configuration, and to client-information acquisition failures that prevent its execution under 8.6.3.

The fallback shall use the Nauvim default configuration. Defaults expressly specified in this document shall apply; the remaining default values shall be specified separately. Where this document specifies a field-specific fallback or recovery rule, that rule shall apply.

Failures in obtaining required OpenSSH-source settings during host resolution shall instead follow 8.5.4 and the connection-failure procedure in 6.9.

### 8.2 Configuration scope

The configuration interface shall provide settings for:

- host selection and automatic connection;
- SSH connections;
- the Neovim program specification and Neovim configuration files;
- client-side security policies;
- the client-side clipboard backend and its timeout setting;
- UI-specific application settings, including fonts.

The intended configuration scope is limited to Nauvim application settings. An extensive editor-configuration interface comparable to Neovim's is outside the scope of this draft.

Except within UI-specific option dictionaries (8.10), all predefined field names in the Lua configuration shall use snake_case. Alternative kebab-case spellings of those names shall not be accepted. This naming requirement shall not constrain user-defined host aliases or keys in `ui_opts`.

Nauvim shall provide default configuration values. The native SSH host-setting defaults are specified in 8.4.2. Defaults not otherwise specified in this document remain provisional.

#### 8.2.1 Startup host selection

`default_host` shall be a top-level configuration field of type `string | nil`, at the same level as `hosts`. Its default value shall be `nil`.

For host selection, `hosts` shall designate an empty host collection when it is omitted, nil or an empty dictionary.

In this subclause, the **default Host** shall mean a host specification using the defaults of `Host`, including the program and socket settings in 8.7. It shall designate a local host Neovim without SSH settings.

An empty string supplied as `default_host` shall be a configuration-validation error, irrespective of whether `hosts` is empty. That error shall be handled under 8.1.2.

For a configuration that passes validation, Nauvim shall select its startup destination as follows:

1. If `hosts` is empty and `default_host` is nil, Nauvim shall select the default Host as its automatic startup destination and initiate a connection to it.
2. If `hosts` is empty and `default_host` is a non-empty string `X`, Nauvim shall obtain the SSH settings for `X` from the OpenSSH configuration source under 8.5 and initiate a connection using those settings and the applicable defaults for the host Neovim.
3. If `hosts` is non-empty and `default_host` is nil, Nauvim shall display the welcome page and wait for the user to select a host. The choices shall comprise every `Host` entry in `hosts` and the default Host. Nauvim shall initiate the selected connection after the user makes that selection.
4. If `hosts` is non-empty and `default_host` is a non-empty string `X`, Nauvim shall first look for `hosts[X]`. If that entry exists, Nauvim shall initiate a connection using it, applying 8.3 and 8.5. Otherwise, Nauvim shall obtain the SSH settings for `X` from the OpenSSH configuration source under 8.5 and initiate a connection using those settings and the applicable defaults for the host Neovim.

The default Host shall be an independent choice from the named entries in `hosts`. Its presentation should distinguish it clearly from those entries, including user-defined entries named `"default"` or `"local"`. Those names shall retain their ordinary meaning as host keys; this selection rule shall not reserve either name. The concrete selection interaction and display labels are otherwise unspecified.

Connection to a selected destination shall follow the startup, ownership, reuse and attachment requirements in Clause 6. OpenSSH-source failures shall follow 8.5.4 and 6.9.

#### 8.2.2 Configuration structure

The top-level configuration fields shall have the following types:

```text
Config = {
    clipboard: string | Clipboard | nil,
    default_host: string | nil,
    hosts: dict[string, Host] | nil,
    ssh: SshConf,
    ui_opts: dict[string, dict[string, object]] | nil,
    use_ui_opts: string | nil,
}

config: Config
```

The requirements for `Host`, `Clipboard` and `SshConf` shall be as specified in 8.3, 8.8 and 8.9, respectively. UI-specific configuration shall follow 8.10. Startup host selection shall follow 8.2.1.

### 8.3 Host configuration and local host designation

`Host` shall have the following type:

```text
Host = {
    nvim: string | NvimProg | (function(HostInfo) -> (string | NvimProg)) | nil,
    policies: table | nil,
    ssh: SshHostConf | nil,
    socket: string | (function(HostInfo) -> string) | nil,
}
```

`SshHostConf` shall follow 8.4. The types, resolution and defaults for `nvim` and `socket` shall follow 8.7.

The `policies` field shall belong to the individual host entry. Its table structure, setting meanings and default policy remain to be specified.

A configured host entry without SSH settings shall designate a host Neovim running in the client environment.

Specifically, if `config.hosts[X]` exists and `config.hosts[X].ssh == nil`, Nauvim shall select a local host Neovim. This condition shall be evaluated before applying the SSH host-resolution procedure in 8.5 to a requested host.

For lookup of an explicitly supplied host name `X`, the absence of `config.hosts[X]` shall not designate a local host. In that case, Nauvim shall resolve `X` through the OpenSSH configuration source in accordance with 8.5. Selection of the default Host under 8.2.1 shall directly select its default local specification and shall not constitute lookup of an absent host name.

### 8.4 Native SSH host settings

#### 8.4.1 Structure, names and validation

The SSH settings for a configured Nauvim host `X` shall be supplied by the Lua table `config.hosts[X].ssh`. In this clause, a **native SSH host table** means this table or an inline jump-host table specified in 8.4.6.

The native host-table type `SshHostConf` shall be:

```text
SshHostConf = {
    host_name: string | nil,
    user: string | nil,
    port: int | string | nil,
    identity_file: string | nil,
    certificate_file: string | nil,
    identities_only: string | bool | nil,
    identity_agent: string | nil,
    proxy_jump: string | SshHostConf | list[string | SshHostConf] | nil,
    kex_algorithms: string | list[string] | nil,
    host_key_algorithms: string | list[string] | nil,
}
```

For this type, `nil` shall mean that a field is absent. Value restrictions and defaults shall follow 8.4.2 to 8.4.6. For named hosts, host resolution and merging shall follow 8.5. In particular, a string-valued `port` shall be a decimal string as specified in Table 3.

This subclause specifies the Lua configuration input format. It does not specify the internal representation of SSH settings or restrict the value forms available in an OpenSSH configuration file.

Native host names shall be matched exactly and case-sensitively. The characters `*`, `?` and `!` in these names shall be treated as literal characters. Native host names shall not be case-folded.

The accepted Lua SSH field names shall be exactly those in Table 3. Native SSH host tables shall not support an `Include` directive.

Unknown SSH fields and invalid field types shall be configuration errors, including when they occur in an unused host or an unused inline jump-host table. These checks shall apply to the SSH tables, not to surrounding Nauvim host settings such as `nvim`, `socket` or `policies`.

Lua expressions may compute field values. The resulting values shall conform to the specified field types; the ability to define a Lua function shall not, by itself, make a function a valid SSH field value.

#### 8.4.2 Fields and defaults

Table 3 specifies the accepted native SSH fields and their defaults. A native default shall apply only when host resolution has not supplied a value for the setting. The distinction between an absent field and a defaulted field shall be preserved during merging as specified in 8.5.2 and 8.5.5.

**Table 3 — Native SSH host settings**

| Lua field | Accepted value | Native default |
| --- | --- | --- |
| `host_name` | String | Supplied host name |
| `user` | String | Current operating-system user in the client environment |
| `port` | Integer or decimal string representing an integer from 0 to 65535, inclusive | `22` |
| `identity_file` | Absolute path string for one identity file | No configured identity file |
| `certificate_file` | Absolute path string for one SSH user certificate | None |
| `identity_agent` | Absolute path string for an authentication-agent socket | No authentication agent |
| `identities_only` | Boolean or boolean string specified in 8.4.3 | `false` |
| `proxy_jump` | Host-alias string, inline jump-host table, or sequence containing either | No jump hosts |
| `kex_algorithms` | Ordered comma-separated string or array of algorithm names | Transport defaults |
| `host_key_algorithms` | Ordered comma-separated string or array of host-key algorithm names | Transport defaults with known-host preferences |

#### 8.4.3 Boolean values

The `identities_only` field shall accept the booleans `true` and `false`, and the case-insensitive strings `"true"`, `"false"`, `"yes"` and `"no"`. The strings `"true"` and `"yes"` shall represent true; `"false"` and `"no"` shall represent false.

The `identities_only` setting shall filter the identities used for SSH user authentication. It shall not, by itself, disable the authentication agent.

#### 8.4.4 Paths and authentication agents

Native `identity_file`, `certificate_file` and `identity_agent` values shall be non-empty absolute paths in the client environment. A relative or empty path shall be a configuration error. Configuration loading shall not check the existence or key format of the files designated by these fields.

Lua strings supplied for `identity_file`, `certificate_file` and `identity_agent` shall not undergo tilde expansion or environment-variable-reference expansion. A Lua expression may explicitly obtain such information and construct an absolute path.

EXAMPLE `identity_agent = os.getenv("SSH_AUTH_SOCK")` explicitly obtains an agent-socket value during Lua evaluation.

The native value `identity_agent = "none"` shall be rejected because it is not an absolute path. Native Lua configuration shall not implicitly read `SSH_AUTH_SOCK`. When no value for the agent is supplied by host resolution, the native default shall be to use no authentication agent.

These rules apply to native Lua input. The interpretation of settings obtained from the OpenSSH configuration source is governed by 8.5.4.

#### 8.4.5 Algorithm preferences

The order of algorithms in a comma-separated string or an array shall be preserved.

Without a modifier, the supplied list shall replace the transport's built-in list. A modifier on the first algorithm shall have the following effect relative to the transport's built-in defaults:

- `+` shall append the supplied algorithms;
- `-` shall remove the named algorithms;
- `^` shall prepend the supplied algorithms.

Modifiers shall use the transport defaults as their base, not an algorithm list overridden by the host merge. The transport-default certificate-algorithm list shall be empty.

Unknown algorithm names in replacement, append or prepend lists shall be errors. Removal shall ignore names absent from the transport defaults. Wildcards, empty algorithm lists and empty resulting algorithm lists shall be errors.

Host-certificate algorithm names, such as `ssh-ed25519-cert-v01@openssh.com`, shall be supported. Every certificate algorithm shall precede every plain host-key algorithm in a host-key list. A supplied list or modifier result that violates this order shall be rejected.

An explicitly configured host-key algorithm list, after any modifier is applied, shall be used without known-host reordering or additional algorithms or certificates. When the setting is absent, known-host preference behaviour shall be retained.

Each jump host, including an inline jump host, shall have its own algorithm policy. KEX extension signals shall be managed by the SSH client and shall not be supplied as algorithm names in these settings.

#### 8.4.6 Jump hosts

The `proxy_jump` setting shall accept a host-alias string, an inline host table, or an ordered sequence containing strings and inline host tables.

A string shall designate an SSH host alias resolved using 8.5. Each bastion shall use the same host-lookup and supplementation rules as other SSH destinations. Native SSH settings shall take precedence; supplementation from the OpenSSH configuration source shall follow 8.5.4 before native defaults are applied.

An inline host table shall accept the same SSH fields as a named native host table, including a nested `proxy_jump`. An inline table's `host_name` shall designate a literal network destination. For an inline host with lookup name `Y`, Nauvim shall use the inline table as `B` and apply steps 3 to 5 of 8.5.3. Its explicitly supplied values shall take precedence over the native settings in `H[Y]` and the OpenSSH settings in `O(Y)`.

The lookup name for an inline host shall be its supplied `host_name`, subject to the applicable host-name default. The resolved settings shall provide the destination. If a host name required for resolution cannot be obtained, the ordinary host-resolution failure rules in 8.5.4 and 6.9 shall apply.

An empty sequence, written as `{}` in Lua, shall select no jump hosts. The native string `"none"` shall be treated as a literal host alias, not as a keyword that disables jump hosts.

Only the first jump host's nested `proxy_jump` shall add ancestor jump hosts.

The user shall be responsible for ensuring that proxy-jump definitions contain no cycles. `nauvim-core` shall not check those definitions for cycles. Any error encountered while using such definitions shall be handled under the otherwise applicable requirements of this document; no cycle-specific detection or failure behaviour is specified.

### 8.5 SSH host resolution

#### 8.5.1 Notation and scope

In this subclause, `config` denotes the evaluated Nauvim configuration. The logical SSH host-table collection `H` shall be defined by:

```text
H[X] = config.hosts[X].ssh, if config.hosts[X] exists;
H[X] = nil, otherwise.
```

If `config.hosts` is omitted or nil, `H[X]` shall be nil for every `X`.

Thus, `H[X]` is the SSH table within a Nauvim host entry. It is not the surrounding Nauvim host entry. The merge specified below shall apply only to SSH settings. It shall not inherit another host entry's `nvim`, `socket` or `policies` setting.

`O(X)` denotes the SSH settings produced by the implementation's processing of the client-side OpenSSH configuration source selected by `config.ssh.config` for the supplied host name `X`, under 8.5.4.

The local-host selection rule in 8.3 shall precede this procedure when Nauvim selects a host Neovim. SSH jump-host resolution shall use the SSH collection `H`.

#### 8.5.2 Native table merge

For native Lua SSH tables, merging `A` into `B` shall use `A` as defaults and give precedence to the fields supplied by `B`. For each setting `k`, the merged result `R` shall be:

```text
R[k] = B[k], if B supplies k;
R[k] = A[k], otherwise.
```

For native Lua tables, a `nil` value shall mean that the field is absent. The values `false`, `0` and an empty sequence shall be treated as supplied values.

The merge shall be shallow and shall operate only on the immediate fields of `A` and `B`. It shall not recursively merge nested tables. A supplied value in `B` shall replace the corresponding value in `A` as a whole, irrespective of whether either value is a scalar or a table.

In particular, the merge shall not combine jump-host or algorithm lists, or recursively merge inline jump-host tables.

The `host_name` field shall participate in this merge in the same way as the other fields. If `B` supplies `host_name`, the value shall remain effective even if `A` supplies a different destination name.

Supplementation from the OpenSSH configuration source is governed by 8.5.4.

#### 8.5.3 Resolution procedure

For an SSH destination, after applying 8.3 where applicable, Nauvim shall resolve the requested SSH host name `X` as follows:

1. If `H[X] == nil`, obtain and return `O(X)`. If that operation fails, return the error as specified in 8.5.4. The procedure shall then terminate; steps 2 to 5 shall not apply.
2. Otherwise, let `B` be `H[X]` and let `Y` be its supplied `host_name`. If `host_name` is absent, use `X` as `Y`, in accordance with the supplied-host-name default in Table 3.
3. Let `R` initially contain the explicitly supplied settings in `B`. If `H[Y] ~= nil`, merge that native SSH table into `R` in accordance with 8.5.2, preserving the precedence of the settings supplied by `B`.
4. If settings remain absent from `R`, obtain `O(Y)` and use it as the source of supplementation under 8.5.4. Any settings adopted in this step shall fill only settings still absent after the native merge. If this operation encounters an error, return the error as specified in 8.5.4.
5. Apply the defaults for settings that remain absent in accordance with 8.5.5, and return the result.

The resulting precedence shall be: the explicitly supplied settings in `B`, then settings from `H[Y]` when that table exists, then any settings adopted from `O(Y)`, and finally native defaults. The shallow merge requirement in 8.5.2 shall apply to the native table merge in step 3.

The lookup of `H[Y]` in step 3 shall be one direct lookup. Nauvim shall not recursively apply this host-resolution procedure to that table before merging it into `R`. In particular, it shall not follow that table's `host_name` to perform another native or OpenSSH lookup. Nested jump-host processing under 8.4.6 is a separate operation.

If `Y == X`, the native merge shall leave the supplied table unchanged. If settings remain absent, step 4 shall use `O(X)` before native defaults are applied.

If a Nauvim entry exists for `Y` but contains no `.ssh` table, it shall not supply native SSH settings. In this SSH-resolution context, `H[Y] == nil`; step 4 shall apply to any remaining absent settings using `O(Y)`.

#### 8.5.4 OpenSSH configuration source, compatibility and errors

When the resolution procedure requires `O(X)` or `O(Y)`, Nauvim shall read the file selected by `config.ssh.config` and use it as an OpenSSH configuration source for that host under this subclause.

`nauvim-core` should interpret and apply that source in a manner that approximates OpenSSH behaviour as closely as practicable, subject to the explicit requirements of this document, including 8.5.6. The concrete parsing, selection and application of settings inherited from that source are unspecified. This document does not require a correct or complete implementation of OpenSSH configuration-file syntax or semantics, or equivalent host-resolution results. Conformance of the SSH client used internally by Nauvim to the OpenSSH client is outside the scope of this document.

The native Lua input rules, including field types, literal-name matching and the absence of a native `Include` directive, do not define the interpretation of the OpenSSH source. The native table merge in 8.5.2 does not prescribe how OpenSSH-derived settings are inherited. Annex C provides informative correspondences between `SshHostConf` fields and OpenSSH directives, illustrating inheritance of some or all configured values or OpenSSH default values without requiring a particular selection or application method.

If an error occurs while reading or processing the required OpenSSH configuration source, or the required host settings cannot be obtained, host resolution shall fail and return the error to its caller. Nauvim shall not continue by using only the native table, substituting an empty configuration, selecting another configuration file, or relying on built-in connection defaults as a fallback for that error.

Where the implementation processes an `Include` directive, that processing forms part of source evaluation and is not an error fallback.

#### 8.5.5 Application of defaults

Native host tables shall retain their supplied-field information until the native and OpenSSH supplementation steps are complete. A built-in default shall not be treated as a supplied native value that overrides a setting obtained from either source.

For a native host resolved under steps 2 to 5 of 8.5.3, and for an inline jump host resolved under 8.4.6, the defaults in Table 3 shall fill only settings that remain absent after the native table merge and the OpenSSH-source processing under 8.5.4. Using the supplied-host-name default to select `Y` shall not cause other defaults to be inserted into `B` before merging.

EXAMPLE If the native settings do not supply `port` and the implementation adopts `2222` from an OpenSSH `Port` directive, the resulting port is `2222`. The native default `22` does not override that adopted value.

#### 8.5.6 Pseudo-terminal allocation

Nauvim shall not request a pseudo-terminal for an SSH session. Its effective terminal-allocation policy shall always be equivalent to `RequestTTY no`.

The Lua configuration shall not provide a `request_tty` field. Any `RequestTTY` directive in an OpenSSH configuration file shall be ignored and shall not alter this policy.

### 8.6 Host information

#### 8.6.1 HostInfo type

`HostInfo` shall have the following structure. All fields other than `config_file` shall be present and have the indicated types. The value of `config_file` shall follow the client and host role requirements in 8.6.3 and 8.7.1.

```text
HostInfo = {
    host_name: string,
    user: string,
    config_file: string | nil,
    home_dir: string,
    project_dirs: {
        cache_dir: string,
        data_dir: string,
        runtime_dir: string,
    },
}
```

The injected client-information module shall have the following result type:

```text
require("nauvim-client-info"): HostInfo
```

The fields shall provide values directly. They shall not be getter functions.

#### 8.6.2 Values, defaults and path properties

`nauvim-core` shall obtain the values for the environment represented by the `HostInfo`. The specific acquisition methods are unspecified.

On GNU/Linux, `user` shall be the effective user's username.

The defaults for `project_dirs` shall be those in Table 4, interpreted in the environment represented by the `HostInfo`.

**Table 4 — Project-directory defaults**

| Field | Default |
| --- | --- |
| `project_dirs.cache_dir` | `$XDG_CACHE_HOME/nauvim` |
| `project_dirs.data_dir` | `$XDG_DATA_HOME/nauvim` |
| `project_dirs.runtime_dir` | `$XDG_RUNTIME_DIR/nauvim` |

The values of `home_dir` and all three path fields within `project_dirs` shall be absolute paths with symbolic links resolved. When `config_file` is non-nil, it shall also be an absolute path with symbolic links resolved. These requirements shall apply to the resulting path values, including any default values used. The method used to obtain or resolve these paths is unspecified.

#### 8.6.3 Client information before configuration execution

The `HostInfo` returned by `require("nauvim-client-info")` shall describe the client environment. Its `config_file` shall identify the selected Nauvim configuration file and shall be non-nil.

If acquisition of a required client-information value fails and a reasonable default value exists, `nauvim-core` shall use that default. If no reasonable default value exists, it shall return an error and shall not execute the selected Lua configuration file.

An acquisition failure that prevents configuration execution shall be handled as a configuration-loading failure under 8.1.2.

Client-information acquisition shall be completed before any code in the selected Lua configuration file is executed, regardless of whether that code calls `require("nauvim-client-info")`. A partially populated `HostInfo` shall not be used to begin configuration execution.

In the failure case that prevents execution, whether the selected Lua configuration file is read or parsed is unspecified.

### 8.7 Neovim program and socket settings

#### 8.7.1 Accepted forms and host-information acquisition

The structured program type `NvimProg` shall be:

```text
NvimProg = {
    prog: string | nil,
    args: list[string] | nil,
    env: dict[string, string] | nil,
}
```

`NvimProg` shall specify a program through `prog`, an ordered sequence of argument strings through `args`, and a mapping of environment-variable names to string values through `env`.

An omitted or nil `prog` shall default to `"nvim"`. An omitted or nil `args` shall default to an empty sequence of configured arguments. An omitted or nil `env` shall default to an empty mapping of configured environment entries. The execution interpretation of that mapping shall remain subject to 8.7.2.

The settings shall accept the following types, irrespective of whether the host Neovim is local or remote:

```text
hosts[X].nvim: string | NvimProg | (function(HostInfo) -> (string | NvimProg)) | nil
hosts[X].socket: string | (function(HostInfo) -> string) | nil
```

An omitted or nil `nvim` setting shall default to `"nvim"`. The default for an omitted or nil `socket` setting is unspecified and shall remain subject to the socket and session-ownership requirements of this document. These field defaults shall also apply when no Lua host entry supplies the setting; host selection shall continue to follow 8.3 and 8.5.

For a directly supplied `nvim` value, whether a string or a structured `NvimProg`, and for a string-valued `socket` field, Nauvim shall use the supplied value without acquiring host-side `HostInfo` for the purpose of resolving that field. This rule shall not remove the client-information acquisition required before configuration execution by 8.6.3.

For a function-valued field, `nauvim-core` shall obtain `HostInfo` in the host environment and pass it to the function. If that acquisition fails, Nauvim shall return an error and shall not invoke the function. The result of an `nvim` function shall be a string or `NvimProg`; the result of a `socket` function shall be a string. The `NvimProg` field defaults shall also apply to a function result of that type.

The host-side `HostInfo` passed to either function shall have `config_file == nil`. This requirement shall apply to both local and remote host Neovims. This nil value is prescribed for the host role and shall not be treated as an information-acquisition failure.

The host environment in this rule shall be the environment in which the selected host Neovim runs or is to run. For a local host Neovim, it is the client environment. For a remote host Neovim, it is the remote host environment.

#### 8.7.2 Neovim program interpretation

The resolved `nvim` setting shall specify the program used to launch the host Neovim. It may specify Neovim directly or a wrapper program. The host-startup requirements in 6.1 shall apply.

The string form of the `nvim` setting and the `prog` field of `NvimProg` shall not be required to be absolute paths or to have symbolic links resolved. The path requirements for values within `HostInfo` in 8.6.2 shall not be applied to these program values.

Nauvim shall append the following three arguments immediately after the configured `args` sequence, in the order shown:

```text
"--headless", "--listen", socket
```

Here, `socket` shall be the value obtained by resolving the selected host's `socket` setting under 8.7.1. For a string-valued `nvim` specification, the configured argument sequence shall be treated as empty for this purpose. This rule shall apply to directly supplied values and function results, and to both direct Neovim programs and wrapper programs, for local execution and execution through SSH.

Subject to this required argument suffix and the socket-argument requirements in 8.7.3, the concrete expansion and execution interpretation of `prog`, `args` and `env` are unspecified for both local execution and execution through SSH. The same qualification shall apply to the concrete expansion and execution interpretation of a string-valued `nvim` specification.

Nauvim shall not validate program-path interpretability in advance. Launch failures shall be determined from the errors reported by the execution mechanism and handled in accordance with 6.1.

#### 8.7.3 Socket path interpretation and argument boundaries

The string supplied for `socket`, or returned by its function, shall not be required to be an absolute path or to have symbolic links resolved.

For a relative socket path, being interpretable by `exec(2)` for local execution, or by the host environment's shell for remote execution, shall be the necessary and sufficient path-interpretation requirement. Nauvim shall not validate this requirement in advance. It shall determine failure from the errors reported by local execution or the remote shell, respectively.

The path requirements for values within `HostInfo` in 8.6.2 shall not be applied to the resulting `socket` string.

The resulting `socket` value shall be passed as one string in `argv`. Whitespace within the value shall not cause it to be split into multiple arguments.

EXAMPLE The socket value `"spaced socket.sock"` is one argument, `"spaced socket.sock"`; it is not the two arguments `"spaced"` and `"socket.sock"`.

When execution uses SSH, Nauvim shall escape the socket value correctly so that the host environment's shell interprets it as one argument.

The effect of that escaping on shell-specific expressions, including `~` and variable expansion, is unspecified.

### 8.8 Clipboard backend configuration

#### 8.8.1 Form, defaults and timeout validation

Nauvim shall accept a top-level `clipboard` setting in the Lua configuration file, at the same level as `hosts`, with the following type:

```text
Clipboard = {
    backend: string | nil,
    timeout_ms: int | nil,
}

clipboard: string | Clipboard | nil
```

The string form shall identify a clipboard backend. In the table form, a non-nil `backend` shall identify the backend and a non-nil `timeout_ms` shall specify a positive integer timeout value in milliseconds. An omitted or nil `backend` shall default to `"auto"`.

The default value of `timeout_ms` is unspecified. Nauvim shall use that default when the string form is used, when `clipboard` is omitted or nil, or when the table form omits `timeout_ms` or supplies it as nil. Such omission or nil shall not itself be a validation error.

For a non-nil `timeout_ms` in the table form, `nauvim-core` shall check that the value has integer type and is greater than or equal to `1`. Floating-point values shall be invalid, including values that are mathematically integral. Other supplied non-nil values of non-integer types shall also be invalid.

If this validation fails, `nauvim-core` shall display an error message in the client environment and fall back to the default timeout value. Nauvim shall proceed using that default rather than fail configuration loading because of the invalid timeout value. Backend availability validation shall follow 8.8.2.

The backend identifier `"auto"` shall request automatic detection of a clipboard backend in the client environment. It shall be accepted in either form. When `clipboard` is omitted or nil, its value shall default to `"auto"`.

The specific backend identifiers accepted other than `"auto"` are unspecified. Backend implementations and their runtime dependencies shall not be limited to external commands.

EXAMPLE `"xclip"` and `"xsel"` illustrate intended backend identifiers. These examples do not require support for either particular backend.

#### 8.8.2 Availability validation

After executing the Lua configuration file, Nauvim shall validate the availability of the selected backend's required dependencies and execution environment in the client environment, or perform backend detection when `"auto"` is selected.

If the required runtime dependencies or execution environment are unavailable, or automatic detection fails, Nauvim shall display an error message in the client environment after configuration execution.

Following such a failure, Nauvim shall continue normal editor operation with clipboard functionality unavailable, subject to the environment-change provisions in 8.8.3. The failure shall not prevent Nauvim from operating as a Neovim GUI.

#### 8.8.3 Runtime errors and environment changes

Retention or reuse of an availability-validation or automatic-detection result is not guaranteed. Behaviour in response to subsequent changes in the client environment is unspecified.

Nauvim shall display error messages in the client environment for errors encountered during clipboard operations, including runtime failures of the selected backend. This requirement shall apply irrespective of the result of an earlier validation or detection attempt.

EXAMPLE A dependency can be installed after validation fails, or removed after validation succeeds. Whether and when Nauvim detects such changes, repeats validation or selects another backend are unspecified.

### 8.9 Global SSH configuration

The top-level `ssh` setting shall have the following type:

```text
SshConf = {
    config: string | nil,
    known_hosts: string | nil,
}

config.ssh: SshConf
```

`config.ssh.config` shall select the OpenSSH configuration file used by 8.5. `config.ssh.known_hosts` shall select the known-hosts file used by Nauvim's SSH client.

An omitted or nil `config` field shall default to `$HOME/.ssh/config`. An omitted or nil `known_hosts` field shall default to `$HOME/.ssh/known_hosts`. In this default-path notation, `$HOME` denotes the home directory in the client environment.

Path values in `SshConf` shall be absolute paths in the client environment. A relative or empty path shall be a configuration-validation error and shall be handled under 8.1.2.

Nauvim may omit permission checks for the paths in `SshConf`. Conformance to the OpenSSH client's permission-checking behaviour is not required. Errors encountered when accessing these files shall be handled under the otherwise applicable requirements of this document, including 8.5.4 for the OpenSSH configuration source.

<a id="810-font-configuration-and-responsibilities"></a>

### 8.10 UI-specific configuration

The `ui_opts` setting shall supply application settings specific to the `nauvim` implementation. Neovim UI attachment options shall remain subject to 6.3.1.

#### 8.10.1 Structure, defaults and common validation

The top-level `ui_opts` and `use_ui_opts` fields shall have the following types:

```text
config.ui_opts: dict[string, dict[string, object]] | nil
config.use_ui_opts: string | nil
```

An omitted or nil `ui_opts` shall be equivalent to an empty mapping. The default value of `use_ui_opts` shall be nil.

Every key in `ui_opts` and every non-nil value of `use_ui_opts` shall be a non-empty, valid UTF-8 byte sequence. It shall not begin with `U+FEFF` (ZERO WIDTH NO-BREAK SPACE) and shall not contain `U+0000` (NUL).

Each non-nil entry in `ui_opts` shall be a dictionary with string keys. In 8.2.2 and this subclause, `object` denotes an arbitrary Lua value, subject to the prohibition on functions. UI-specific option dictionaries shall not contain functions at any nesting level, including as keys or values in nested tables.

Apart from the dictionary structure and the prohibition on functions, value ranges and representations within UI-specific option dictionaries are unspecified by this document. This includes string encoding, integer sign and range, and floating-point precision. The string requirements for keys in `ui_opts` and for `use_ui_opts` shall not apply to strings within option dictionaries.

`nauvim-core` shall validate these common requirements for all entries in `ui_opts`, including unselected entries, and for `use_ui_opts`. Validation failures shall be handled under 8.1.2.

#### 8.10.2 Selection and delivery

After configuration evaluation, `nauvim-core` shall determine the selection key as follows:

1. If `use_ui_opts` is non-nil, its value shall be used.
2. Otherwise, the UI implementation ID supplied by `nauvim` under 4.5 shall be used.

Keys shall be matched exactly. A key supplied by `use_ui_opts` need not match a UI implementation ID.

`nauvim-core` shall treat an absent or nil entry at the selection key as an empty dictionary. A non-nil entry shall be a dictionary with string keys; otherwise, configuration validation shall fail under 8.1.2. `nauvim-core` shall make the resulting dictionary available to `nauvim`.

An absent or nil entry shall not itself be a configuration-validation error. When an explicit `use_ui_opts` value has no matching entry or its entry is nil, `nauvim-core` shall use the empty dictionary rather than fall back to lookup using the UI implementation ID. Option dictionaries shall not be merged as part of selection.

#### 8.10.3 Interpretation and responsibilities

Each UI implementation shall define and document its UI-specific option schema, including field names, types, meanings and defaults. These schemas are outside the scope of this document.

`nauvim` shall interpret and validate the selected option dictionary according to its schema. Such validation failures shall be handled under 8.1.2. Unselected dictionaries shall not be interpreted or validated against UI-specific schemas. The common validation requirements in 8.10.1 shall remain applicable.

`nauvim-core` shall not interpret or validate UI-specific option fields beyond the common requirements in 8.10.1. The concrete interface used to deliver the selected dictionary to `nauvim` is unspecified.

Font configuration shall be part of the UI-specific option schema. `nauvim` shall be responsible for font discovery, selection and application.

`nauvim-core` shall not provide methods for searching for, applying, investigating or diagnosing fonts.

UI-specific options shall not override the requirements of this document.

### 8.11 Other configuration details

Annex A illustrates the configuration model, including client information, host entries, SSH settings, socket paths, clipboard settings and UI-specific options. Annex A does not add normative requirements to Clause 8.

Configuration details that remain to be specified shall be distinguished from details expressly designated as unspecified or implementation-dependent. The matters still requiring specification are listed in Annex B.

## 9 Build-time features and dependencies

### 9.1 Build-time features

Except for the functionality corresponding to `nauvim-plugin::client`, `nauvim-core` shall permit plugin-specific functionality to be excluded at build time using Cargo feature flags.

The `nauvim-core` functionality corresponding to `nauvim-plugin::client` shall always be implemented and shall not be controlled by a Cargo feature flag.

For a plugin module denoted by `nauvim-plugin::X` whose core functionality is controlled by a feature flag, that flag shall be named `plugin-X`. The optional plugin modules specified in Clause 11 shall use the feature names in Table 5.

**Table 5 — Plugin module feature flags**

| Plugin module | Cargo feature |
| --- | --- |
| `nauvim-plugin::local-forward` | `plugin-local-forward` |
| `nauvim-plugin::filesystem` | `plugin-filesystem` |
| `nauvim-plugin::http` | `plugin-http` |
| `nauvim-plugin::clipboard` | `plugin-clipboard` |

SSH support shall be controlled by the `ssh` feature.

The SSH feature and the feature flags controlling plugin-specific functionality shall be enabled by default.

EXAMPLE Conditional compilation for the `local-forward` module can use `#[cfg(feature = "plugin-local-forward")]`.

Requirements for optional functionality apply when the corresponding functionality is enabled in the build.

RPC dispatch shall not distinguish a method excluded by a disabled feature from any other method that the Nauvim engine does not recognize. It shall not maintain a separate disabled-feature method category. Such a method shall be handled as an invalid method under 11.2.3, subject to the client-ID checks in 11.2.5.

### 9.2 Dependency allocation

Subject to 9.1, the implementation dependencies shall be allocated as listed in Table 6.

**Table 6 — Implementation dependencies**

| Component | Dependency | Purpose or configuration |
| --- | --- | --- |
| `nauvim-core` | `openrussh-client` | SSH support |
| `nauvim-core` | `mlua` | Lua configuration evaluation with the `lua54` and `vendored` features |
| `nauvim-core` | `directories` | General-purpose directory-path handling for `nauvim-core` configuration and directory-related `HostInfo` values, among other uses |
| `nauvim-core` | `tempfile` | Management of the process temporary directory |
| `nauvim-core` | `nvim-rpc-plugin` | Neovim RPC integration |
| `nauvim-core` | `nvim-rpc-ui` | Neovim UI integration |
| `nauvim-core` | `tokio` | Asynchronous runtime |
| `nauvim-core` | `hyper` | HTTP implementation for plugins, including client-side HTTP(S) server plugins |
| `nauvim` | Flutter | GUI framework |
| `nauvim` | `flutter_rust_bridge` | Integration between Flutter and Rust |
| `nauvim` | `nauvim-core` | Nauvim engine |

`nauvim-plugin` shall be implemented in pure Lua and shall require no external plugin or library dependencies beyond the host Neovim runtime.

Clipboard integration may require additional dependencies in the client environment. It shall not require additional dependencies in the host environment beyond the host Neovim runtime and `nauvim-plugin`.

EXAMPLE Client-side clipboard access can use utilities such as `xclip` or `pbcopy`.

NOTE `openrussh-client` is developed as a separate project.

## 10 User interaction, rendering and limitations

### 10.1 Keyboard and mouse interaction

Nauvim shall accept keyboard and mouse input.

When clipboard functionality is enabled, `nauvim` should provide key bindings that users would find natural on the client platform to invoke the GUI paste functionality of `nauvim-core` specified in 7.8. The specific key bindings are unspecified.

EXAMPLE `CTRL-V`, `CTRL-SHIFT-V` and `CMD-V` illustrate possible paste key bindings. These examples do not require any particular binding.

Nauvim shall not transmit mouse input to the host Neovim as Neovim mouse events or otherwise deliver that input through Neovim's mouse-input facilities.

Nauvim may use mouse input for GUI operations, including text selection, copying and actions invoked by clicking. Such actions may send RPC requests or notifications to the host Neovim, provided that they do not deliver mouse input through Neovim's mouse-input facilities.

If Nauvim implements mouse-based text selection in Neovim UI grids, selection should be independent for each grid. Each selection should include only text within its target grid.

These requirements shall also apply to GUI windows opened by `nauvim-plugin::http.open_webview`.

The prohibition on delivering Neovim mouse input shall apply regardless of the host Neovim's `mouse` option or changes to that option. This document does not require or guarantee that the option is disabled or has an empty value.

NOTE The prohibited delivery includes sending mouse input using `nvim_input_mouse()` or encoded mouse-event input sequences. Neovim's input interfaces are described in [1].

### 10.2 Input method editors

Native IME integration is outside the scope of this draft.

Neovim plugins can provide input-method functionality within the host Neovim.

EXAMPLE `skkeleton` provides a plugin-based input method.

### 10.3 Placement and rendering of Neovim windows and grids

The arrangement and rendering of Neovim windows and UI grids are implementation-dependent.

The cell-coordinate requirements for `nauvim-plugin::http.open_webview` in 11.5.3 shall apply relative to the specified grid.

## 11 Nauvim plugin API

### 11.1 Notation, strings and RPC method names

In this document, the following notation applies:

- `nauvim-plugin::X` denotes `require("nauvim-plugin.X")`;
- `nauvim-plugin::X.Y` denotes `require("nauvim-plugin.X").Y`.

In API signatures, `int`, `string`, `bool` and `dict` denote an integer, a string, a Boolean value and a mapping, respectively. Type annotations in these signatures are descriptive notation rather than Lua syntax.

Each `nauvim-plugin` Lua function specified by a signature in this clause shall take a single dictionary argument with the fields shown in that signature. This requirement shall include the internal `client.setup` entry point in 11.7.1. Neovim-facing clipboard-provider callbacks shall retain the signatures specified in 11.6.

Every string carried in a plugin RPC request, notification or response shall be a valid UTF-8 byte sequence. It shall not begin with `U+FEFF` (ZERO WIDTH NO-BREAK SPACE, UTF-8 bytes `EF BB BF`) and shall not contain `U+0000` (NUL, UTF-8 byte `00`). These requirements shall apply to each string at every nesting level, including string arguments, results, error values, mapping keys and clipboard data. Additional method-specific constraints shall also apply.

NOTE `U+FEFF` has a valid UTF-8 encoding; its exclusion at the start of a string is a restriction of this interface. At the start of a UTF-8 stream, it can serve as an encoding signature, also called a byte order mark (BOM), but does not indicate byte order. See [23], Clause 6.

These string requirements shall not apply to Lua error strings generated locally in the host Neovim that are not carried in a plugin RPC response.

`GridId` denotes an identifier of a Neovim UI grid, as used by the Neovim UI protocol. See [2].

RPC request and notification method names addressed to the Nauvim engine through `nauvim-plugin` shall match the following complete-string pattern:

```regex
^nauvim-plugin/[A-Za-z_:.-]+$
```

The RPC message kinds and method names for the interfaces listed in Table 7 shall be as specified in that table.

Every plugin RPC request or notification addressed to the Nauvim engine shall carry a mandatory `client_id: int` as its first method argument.

The client ID shall be encapsulated within `nauvim-plugin` and shall not be a field of its public operation input dictionaries. For each interface in Table 7, `nauvim-plugin` shall convert the input dictionary to a positional RPC argument array. It shall place its internally stored client ID first, followed by the values of the dictionary fields in the order specified in Table 7. The public Lua signatures describe the caller-facing dictionary fields; the RPC argument lists in Table 7 describe the arrays received by `nauvim-core`.

The requirement to prepend the stored client ID shall also apply to the connection notification in 6.5 and to the clipboard RPCs in 11.6. Native Neovim API calls, including `nvim_set_client_info` and `nvim_exec2`, shall retain their Neovim-defined argument lists.

EXAMPLE With a selected destination, the public call `require("nauvim-plugin.local-forward").clear({ handle = handle })` is encoded as the RPC notification `nauvim-plugin/local-forward.clear` with the parameter list `{ stored_client_id, handle }`.

**Table 7 — Public Lua API and RPC method mapping**

| Public Lua API notation | RPC message kind | RPC method name | RPC arguments, in order |
| --- | --- | --- | --- |
| `nauvim-plugin::local-forward.open_tcp` | Request | `nauvim-plugin/local-forward.open_tcp` | `client_id, client_port, host_addr, host_port` |
| `nauvim-plugin::local-forward.open_socket` | Request | `nauvim-plugin/local-forward.open_socket` | `client_id, client_socket_path, host_socket_path` |
| `nauvim-plugin::local-forward.get_port` | Request | `nauvim-plugin/local-forward.get_port` | `client_id, handle` |
| `nauvim-plugin::local-forward.clear` | Notification | `nauvim-plugin/local-forward.clear` | `client_id, handle` |
| `nauvim-plugin::filesystem.copy` | Request | `nauvim-plugin/filesystem.copy` | `client_id, client_file, host_file, override` |
| `nauvim-plugin::filesystem.clear` | Notification | `nauvim-plugin/filesystem.clear` | `client_id, handle` |
| `nauvim-plugin::http.serve_file` | Request | `nauvim-plugin/http.serve_file` | `client_id, client_file, client_port, https, http2` |
| `nauvim-plugin::http.get_port` | Request | `nauvim-plugin/http.get_port` | `client_id, handle` |
| `nauvim-plugin::http.open_webview` | Request | `nauvim-plugin/http.open_webview` | `client_id, url, position` |
| `nauvim-plugin::http.clear` | Notification | `nauvim-plugin/http.clear` | `client_id, handle` |
| `nauvim-plugin::clipboard.copy` | Notification | `nauvim-plugin/clipboard.copy` | `client_id, register, text` |
| `nauvim-plugin::client.connect` | Notification | `nauvim-plugin/client.connect` | `client_id, host` |

### 11.2 Common rules

The no-effect rule in 7.3 shall apply to the functions specified in this clause, except for `nauvim-plugin::client.setup` and the clipboard-provider `paste` callbacks specified in 11.6.2.

`nauvim-core` shall perform the argument validation required by this document for plugin RPCs.

`nauvim-core` shall ensure that all result and error values returned in plugin RPC responses satisfy the applicable requirements of this document. This requirement shall apply to values of every type and at every nesting level. The means by which response-value conformance is ensured are unspecified.

`nauvim-plugin` shall check its stored client ID and RPC channel ID for `nil` as specified in 7.3. It shall not validate plugin RPC arguments or responses received through RPC.

Plugin RPCs received by `nauvim-core` shall be subject to the client-ID checks in 11.2.5 before method-specific requirements are applied.

#### 11.2.1 Lua return convention

Every `nauvim-plugin` RPC-request function shall use the Lua return convention `ret, err`.

The value of `err` shall be `nil` when no request error is returned and non-`nil` when a request error is returned. Request errors shall be represented by this return value rather than raised as Lua errors.

The type of `err` shall be `string | nil`. A non-nil `err` shall be a non-empty string; `err = ""` shall not be permitted. An `err` representing an error returned in a plugin RPC response shall satisfy 11.1.

The absence of a request error shall be determined using `err == nil`. The value or truthiness of `ret` shall not be used as the error indicator.

The meaning and type of a successful `ret` shall be as specified for the applicable method.

An RPC notification shall not produce an RPC response. The `ret, err` convention does not establish a response for a notification.

NOTE This is a Lua interface convention. MessagePack-RPC represents a response using separate error and result fields in a single response message. See [7].

#### 11.2.2 Asynchronous operations and resource handles

Nauvim shall perform the operations identified as asynchronous in this clause without requiring their completion before returning the corresponding resource handle, except for checks explicitly required within an RPC request.

Returning a resource handle does not indicate that the associated resource is ready or that the operation has completed successfully. Work can begin or finish before or after the response is returned, subject to the requirements of the applicable method.

The scheduling mechanism for asynchronous work is implementation-dependent. This document does not prescribe whether such work is executed concurrently or in parallel.

Nauvim shall provide fair scheduling. Asynchronous work that remains eligible for execution and has not been cancelled shall eventually be scheduled for execution. This requirement shall apply both to starting an operation and to resuming it after a wait when it becomes eligible to execute again.

NOTE Fairness concerns scheduling. It does not establish a fixed scheduling delay or guarantee completion of an external I/O operation.

A resource handle shall identify the associated operation and resources sufficiently to perform the corresponding `clear` operation.

Resource handles associated with operations or resources at the same time shall be unique within a Nauvim session. This requirement shall apply across plugin modules and resource types within that session.

Uniqueness across the Nauvim process as a whole, across clients or in any other broader scope is not required. Randomness and unpredictability are not required.

Resource-handle values may be reused after their previous associations have been removed, including within the same session. Equality with a previously allocated handle value is permitted.

`nauvim-plugin` shall address a resource using its internally stored client ID and the resource handle supplied by the caller. After the client-ID checks in 11.2.5 succeed, Nauvim shall determine the resource to operate on solely from the supplied handle value. If a corresponding resource exists in the receiving process, Nauvim shall apply the requested operation to it, subject to the applicable method's resource-type requirements. The session in which the handle or resource was created, including whether it is the current or an earlier session, shall not be an additional condition for this determination.

Historical allocations shall not establish a separate resource identity. A reused handle shall refer to the resource corresponding to its value at the time of the operation, including when the caller supplies a value retained from an earlier allocation. If no corresponding resource exists, the applicable method's rules for an unknown handle shall apply. This lookup rule shall not add a requirement to exclude equal handle values originating in different sessions.

The process client ID shall remain subject to 7.2.1, including its collision provisions.

Resource-handle generation shall constitute an atomic allocation with respect to other handle allocations in the same Nauvim session. This shall include any uniqueness check and reservation needed to establish the handle, so that a time-of-check-to-time-of-use (TOCTOU) race cannot cause duplicate simultaneous allocations.

The specific generation mechanism and algorithm are unspecified.

Handling of resource associations, including repeated operations following cleanup or failure, shall satisfy the idempotency requirements in 11.2.7 and the applicable resource-lifecycle requirements.

#### 11.2.3 Error reporting

Nauvim shall return request errors through the convention in 11.2.1 where the applicable method specifies a request error.

Where the applicable method specifies a Nauvim error, Nauvim shall display the error in the client environment.

Errors encountered when sending RPC requests or notifications, including errors encountered when a stored channel ID is stale, shall be handled using the same ordinary RPC error handling as other request errors. A stale channel shall not introduce a separate error category or recovery procedure. RPC-request functions shall report such errors through 11.2.1. Notifications shall retain the no-response rule in 11.2.1. The specific send-failure behaviour of `client.connect` shall follow 11.7.4.

Subject to 11.2.5, an invalid or unrecognized RPC method shall be treated as an ordinary request error. For a request, Nauvim shall return that request error; for a notification, Nauvim shall not send an RPC response. This handling shall apply equally when a method is absent because its corresponding feature was excluded from the build.

Forwarding-operation errors classified as Nauvim errors under 11.3.3 shall not be reported to the host Neovim, except for binding failures returned in response to `local-forward.get_port` as specified in 11.3.5. The copy-operation errors specified in 11.4 shall not be reported to the host Neovim.

Such an error shall not by itself cause the corresponding `local-forward.open_tcp`, `local-forward.open_socket` or `filesystem.copy` request's `err` result to become non-`nil`. This requirement applies irrespective of whether the error is detected before or after that request's response is returned.

NOTE For a request that returns a resource handle, an `err` value of `nil` reports the absence of a request error; it does not by itself indicate successful completion of the associated asynchronous operation.

#### 11.2.4 Clearing and cancellation

Each `clear` function shall issue an RPC notification and shall have the effects specified for its module.

Conditions referring to cancellation before binding, copying or server startup shall be evaluated when Nauvim processes the notification. Cancellation requirements for pending operations take precedence over a no-effect rule based solely on the absence of an instantiated resource.

Detachment cleanup shall apply to all associated client resources under 6.3, including HTTP(S) servers and the copy operations and destinations specified in 11.4.5. Application-shutdown cleanup shall follow 6.6, and connection-recovery cleanup shall follow 6.8.

The task-stopping provisions in 6.6.3 apply to application shutdown and, as specified in 6.8.2, connection recovery. They shall not relax the module-specific stopping and deletion requirements of a `clear` operation or the copy-cleanup requirements on detachment in 11.4.5.

#### 11.2.5 Client-ID checks for incoming plugin RPCs

For each incoming plugin RPC, `nauvim-core` shall examine the mandatory first argument, `client_id`, before performing method-specific validation, allocating a resource handle or executing the requested operation.

These requirements shall apply whether the RPC was sent through `nauvim-plugin` or constructed directly by another caller.

`nauvim-core` shall perform no operation if `client_id` is missing, is `nil`, is not an integer, is outside the inclusive range from `0` to `4294967295`, or does not equal the client ID of the receiving Nauvim process.

In each such case, an RPC request shall complete with a nil result and no error, corresponding to the Lua return values `nil, nil`. An RPC notification shall have no effect and shall produce no response. These no-effect rules shall take precedence over the applicable method's normal validation, result and operation requirements.

#### 11.2.6 Client POSIX paths and confinement

This subclause shall apply to the `client_file` input fields of `filesystem.copy` and `http.serve_file` and the `client_socket_path` input field of `local-forward.open_socket`, and to their corresponding RPC arguments. Let `T` denote the process temporary directory specified in 6.6.5, and let `P` denote the supplied path value. These interfaces shall use `T` as the base directory for `P`.

Nauvim shall interpret `P` as a POSIX path using the supplied string. A string resembling a Windows path or a URL with a scheme shall still be interpreted as a POSIX path; it shall not receive Windows-path or URL interpretation. In particular, URL percent-decoding shall not be applied to `P`.

If the supplied bytes are not valid UTF-8 and therefore cannot be decoded as Rust `str` or `String`, Nauvim shall reject the argument as invalid and return a request error. The common string requirements in 11.1 shall also apply.

Within the RPC request, before initiating copying, server startup or client-side socket binding, Nauvim shall perform the following path interpretation and validation:

1. Remove all leading slash (`/`) characters from `P`.
2. Interpret the remaining string as a relative POSIX path under `T` and resolve the location designated by `T/P`, including symbolic links and relative path components.
3. Determine whether the resulting location is a strict descendant of `T`, using the resolved filesystem hierarchy and path components.
4. If the resulting location is `T` itself or lies outside `T`, reject the argument as invalid and return a request error. Otherwise, proceed with the applicable method-specific checks and operation.

Only a strict descendant, such as `T/foo` or `T/foo/bar`, shall be an accepted target. A shared textual prefix with `T` shall not by itself establish containment.

Subsequent references to the location designated by either `client_file` or `client_socket_path`, including copying, serving-root selection, client-side socket binding and associated cleanup, shall refer to the target determined by this rule. The original final destination entry shall remain subject to the copy-specific type checks in 11.4.2.

The same target restriction shall apply to `filesystem.clear`, to `http.clear` for an operation or server created by `http.serve_file`, and to `local-forward.clear` for a Unix domain socket forwarding operation. If the associated resolved target is `T` itself or lies outside `T`, the notification shall have no effect. Window resources created by `http.open_webview` shall continue to be governed by 11.5.4 without a `client_file` path requirement.

EXAMPLE 1 The argument values `preview/index.html`, `/preview/index.html` and `///preview/index.html` all produce the path `T/preview/index.html` before symbolic-link resolution and the strict-descendant check.

EXAMPLE 2 An empty path after leading-slash removal, or `P = "."`, designates `T` itself and is rejected. A path resolving outside `T`, including through a symbolic link, is also rejected.

#### 11.2.7 Idempotency of handle operations

Operations addressed by a resource handle shall be idempotent. Applying the same RPC request or notification to the same handle with the same arguments multiple times shall have the same result as applying it once.

Differences caused by events outside `nauvim-core` shall not be subject to this idempotency requirement. Such events include physical disconnection of storage and interference by another process that changes an operation's ability to succeed.

Handle lookup for each invocation shall follow 11.2.2. The idempotency requirement shall not introduce an additional resource-selection condition based on the session of creation or historical allocation identity.

### 11.3 Local forwarding

#### 11.3.1 Opening a TCP forward

**Signature**

```text
nauvim-plugin::local-forward.open_tcp({
    client_port: int,
    host_addr: string,
    host_port: int
}) -> ret, err
```

This function shall issue an RPC request.

Within the RPC request, Nauvim shall verify that `client_port` is an integer from `0` to `65535`, inclusive, and that `host_port` is an integer from `1` to `65535`, inclusive. An invalid type or a value outside the applicable range shall be treated as an invalid argument and returned as a request error before the forwarding operation is initiated.

Nauvim shall initiate an asynchronous operation to bind a TCP listener at `127.0.0.1:client_port` in the client environment and forward accepted connections to `host_addr:host_port` reached from the host environment. The client-side bind address shall always be the IPv4 loopback address `127.0.0.1`.

The successful `ret` shall be a resource handle for the forwarding operation. Completion of `bind(2)` before returning the handle is not guaranteed.

The binding outcome shall remain available through the resource handle for `local-forward.get_port` to report in accordance with 11.3.5.

For forwarding through SSH, each accepted connection shall be forwarded using an SSH channel of type `direct-tcpip`. See [6].

#### 11.3.2 Opening a Unix domain socket forward

**Signature**

```text
nauvim-plugin::local-forward.open_socket({
    client_socket_path: string,
    host_socket_path: string
}) -> ret, err
```

This function shall issue an RPC request.

Within the RPC request, Nauvim shall interpret and validate `client_socket_path` in accordance with 11.2.6 before initiating the forwarding operation.

Nauvim shall initiate an asynchronous operation to bind a Unix domain socket listener at `client_socket_path` in the client environment and forward accepted connections to `host_socket_path` in the host environment.

The host-side destination path shall follow Neovim's path and current-directory semantics. If `host_socket_path` is relative, its base shall be the directory that the host Neovim considers current at the time it executes the `local-forward.open_socket` plugin code.

The successful `ret` shall be a resource handle for the forwarding operation. Completion of `bind(2)` before returning the handle is not guaranteed.

For forwarding through SSH, each accepted connection shall be forwarded using an SSH channel of type `direct-streamlocal@openssh.com`. See [11].

#### 11.3.3 Forwarding errors

Errors encountered during forwarding operations, including `bind(2)` failures, `accept(2)` failures, failures to connect to the forwarding destination and failures to establish an SSH forwarding channel, shall be Nauvim errors and shall be handled in accordance with 11.2.3. Request-argument validation errors shall retain the request-error handling specified by the applicable method.

A failure to bind because the requested client port or socket endpoint is already in use shall not be returned as a request error by `local-forward.open_tcp` or `local-forward.open_socket`.

Binding failures queried through `local-forward.get_port` shall be returned as request errors in accordance with 11.3.5, including failures already reported as Nauvim errors.

#### 11.3.4 Clearing a forward

**Signature**

```text
nauvim-plugin::local-forward.clear({ handle: int })
```

This function shall issue an RPC notification.

For a Unix domain socket forwarding operation, the target restriction in 11.2.6 shall apply first. If the associated resolved client-side socket target is the process temporary directory itself or lies outside that directory, the notification shall have no effect.

If the handle identifies a local forward or a pending forwarding operation, Nauvim shall stop and close its listener or cancel its establishment, as applicable. If a listener is established while cancellation is taking effect, Nauvim shall close that listener.

Whether this notification closes sockets or streams that have already been accepted is unspecified. The applicable cleanup requirements in Clause 6 shall remain in effect.

If Nauvim processes the notification before the associated binding operation has begun, Nauvim shall cancel the pending forwarding operation without performing the bind operation.

The client-side Unix domain socket pathname created by a forwarding operation shall be an associated resource. Nauvim shall remove that pathname when clearing the forward and during applicable detachment, shutdown or connection-recovery cleanup.

If no associated local forward or pending forwarding operation exists, the notification shall have no effect.

#### 11.3.5 Obtaining the assigned forwarding port

**Signature**

```text
nauvim-plugin::local-forward.get_port({ handle: int }) -> ret, err
```

This function shall issue an RPC request using the method name `nauvim-plugin/local-forward.get_port`.

Nauvim shall resolve `handle` to a TCP-forwarding operation or TCP forward created through `local-forward.open_tcp`. If the handle cannot be resolved to such an operation or forward, the request shall return a request error. This includes handles for Unix domain socket forwards and resources created by other interfaces.

If binding has not yet completed or failed, Nauvim shall wait for the outcome of the associated binding operation. This waiting shall be non-blocking for Nauvim: the application shall remain able to process UI events, RPC messages and other work while the request is pending.

If Nauvim processes the corresponding `local-forward.clear` notification while this request is waiting, it shall immediately complete the request with a nil result and no error. The Lua return values shall be `nil, nil`. Completion of binding or cleanup shall not be awaited before completing that request.

Once binding has succeeded, the successful `ret` shall be the integer port number actually assigned to the TCP listener in the client environment. If binding had already succeeded before the request was received, Nauvim shall return that assigned port.

If binding has failed, `get_port` shall return a request error. This requirement shall apply whether the failure occurred before the request or while the request was waiting, including when the failure has already been reported as a Nauvim error.

All errors encountered in resolving the handle, waiting for binding or obtaining the assigned port shall be returned as request errors in accordance with 11.2.1.

NOTE 1 The non-blocking requirement applies to Nauvim. The RPC response can remain pending while Nauvim processes other work.

NOTE 2 A successful response reports successful client-side binding and the assigned port. The port of a bound socket can be obtained using `getsockname(2)` or an equivalent interface. See [15].

### 11.4 Filesystem operations

#### 11.4.1 Copy request and transport

**Signature**

```text
nauvim-plugin::filesystem.copy({
    client_file: string,
    host_file: string,
    override: bool
}) -> ret, err
```

This function shall issue an RPC request.

Within the RPC request, Nauvim shall interpret and validate `client_file` in accordance with 11.2.6. After that validation accepts the target, Nauvim shall initiate an asynchronous copy operation from `host_file` in the host environment to that client-side destination, subject to 11.4.2 and 11.4.3.

`host_file` shall follow Neovim's path and current-directory semantics. If it is relative, its base shall be the directory that the host Neovim considers current at the time it executes the `filesystem.copy` plugin code.

The successful `ret` shall be a resource handle associated with the copy operation and its client-side destination. Completion of the copy before returning the handle is not guaranteed.

When the host is accessed through SSH, Nauvim shall use the SFTP subsystem for host-side file access, metadata retrieval, directory enumeration and symbolic-link inspection required by the copy operation. See [12].

#### 11.4.2 Source types and destination replacement

Nauvim shall resolve source symbolic links in the host environment. A relative link target shall be interpreted relative to the directory containing that symbolic link. The copy rules shall then be applied to the resolved target.

For destination-type checks, Nauvim shall determine the type of the original final destination entry without following a symbolic link at that entry. Symbolic-link resolution for the confinement check in 11.2.6 shall not replace this entry-type check. An existing destination symbolic link shall be rejected when copying a regular file or directory, irrespective of `override`.

The action for the source path specified by `host_file` shall be as specified in Table 8, subject to the error handling in 11.4.3.

**Table 8 — Copy action by source type and destination state**

| Resolved source type | Destination state | `override == false` | `override == true` |
| --- | --- | --- | --- |
| Regular file | Absent | Copy the file | Copy the file |
| Regular file | Regular file | Leave the destination unchanged | Overwrite the destination |
| Regular file | Any other type, including a symbolic link | Report a Nauvim error | Report a Nauvim error |
| Directory | Absent | Copy recursively | Copy recursively |
| Directory | Directory | Leave the destination unchanged | Recursively delete the destination, then copy recursively |
| Directory | Any other type, including a symbolic link | Report a Nauvim error | Report a Nauvim error |
| Any other source type | Any destination state | Perform no copy | Perform no copy |

The destination-type error requirements in Table 8 shall take precedence over the rule that preserves an existing destination when `override` is false.

When an existing destination directory is replaced, Nauvim shall delete it recursively before copying the source directory. The operation shall replace the directory rather than merge the source into the existing directory.

NOTE Source symbolic-link resolution copies the target's contents. It does not reproduce the source symbolic link as a client-side symbolic link. Relative symbolic-link resolution is described in [13].

#### 11.4.3 Recursive traversal and copy errors

During recursive directory copying, Nauvim shall copy regular files and directories and shall resolve symbolic links to those types using the rules in 11.4.2. Other entry types shall be ignored.

Subject to the applicable requirements of this document, the concrete copying method is unspecified. This includes whether processing of other entries continues after an error.

A failure to retrieve metadata, resolve a symbolic link, enumerate a directory, read a source file or perform destination I/O shall be a Nauvim error. All other filesystem and I/O errors encountered by the copy operation shall also be Nauvim errors.

Insufficient permissions to read a source regular file, enumerate a source directory or traverse its entries shall be handled as Nauvim errors. These errors shall not be reported to the host Neovim, in accordance with 11.2.3.

A failure to resolve or inspect a source symbolic-link target shall be handled as an error, rather than as an entry of an unsupported type to be ignored.

NOTE Directory enumeration requires read access. Looking up entries and traversing directories requires search access, represented by execute permission for directories on the target Unix-like platforms. These access requirements are distinct from executing a regular file. See [13] and [14].

#### 11.4.4 Clearing a copy destination

**Signature**

```text
nauvim-plugin::filesystem.clear({ handle: int })
```

This function shall issue an RPC notification.

The following rules shall apply in the order given:

1. If the handle identifies neither a copy operation nor an associated destination, or the associated resolved target is the process temporary directory itself or lies outside that directory under 11.2.6, the notification shall have no effect and no further steps shall be performed.
2. If the associated copy operation has not yet begun, Nauvim shall cancel it and prevent it from starting. If it is in progress, Nauvim shall stop it. Nauvim shall complete the applicable cancellation or stopping step before proceeding. The cancelled or stopped operation shall perform no further writes to the destination.
3. If the destination was already present for the associated `copy` invocation and `override` was false, the notification shall leave that destination unchanged and no further steps shall be performed. This condition concerns a pre-existing destination, not a destination created by the copy operation.
4. Nauvim shall delete the associated destination if it exists, recursively if it is a directory, irrespective of whether the copy operation completed successfully, failed or was cancelled. If the destination does not exist, deletion shall have no effect.
5. If deletion fails, Nauvim shall terminate that deletion attempt without retrying or resuming it.

Completion or failure of the copy operation shall not by itself discard the association needed to clear its destination or perform detachment cleanup.

#### 11.4.5 Cleanup on detachment

On detachment, Nauvim shall clean up every copy operation and associated client-side destination belonging to the detached attachment.

If a copy operation has not yet begun, Nauvim shall cancel it and prevent it from starting. If a copy operation is in progress, Nauvim shall send it a stop signal and wait until it has stopped before deleting its destination. Nauvim shall not begin destination deletion while the copy operation can still write to that destination.

This wait shall be non-blocking for Nauvim: the application shall remain able to process UI events and other work while waiting for the copy task to stop.

After cancellation or termination of the copy operation, destination preservation and deletion shall follow steps 3 to 5 of 11.4.4, subject to the target restriction in 11.2.6. In particular, a pre-existing destination preserved because `override` was false shall remain unchanged; other associated destinations shall be deleted if present, recursively for directories, including partial results from failed or cancelled copies. A deletion attempt that fails shall end without retrying or resuming that attempt.

Detachment cleanup shall retain the process temporary directory itself in accordance with 6.6.5. Copy-destination deletion under this subclause shall apply only to accepted strict-descendant targets under 11.2.6.

NOTE The primary intended use of `filesystem.copy` is to create temporary client-side files or directories.

#### 11.4.6 Interference between copy destinations

Behaviour is undefined when distinct copy operations, or their associated destination-cleanup operations, interfere through identical destination paths or destination paths in an ancestor-descendant relationship.

This rule shall apply within an attachment and across attachments, including when cleanup belonging to an earlier attachment overlaps copying by a new attachment during session switching.

### 11.5 HTTP services and GUI access

#### 11.5.1 Serving a client-side file or directory

**Signature**

```text
nauvim-plugin::http.serve_file({
    client_file: string,
    client_port: int,
    https: bool,
    http2: bool
}) -> ret, err
```

This function shall issue an RPC request.

Within the RPC request, Nauvim shall first interpret and validate `client_file` in accordance with 11.2.6. Before starting the server, Nauvim shall also verify that:

- `client_file` exists in the client environment;
- its metadata identifies a regular file, a directory, or a symbolic link resolving to a regular file or directory;
- `client_port` is in the range from `0` to `65535`, inclusive;
- if `http2` is true, `https` is also true.

Failure of these checks shall be returned to the host Neovim as a request error.

In particular, `https == false` together with `http2 == true` shall be treated as an argument-validation error. Nauvim shall return a request error without starting the server.

For a directory or a symbolic link to a directory, Nauvim shall not inspect the metadata of descendant entries as part of the RPC request. The server shall check that metadata dynamically when serving the corresponding files.

After the checks succeed, Nauvim shall initiate asynchronous startup of a client-side HTTP server using the IPv4 loopback address `127.0.0.1` and the requested `client_port`.

The successful `ret` shall be a resource handle for the server-startup operation and its associated server. Completion of binding or startup before returning the handle is not guaranteed. The assigned port shall be obtained through `http.get_port` as specified in 11.5.2.

Errors encountered during asynchronous HTTP server startup, including binding failures, shall be reported as Nauvim errors in accordance with 11.2.3. The binding outcome shall remain available through the resource handle for `http.get_port` to report in accordance with 11.5.2.

If the resolved `client_file` is a regular file, the server shall serve that file. If it is a directory, the server shall serve files from that directory recursively.

The server shall handle each HTTP request using the file selected when that request is processed, in accordance with 11.5.5. The progress or completion of a `filesystem.copy` operation shall not be an additional condition for serving that file.

The server's HTTP request handling, URL-path mapping, response statuses and response headers shall comply with 11.5.5.

When `https` is true, the server shall use a self-signed certificate. Cryptographic algorithms, including signature algorithms, are implementation-dependent.

When `http2` is false, the server shall serve HTTP/1.1 only. When `http2` is true, the server shall serve HTTP/2 only. These version-selection requirements shall apply to all combinations accepted by the validation requirements above.

Whether the server handles multiple HTTP requests simultaneously is implementation-dependent.

#### 11.5.2 Obtaining the assigned server port

**Signature**

```text
nauvim-plugin::http.get_port({ handle: int }) -> ret, err
```

This function shall issue an RPC request using the method name `nauvim-plugin/http.get_port`.

Nauvim shall resolve `handle` to a server-startup operation or server created through `http.serve_file`. If the handle cannot be resolved to such an operation or server, the request shall return a request error.

If binding has not yet completed or failed, Nauvim shall wait for the outcome of the associated binding operation. This waiting shall be non-blocking for Nauvim: the application shall remain able to process UI events, RPC messages and other work while the request is pending.

If Nauvim processes the corresponding `http.clear` notification while this request is waiting, it shall immediately complete the request with a nil result and no error. The Lua return values shall be `nil, nil`. Completion of binding or cleanup shall not be awaited before completing that request.

Once binding has succeeded, the successful `ret` shall be the integer port number actually assigned to the bound socket. If binding had already succeeded before the request was received, Nauvim shall return that assigned port. When `serve_file` was called with `client_port == 0`, this result shall be the port allocated by the operating system.

If binding has failed, `get_port` shall return a request error. This requirement shall apply whether the failure occurred before the request or while the request was waiting, including when the failure has already been reported as a Nauvim error.

All errors encountered in resolving the handle, waiting for binding or obtaining the assigned port shall be returned as request errors in accordance with 11.2.1.

NOTE 1 The non-blocking requirement applies to Nauvim. The RPC response can remain pending while Nauvim processes other work.

NOTE 2 A successful response reports successful binding and the assigned port. It does not indicate that HTTP request processing has begun. The port of a bound socket can be obtained using `getsockname(2)` or an equivalent interface. See [15].

#### 11.5.3 Opening a window for an HTTP(S) URL

**Signature**

```text
nauvim-plugin::http.open_webview({ url: string, position: dict })
    -> ret, err
```

This function shall issue an RPC request.

Nauvim shall accept only the URL schemes `http` and `https` for `url`. A URL with any other scheme shall cause a request error to be returned to the host Neovim.

The `position` field shall be a dictionary with the following fields:

```text
{
    grid: GridId,
    top: int,
    left: int,
    height: int,
    width: int,
}
```

`grid` shall identify the Neovim UI grid relative to which the GUI window is positioned. `top`, `left`, `height` and `width` shall be measured in that grid's cells, in the sense used by the Neovim UI protocol.

The upper-left corner of the grid shall be `{ top = 0, left = 0 }`. Increasing `left` shall move to the right, and increasing `top` shall move downward. `top` and `left` shall specify the upper-left corner of the requested window; `height` and `width` shall specify its vertical and horizontal extents, respectively.

Each of `top`, `left`, `height` and `width` shall be an integer from 0 to 65535, inclusive. Nauvim shall validate these four values as part of the RPC request and return a request error for a value that does not satisfy this requirement.

For a request that passes the required URL-scheme and position-value validation, a `grid` identifier that does not identify an existing grid shall result in a no-effect operation: Nauvim shall not create a GUI window or initiate access to `url` for that operation.

Nauvim is not required to check whether a position whose four numeric values satisfy the required range fits within the actual grid. If the grid exists and those values are within range but the position or dimensions extend beyond the grid, behaviour is undefined.

For an existing grid and a position that fits within that grid, Nauvim shall initiate asynchronous creation of a new GUI window at `position` and access `url` through that window. The successful `ret` shall be a resource handle for the requested operation, including a no-effect operation for a non-existent grid. Creation of a window before returning the handle is not guaranteed.

Errors encountered during GUI window creation or access to `url` shall be reported as Nauvim errors in accordance with 11.2.3.

RPC handling by `nauvim-core` and concrete window creation by `nauvim` shall follow the allocation of responsibilities in 7.4.

The method name does not require the implementation to use a WebView. A WebView or another presentation component can provide the required access to `url`.

Mouse interaction in the opened window shall be governed by 10.1.

Resolution of the URL and its host name, where applicable, shall take place in the client environment through the presentation component used for that window.

URLs containing fragment identifiers shall be supported as specified in 11.5.5.7.

For HTTPS access, the presentation component used by `open_webview` shall accept self-signed certificates generated by `http.serve_file`.

Except for this acceptance requirement, the public key infrastructure (PKI) and certificate-validation behaviour, including the handling of root and intermediate certificate authorities (CAs) and their certificates, are unspecified.

NOTE The field name `url` describes the HTTP(S) resource to be accessed. URL terminology is described in [16].

#### 11.5.4 Clearing an HTTP server or window

**Signature**

```text
nauvim-plugin::http.clear({ handle: int })
```

This function shall issue an RPC notification.

For an operation or server created by `http.serve_file`, the target restriction in 11.2.6 shall apply first. If its associated resolved `client_file` target is the process temporary directory itself or lies outside that directory, the notification shall have no effect.

If Nauvim processes the notification before the associated `serve_file` server has started, Nauvim shall cancel the pending server-startup operation.

Otherwise, if the handle identifies an existing HTTP server or GUI window, Nauvim shall close it.

If no associated server or window exists and there is no pending `serve_file` startup to cancel, the notification shall have no effect.

NOTE The no-effect rule also applies when the window requested by `open_webview` does not yet exist.

#### 11.5.5 HTTP request and response rules

##### 11.5.5.1 Protocol upgrades

For both values of `http2`, a `Connection` header field containing the `upgrade` connection option shall be prohibited. The server shall not perform the requested protocol upgrade.

Whether such an upgrade attempt is ignored or rejected with an error at the application level is unspecified.

For HTTP/2, the protocol-level requirements for connection-specific fields and malformed messages in RFC 9113, 8.2.2 and 8.1.1, shall apply independently of that application-level choice.

NOTE HTTP/2 prohibits the `Connection` header field itself. Ignoring an upgrade attempt at the application level does not make a malformed HTTP/2 message valid.

##### 11.5.5.2 Request methods

The file-serving interface shall accept `GET` requests.

When `http2` is true, the server shall also accept any additional method whose acceptance is required or recommended for the server by the applicable HTTP/2 and HTTP semantics provisions in RFC 9113 and RFC 9110.

Methods other than `GET` and those covered by the preceding exception shall not be accepted by the file-serving interface. Whether requests using those methods are ignored or rejected with an error is unspecified.

NOTE The method restriction applies to HTTP requests handled by the file-serving interface. It does not disable processing of the HTTP/2 connection preface or protocol-control frames.

##### 11.5.5.3 URL-path mapping and confinement

The server shall map the URL path to a target in the client environment as specified in Table 9.

**Table 9 — URL-path mapping**

| URL path | Client-side target |
| --- | --- |
| `/` | `client_file` |
| `/path` | `client_file/path` |

In Table 9, `client_file` denotes the filesystem location obtained by applying 11.2.6 to the input field's value. The term `path` denotes the non-empty suffix following the initial slash and can contain further slash-separated path segments.

Relative path references shall be supported and resolved against their applicable base URL in accordance with the reference-resolution rules in RFC 3986, Clause 5. The resulting URL path shall be mapped in accordance with Table 9.

The serving root shall be the resolved filesystem location designated by `client_file` after the interpretation required by 11.2.6. When that location is a directory, the permitted scope shall comprise that directory and its descendants. When it is a regular file, the permitted scope shall comprise that file only.

The server shall assess confinement against the filesystem target actually selected after the applicable URL/URI processing in 11.5.5.6 and filesystem path resolution. Symbolic links in any component of the requested path shall be taken into account. Directory containment shall be determined using the resolved filesystem hierarchy and path components.

If the resolved target lies outside the serving root, the server shall return `400 Bad Request` with an empty response body. This rule shall also apply when the outside target is reached through a symbolic link, including a link in an intermediate path component.

The confinement requirement shall apply to the target actually used for file access, including when its selection is affected by an implementation's Unicode handling.

For `/`, target classification shall use the resolved `client_file` as specified in 11.5.1. Acceptance of a directory as the serving root shall not cause the server to substitute an index file or generate a directory listing for that path.

NOTE 1 When `client_file` resolves to a directory, `/` maps to that directory and therefore produces the non-regular-file response specified in 11.5.5.4. A regular file below that directory is addressed using its corresponding path suffix.

NOTE 2 Relative URL references can be resolved by the requesting client before an HTTP request is formed. The server's mapping and confinement rules apply to the resulting target; the presence of a `..` segment alone does not establish that the filesystem target is outside the serving root.

##### 11.5.5.4 File responses and content headers

The status codes in this subclause apply to HTTP responses sent to the HTTP client. For requests handled by the file-serving interface, the server shall apply the error-response rules in Table 10.

The implementation should take care to avoid time-of-check-to-time-of-use (TOCTOU) races throughout the processing of a single HTTP request, including between path resolution, containment and metadata checks, and file access.

**Table 10 — Error responses for mapped file targets**

| Condition at the mapped target | HTTP status | Response body |
| --- | --- | --- |
| Resolution establishes that the target lies outside the serving root, including through a symbolic link | `400 Bad Request` | Empty |
| The target does not exist | `400 Bad Request` | Empty |
| The target exists but is not a regular file | `400 Bad Request` | Empty |
| The target exists, but another error prevents it from being served | `500 Internal Server Error` | Empty |
| An error prevents the server from determining whether the target exists | `500 Internal Server Error` | Empty |

Every response specified in Table 10 shall contain zero octets of response content.

When resolution has established that the target is outside the serving root, the outside-root response shall apply irrespective of the target's existence or type. A confirmed absence shall be handled as a missing target, rather than as a failure to determine existence.

Every `200 OK` response shall include the `Content-Type` and `Content-Length` header fields. The specific value of `Content-Type` is unspecified. The value of `Content-Length` shall have the semantics specified in RFC 9110, 8.6.

If a `HEAD` request is accepted under 11.5.5.2, its response shall contain no response content, in accordance with RFC 9110, 9.3.2. For a `200 OK` response to that request, `Content-Length` shall indicate the number of octets that would have been sent in the corresponding `GET` response.

Handling of file-read or transmission errors after a response has begun, including after `200 OK` and its headers have been sent, is implementation-dependent. Such errors shall be handled in the same manner as other Nauvim errors under 11.2.3. HTTP-client behaviour in response to the resulting transfer or connection state is outside the scope of this document.

NOTE Header field names use conventional capitalization in this document. HTTP/2 encodes field names in lowercase, as specified in RFC 9113, 8.2.

##### 11.5.5.5 Cross-origin response headers

Every HTTP response emitted by the server shall include an `Access-Control-Allow-Origin` header field, including the error responses specified in Table 10.

Its value shall be a single explicit serialized origin. The values `*` and `null` shall not be used.

Within these constraints, the server shall select the value most strongly recommended from a security perspective for the intended use. The specific origin and the mechanism used to select it are unspecified, subject to this security requirement.

EXAMPLE `Access-Control-Allow-Origin: https://example.test:8443`

NOTE 1 A serialized origin identifies the scheme, host and applicable port; it does not include a URL path. The CORS header syntax is described in [17].

NOTE 2 Selecting only explicitly trusted origins and avoiding unchecked reflection of the request's `Origin` value are security practices described in [20].

##### 11.5.5.6 Path characters and encoding

The path-handling interface shall support all characters in the range U+0020 to U+007E, inclusive, corresponding to ASCII `0x20` to `0x7e`, and UTF-8-encoded path characters, subject to the Unicode provisions in this subclause.

Characters shall be represented and interpreted in accordance with the applicable HTTP and URL/URI syntax, including the reserved-character and percent-encoding rules in RFC 3986, Clause 2. Characters that require percent-encoding in their syntactic position shall be represented using percent-encoding. The server shall support percent-encoded path data.

Parsing and percent-decoding shall preserve URL component boundaries in accordance with RFC 3986, 2.4. Encoded path data shall not be reinterpreted as a fragment or query delimiter after decoding, or repeatedly decoded as though it were a newly encoded URL.

Outside the expressly supported ASCII range, the acceptance and processing of invisible Unicode characters, including bidirectional formatting and control characters, are unspecified. The handling of ideographic variation sequences (IVS) and Unicode normalization is also unspecified.

NOTE 1 The character-support requirement concerns characters represented in paths, including through percent-encoding where necessary. It does not permit characters to override the HTTP request-target syntax or the structural roles of URL delimiters.

NOTE 2 Unicode normalization is distinct from the URL/URI reference-resolution and encoding rules required by this subclause and 11.5.5.3.

##### 11.5.5.7 URL fragments

URLs for resources served by `http.serve_file` shall support fragment identifiers in accordance with RFC 3986, 3.5.

The fragment component shall be excluded from the HTTP target URI and from the filesystem target used by the server, in accordance with RFC 9110, 7.1. When such a URL is opened through `http.open_webview`, the presentation component shall preserve and process the fragment in accordance with the applicable representation semantics.

EXAMPLE For `/a%23b.html#section`, the server-side path identifies `a#b.html` beneath a directory serving root. The fragment identifier `section` is processed by the client and is not appended to the filesystem path.

### 11.6 Clipboard provider

#### 11.6.1 Copying text

**Signature**

```text
nauvim-plugin::clipboard.copy({
    register: "+" | "*",
    text: string
})
```

This function shall issue an RPC notification using the method name `nauvim-plugin/clipboard.copy`. It shall use the internally stored channel ID as the RPC destination and send the parameter list `{ client_id, register, text }`, with the internally stored client ID as its first argument.

The `register` field shall be either `"+"` or `"*"`.

The `text` field shall be a single string satisfying 11.1 and representing single-line or multiline text. Lines shall be separated by LF (`U+000A`, byte `0x0a`). The string may include or omit a trailing LF.

`nauvim-core` shall validate these arguments in accordance with 11.2 and 11.2.5 before passing `text` to the clipboard backend. If either argument is invalid, it shall not pass the text to the backend.

For a valid notification, `nauvim-core` shall pass `text` to the selected clipboard backend as specified in 7.8. How it uses `register` is unspecified.

The notification shall produce no RPC response. The function shall return `nil`; its return shall not indicate completion of the backend operation.

The no-effect rule in 7.3 shall apply to this function. Backend availability and clipboard-operation errors shall follow 8.8.

#### 11.6.2 Provider table

**Value**

```text
nauvim-plugin::clipboard.provider
```

`nauvim-plugin` shall expose `require("nauvim-plugin.clipboard").provider` as a clipboard-provider table suitable for direct assignment to `vim.g.clipboard` in the host Neovim.

The table shall conform to Neovim's clipboard-provider interface. Its `copy` and `paste` dictionaries shall each provide callbacks for both `"+"` and `"*"`. The callback argument and result contracts shall follow that interface. See [18].

The `copy` callbacks shall use `nauvim-plugin::clipboard.copy` to copy the supplied clipboard-register contents. How they use that function, including how they adapt Neovim's callback arguments to its input dictionary, is unspecified, subject to the requirements of the clipboard-provider interface and 11.6.1. The no-effect rule in 7.3 shall apply to the `copy` callbacks.

The `paste` callbacks shall supply no text for clipboard-register reads. They shall not access the client clipboard backend or issue RPC requests or notifications.

Each `paste` callback should return `{ {}, regtype }`, where the first element is an empty list of lines and `regtype` is a valid register type for a Neovim clipboard-provider result. The concrete implementation and return representation are otherwise unspecified, subject to the clipboard-provider interface and the requirement to supply no text.

These requirements for the `paste` callbacks shall apply regardless of whether an RPC destination is selected. The callbacks shall be exempt from the no-effect and nil-return rules in 7.3.

EXAMPLE The provider can be selected in the host Neovim's Lua configuration by the following assignment:

```lua
vim.g.clipboard = require("nauvim-plugin.clipboard").provider
```

### 11.7 Client registration, RPC destination selection and session switching

#### 11.7.1 Internal setup entry point

**Internal registration signature**

```text
nauvim-plugin::client.setup({ client_id: int })
```

This function shall serve as Nauvim's internal registration entry point and shall execute in the host Neovim. Nauvim shall invoke it through the execution RPC specified in 7.2.2. Its `client_id` input field belongs to this internal registration interface, not to the public operation interfaces described in 11.1.

The function shall construct `"nauvim-client-{client_id}"` using the supplied client ID and the decimal representation specified in 7.2.1. It shall search the UI clients currently attached to the host Neovim for clients whose advertised `client.name` equals that constructed name.

If one matching client exists, the function shall select its channel ID. If more than one matching client exists, it shall select the channel ID of any one of them; the selection rule is unspecified.

The undefined-behaviour rule for client-ID collisions in 7.2.1 takes precedence over these selection rules.

When a matching client is selected, the function shall store its channel ID and the supplied client ID as the plugin's internal RPC destination in accordance with 11.7.2.

If no attached client matches the constructed name, or the setup invocation otherwise fails, both stored identifiers shall be `nil`. The function shall return without raising or returning an error, displaying a message or taking any further action. Any previously stored destination shall not be retained after such an unsuccessful invocation.

The setup-failure rules in 7.2.2 shall apply. Failure shall leave normal Neovim GUI operation unaffected.

#### 11.7.2 Single destination and replacement

`nauvim-plugin` shall retain at most one selected RPC channel ID and at most one selected client ID at a time. Both stored identifiers shall initially be `nil`. Its internal destination shall consist of the selected channel ID and its associated client ID, or both identifiers shall be `nil`.

Each `setup` invocation that successfully selects a matching client shall replace both previously stored identifiers with the selected channel ID and the supplied client ID. An unsuccessful invocation shall leave both identifiers `nil`, as specified in 11.7.1. The most recent completed setup invocation shall therefore determine the stored destination state until it is changed by a later setup invocation or by `UILeave`.

Replacing or clearing the stored identifiers during setup shall not detach any previously selected Nauvim client or any other attached UI.

While another client is selected, resource handles belonging to a previously selected client cannot be cleared through the current `nauvim-plugin` destination. This loss of access through the plugin interface is permitted. Explicit detachment of the previously selected client remains available, with cleanup governed by 6.3.

NOTE The concurrent-instance limitations in 6.7 apply independently of these destination-selection rules.

#### 11.7.3 Clearing the destination on UILeave

When handling `UILeave`, `nauvim-plugin` shall compare the event's channel ID with its stored channel ID. If they are equal, it shall set both the stored channel ID and the stored client ID to `nil`. Otherwise, it shall leave both stored identifiers unchanged.

The event's channel ID shall be the `chan` value supplied by Neovim in `v:event`. See [19].

This document does not guarantee that `UILeave` occurs for every UI disconnection. Retention of a stale channel ID and its associated client ID when the corresponding event does not occur is permitted.

The interface provides no separate means of checking whether a stored channel ID is stale. `nauvim-plugin` shall use non-nil stored identifiers without a prior channel-staleness check. Errors encountered when sending through that channel shall follow 11.2.3.

#### 11.7.4 Connecting to another host

**Signature**

```text
nauvim-plugin::client.connect({ host: string })
```

This function shall issue an RPC notification using the method name `nauvim-plugin/client.connect`. It shall use the internally stored channel ID as the RPC destination and send the parameter list `{ client_id, host }`, with the internally stored client ID as its first argument.

The `host` field shall identify a host alias resolved by Nauvim in the client environment. Immediately after successfully issuing the notification, the function shall execute host-side detachment in accordance with 6.5. The function shall not wait for completion of detachment cleanup or of the new connection.

If issuing the notification fails, the function shall display an error message and perform no further operation for that call. It shall not initiate host-side detachment, retry the notification or clear the stored identifiers as a consequence of that failure.

The no-effect rule in 7.3 shall apply before either action. If either stored identifier is `nil`, the function shall send no notification, perform no detachment and return `nil`.

Nauvim shall perform the welcome-page transition and new connection attempt in accordance with 6.5. Its corresponding core functionality shall be available unconditionally under 9.1.

## Annex A — Configuration examples

**(informative)**

### A.1 General

This annex illustrates the configuration model specified in Clause 8 and the remaining provisional interfaces. Configuration details and defaults that remain provisional are subject to change during implementation. UI-specific option schemas are defined by the UI implementation under 8.10.

The example uses the `HostInfo` table returned by the `nauvim-client-info` module injected by `nauvim-core`. Its fields describe the client environment, and `client.config_file` is a non-nil path to the selected Nauvim configuration file.

The client-side account name comes from `client.user`. Its role is separate from that of the user name used to authenticate to the SSH server.

### A.2 Example configuration

```lua
local client = require("nauvim-client-info")

local home = client.home_dir

-- Illustrative identity-based socket name, stable across Nauvim restarts.
local client_socket_name = client.host_name
    .. "-"
    .. client.user
    .. ".sock"

return {
    default_host = "my-host",

    hosts = {
        ["local"] = {
            nvim = home .. "/bin/nvim",
            policies = {},
            socket = client.project_dirs.runtime_dir
                .. "/clients/"
                .. client_socket_name,
        },

        ["my-host"] = {
            nvim = function(host)
                return host.home_dir .. "/bin/nvim"
            end,

            ssh = {
                host_name = "my-host.com",
                user = "me",
                port = 2222,
                identity_file = home .. "/.ssh/my-host",
                identities_only = true,
            },

            policies = {},

            socket = function(host)
                return host.project_dirs.runtime_dir
                    .. "/clients/"
                    .. client_socket_name
            end,
        },
    },

    clipboard = "auto",

    ssh = {
        config = home .. "/.ssh/config",
        known_hosts = home .. "/.ssh/known_hosts",
    },
}
```

### A.3 Interpretation of the example

The `"local"` entry has no SSH settings and therefore designates a local host Neovim. The `"my-host"` entry includes SSH settings and is selected for automatic startup connection by the top-level `default_host` setting.

If `default_host` were `nil` in this example, the startup choices on the welcome page would be the two configured entries and the independent default Host. The configured entry named `"local"` would remain distinct from that default Host.

The local entry uses string-valued `nvim` and `socket` settings, so resolving those fields does not acquire host information. The separate client-information acquisition before configuration execution still applies.

The remote entry uses functions whose `host` parameter is a `HostInfo` obtained in the remote host environment. Its executable path uses the host's home directory, and its socket path uses the host's project runtime directory. In both functions, `host.config_file` is `nil`. The same host-role rule applies when a local entry uses function-valued settings.

Both entries derive their socket names from the client hostname and client-side user name. The remote socket function combines that client-derived name with a host-side directory. The concrete socket-path derivation remains unspecified, subject to the ownership and rediscovery requirements in 6.2.

The example also illustrates host-specific policy settings, global SSH file locations and automatic clipboard-backend detection. The top-level `clipboard = "auto"` explicitly selects the default backend-selection mode. The global SSH path defaults are specified in 8.9. Empty policy tables do not establish additional defaults or selection rules.

For `"my-host"`, the SSH lookup name is `"my-host.com"`. The example contains no native SSH entry with that name, so resolution reads `ssh.config` for `"my-host.com"` and uses it for supplementation under 8.5.4, preserving the explicitly supplied SSH fields. The concrete inheritance of OpenSSH-derived settings is unspecified. An error in that required OpenSSH-source operation fails resolution. The example does not designate a fallback for a missing or unreadable SSH configuration file.

### A.4 SSH merge example

The following excerpt illustrates native lookup and merging before supplementation from the OpenSSH configuration source:

```lua
return {
    hosts = {
        work = {
            ssh = {
                host_name = "base",
                user = "alice",
                identities_only = false,
                proxy_jump = {},
            },
        },
        base = {
            ssh = {
                host_name = "server.example.com",
                user = "bob",
                port = 2222,
                identities_only = true,
                proxy_jump = { "bastion" },
            },
        },
    },
}
```

For `work`, `H["work"]` supplies the highest-priority values and `H["base"]` supplies additional native settings. The settings denoted by `O("base")` are then used for supplementation under 8.5.4 before native defaults are applied; the concrete inheritance is unspecified. The resulting fields in Table A.1 retain their native values.

**Table A.1 — Results of the native SSH table merge**

| Setting | Result | Reason |
| --- | --- | --- |
| `host_name` | `"base"` | Explicit value in `work` |
| `user` | `"alice"` | Explicit value in `work` |
| `port` | `2222` | Inherited from `base` |
| `identities_only` | `false` | Explicit false value in `work` |
| `proxy_jump` | `{}` | Explicit empty list in `work` replaces the inherited list |

Resolving `work` does not recursively resolve the `base` table, read OpenSSH settings for `"server.example.com"`, or substitute that name for `"base"`. Its OpenSSH supplementation uses the lookup name `"base"`. Resolving `base` as a separate requested host is a separate operation under 8.5.3.

If an existing native SSH table omits `host_name`, its supplied alias is used for lookup. The native merge with that same table leaves its supplied settings unchanged. If settings remain absent, resolution uses the OpenSSH configuration for that alias under 8.5.4. Native defaults are applied last.

### A.5 Shallow table merge

The following generic tables illustrate the merge operation specified in 8.5.2:

```lua
local A = { a = 1, b = { c = 2 }, f = 5 }
local B = { b = { d = 3 }, f = { g = 6 } }

-- Result of merging A into B:
local R = { a = 1, b = { d = 3 }, f = { g = 6 } }
```

The field `a` is retained from `A` because `B` does not supply it. The entire `b` table is replaced, so `c` is not retained. The scalar value of `f` is replaced by the table supplied by `B`.

### A.6 Configuration module search root

Table A.2 illustrates the root selection specified in 8.1.1. The paths in the first column are the symbolic-link-resolved client-side `config_file` values.

**Table A.2 — Lua module search-path examples**

| Configuration file | `ROOT` | Templates prepended to `package.path` |
| --- | --- | --- |
| `/work/init.lua` | `/work` | `/work/lua/?.lua;/work/lua/?/init.lua` |
| `/work/custom.lua` | `/work` | `/work/lua/?.lua;/work/lua/?/init.lua` |
| `/work/config/init.lua` | `/work/config` | `/work/config/lua/?.lua;/work/config/lua/?/init.lua` |

For either of the first two configuration files, the added templates allow a lookup of `require("another.module")` to search `/work/lua/another/module.lua` and then `/work/lua/another/module/init.lua` before the previously configured Lua-file templates. The configuration filename does not affect the root selection.

For the third configuration file, the same lookup searches `/work/config/lua/another/module.lua` and then `/work/config/lua/another/module/init.lua` before the previously configured Lua-file templates.

### A.7 Neovim program specifications

The following alternatives illustrate the structured `NvimProg` form used as a host entry's `nvim` value.

An argument-bearing Neovim program specification can be written as:

```lua
nvim = {
    prog = "nvim",
    args = { "-O3" },
    env = {},
}
```

A shell-wrapper specification can be represented as:

```lua
nvim = {
    prog = "sh",
    args = { "-c", "some wrapper $@", "nvim" },
    env = {},
}
```

A host-information function can return a structured program specification:

```lua
nvim = function(host)
    return {
        prog = host.home_dir .. "/bin/nvim",
        args = { "-O3" },
        env = {},
    }
end
```

For the first example, Nauvim appends its required arguments to form the following argument sequence, where `socket` denotes the resolved socket setting:

```text
{ "-O3", "--headless", "--listen", socket }
```

For the shell-wrapper example, the corresponding sequence is:

```text
{ "-c", "some wrapper $@", "nvim", "--headless", "--listen", socket }
```

These examples illustrate configuration values and the required argument suffix. Subject to that suffix and the socket-argument requirements, their concrete expansion and execution interpretation, including the handling of the shell expression and environment entries, are unspecified as stated in 8.7.2.

### A.8 Clipboard backend configuration

The following configuration excerpt uses the table form to supply automatic backend selection and an explicit timeout:

```lua
return {
    hosts = {
        -- Host entries.
    },
    clipboard = {
        backend = "auto",
        timeout_ms = 1000,
    },
}
```

The timeout value in this example is illustrative and does not establish a default. Omitting `clipboard` or using `clipboard = "auto"` selects automatic detection under 8.8.1.

### A.9 UI-specific configuration

The following example assumes a UI implementation whose UI implementation ID is `"org.example.nauvim.flutter"` and whose option schema accepts the shown `keymaps`, `colorscheme` and `fonts` fields. The ID, field names, formats and values are illustrative; this document does not require support for this particular schema.

```lua
return {
    ui_opts = {
        ["org.example.nauvim.flutter"] = {
            keymaps = {
                { key = "CTRL_SHIFT_C", assign = "action.copy" },
            },
            colorscheme = {
                red = "rgb(1.0, 0.0, 0.0)",
            },
            fonts = {
                { family = "Example Mono", size = 16 },
            },
        },
    },
}
```

Because `use_ui_opts` is omitted, `nauvim-core` selects the entry using the ID supplied by `nauvim`. With the assumed ID `"org.example.nauvim.flutter"`, the selected dictionary contains the key mappings, colour scheme and font settings shown. An implementation supplying a different ID receives an empty dictionary from this example.

The selection key can instead be supplied explicitly:

```lua
return {
    ui_opts = {
        ["some-alias"] = {},
    },
    use_ui_opts = "some-alias",
}
```

Here, `nauvim-core` selects `ui_opts["some-alias"]` regardless of the supplied UI implementation ID. This allows a user-chosen key when UI implementations use conflicting IDs. The empty dictionary's interpretation is defined by the running UI implementation's schema. Removing the `"some-alias"` entry or setting it to nil also results in an empty dictionary; it does not cause lookup using the UI implementation ID.

## Annex B — Matters requiring further specification

**(informative)**

The matters in Table B.1 are not resolved by this draft. Their inclusion records open design decisions without selecting an implementation or adding requirements.

**Table B.1 — Open design decisions**

| Area | Matter to be specified |
| --- | --- |
| Security-policy configuration | Structure and meaning of host-specific policy settings, and handling of omitted or empty policy tables under 7.6 and 8.3 |

## Annex C — OpenSSH directive correspondences

**(informative)**

Table C.1 identifies the OpenSSH directives corresponding to the native `SshHostConf` fields. The directive names and meanings are documented in `ssh_config(5)` [22].

The table illustrates the intended direction of inheritance: for a setting absent from the native tables, an implementation can inherit some or all values configured by the corresponding directive, or some or all of its OpenSSH default values. The selection and method of inheritance are unspecified under 8.5.4. The table does not establish required directive coverage, value conversions or list-combination rules.

**Table C.1 — Informative mapping to OpenSSH directives**

| `SshHostConf` field | Corresponding OpenSSH directive |
| --- | --- |
| `host_name` | `HostName` |
| `user` | `User` |
| `port` | `Port` |
| `identity_file` | `IdentityFile` |
| `certificate_file` | `CertificateFile` |
| `identities_only` | `IdentitiesOnly` |
| `identity_agent` | `IdentityAgent` |
| `proxy_jump` | `ProxyJump` |
| `kex_algorithms` | `KexAlgorithms` |
| `host_key_algorithms` | `HostKeyAlgorithms` |

The mandatory terminal-allocation policy in 8.5.6 continues to apply: Nauvim ignores `RequestTTY` in the OpenSSH source and uses the equivalent of `RequestTTY no`.

## Bibliography

[1] Neovim. [Nvim API](https://neovim.io/doc/user/api/).

[2] Neovim. [Nvim UI protocol](https://neovim.io/doc/user/api-ui-events/).

[3] Neovim. [Nvim asynchronous IO](https://neovim.io/doc/user/channel/).

[4] Neovim. [Starting Nvim — Views and Sessions](https://neovim.io/doc/user/starting/#views-sessions).

[5] Neovim. [Nvim job control](https://neovim.io/doc/user/job_control/).

[6] IETF. [RFC 4254: The Secure Shell (SSH) Connection Protocol](https://www.rfc-editor.org/rfc/rfc4254.html).

[7] MessagePack-RPC. [MessagePack-RPC specification](https://github.com/msgpack-rpc/msgpack-rpc/blob/master/spec.md).

[8] Neovim. [Editing with multiple windows and buffers](https://neovim.io/doc/user/windows/).

[9] Lua.org. [Lua 5.4 Reference Manual](https://www.lua.org/manual/5.4/manual.html).

[10] freedesktop.org. [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir/).

[11] OpenSSH. [Protocol extensions — Unix domain socket forwarding](https://github.com/openssh/openssh-portable/blob/master/PROTOCOL), Section 2.4.

[12] IETF. [SSH File Transfer Protocol](https://www.ietf.org/archive/id/draft-ietf-secsh-filexfer-02.txt), Internet-Draft `draft-ietf-secsh-filexfer-02`. Informative description of SFTP operations; this draft specification does not select an SFTP version.

[13] Linux man-pages project. [path_resolution(7)](https://man7.org/linux/man-pages/man7/path_resolution.7.html).

[14] Linux man-pages project. [opendir(3)](https://man7.org/linux/man-pages/man3/opendir.3.html).

[15] OpenBSD. [getsockname(2)](https://man.openbsd.org/man2/getsockname.2).

[16] WHATWG. [URL Standard](https://url.spec.whatwg.org/).

[17] WHATWG. [Fetch Standard — CORS protocol](https://fetch.spec.whatwg.org/#http-cors-protocol).

[18] Neovim. [Providers — Clipboard integration and paste](https://neovim.io/doc/user/provider/), help tags `provider-clipboard`, `g:clipboard`, `provider-paste` and `ui-paste`.

[19] Neovim. [Automatic commands — UILeave](https://neovim.io/doc/user/autocmd/#UILeave).

[20] OWASP. [HTML5 Security Cheat Sheet — Cross Origin Resource Sharing](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html#cross-origin-resource-sharing).

[21] Neovim. [Lua engine — Importing Lua modules](https://neovim.io/doc/user/lua/#lua-module-load).

[22] OpenSSH. [ssh_config(5): OpenSSH client configuration file](https://man.openbsd.org/ssh_config).

[23] IETF. [RFC 3629: UTF-8, a transformation format of ISO 10646](https://www.rfc-editor.org/rfc/rfc3629.html), Section 6.

[24] IETF. [RFC 8252: OAuth 2.0 for Native Apps](https://www.rfc-editor.org/rfc/rfc8252.html#section-3), Section 3, definition of reverse domain name notation.
