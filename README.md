# Nauvim

A graphical user interface for Neovim, with persistent local and remote editing sessions.

The design keeps Neovim running independently of the GUI, so sessions can survive detachment and application restarts. Host-side Lua plugins can request client-side previews, port forwarding, temporary file copies and clipboard access.

This README is an informative guide to the [Nauvim specification](specification.md), version **1.0.0**. Specification versions follow [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html).

[Type notation](type-notation.md) defines the shared abstract types, representation rules and annotation syntax used by the specification.

The project separates a Rust RPC layer (`nvim-rpc`), the application engine (`nauvim-core`), a host-side Lua interface (`nauvim-plugin`) and the GUI application (`nauvim`). Flutter is the initial UI backend.

## Where to start

- **Understand the design:** read [Architecture](specification.md#architecture) and [Session management](specification.md#session-management).
- **Configure a client:** start with [Configuration](specification.md#configuration), then follow the examples in [Annex A](specification.md#configuration-examples).
- **Write or implement plugins:** read the [execution model](specification.md#plugin-interface-and-client-operations) and [common API rules](specification.md#common-api-rules), then use the module links below.

## Specification contents

| Section | What it covers |
| --- | --- |
| [1 Scope](specification.md#scope) | The features and components covered by the specification. |
| [2 Normative references](specification.md#normative-references) | External API and protocol documents used by specific requirements. |
| [3 Terms and definitions](specification.md#terms-and-definitions) | Host and client roles; sessions, connections, attachments, identifiers and resources. |
| [4 Architecture](specification.md#architecture) | Responsibilities of the four repositories and the boundary between the engine, plugin and UI backend. |
| [5 Environments and compatibility](specification.md#supported-environments-and-component-compatibility) | GNU/Linux and Apple Silicon macOS targets, Neovim support and core/plugin version compatibility. |
| [6 Session management](specification.md#session-management) | Startup, ownership, reattachment, host switching, shutdown, resource cleanup and connection recovery. |
| [7 Plugin interface and client operations](specification.md#plugin-interface-and-client-operations) | Client registration, operation without a selected destination, delegated client features and security. |
| [8 Configuration](specification.md#configuration) | Lua loading, host selection, host-specific security policies, SSH resolution, host information, program arguments, clipboard backends and UI-specific options. |
| [9 Build-time features and dependencies](specification.md#build-time-features-and-dependencies) | Cargo feature flags, optional functionality and dependencies assigned to each component. |
| [10 User interaction and rendering](specification.md#user-interaction-rendering-and-limitations) | Keyboard input, GUI mouse interaction, grid rendering and the scope of IME support. |
| [11 Nauvim plugin API](specification.md#nauvim-plugin-api) | Lua interfaces and RPC contracts, including asynchronous operations, resource handles, errors and cancellation. |
| [Annex A — Configuration examples](specification.md#configuration-examples) | Example Lua configurations, plugin permissions, shallow merging, module search paths, program specifications, clipboard settings and UI-specific options. |
| [Annex B — OpenSSH correspondences](specification.md#openssh-directive-correspondences) | An informative mapping from native SSH fields to OpenSSH directives. |
| [Bibliography](specification.md#bibliography) | Supporting Neovim, protocol, Lua and operating-system documentation. |

## Plugin API shortcuts

The [common rules](specification.md#common-api-rules) define `(ret, err)` results, asynchronous handles, client-ID and host policy checks, path confinement and idempotent handle operations. Module-specific contracts follow:

| Module | Operations |
| --- | --- |
| [`local-forward`](specification.md#local-forwarding-api) | Open TCP or Unix domain socket forwards, obtain an assigned TCP port and clear a forward. |
| [`filesystem`](specification.md#filesystem-api) | Copy host files into the client's temporary directory and cancel or remove copied resources. |
| [`http`](specification.md#http-plugin-api) | Serve files over HTTP(S), query the bound port and open or focus GUI windows for HTTP(S) URLs. |
| [`clipboard`](specification.md#clipboard-provider) | Supply a Neovim clipboard provider backed by the client system clipboard. |
| [`client`](specification.md#client-api) | Register the RPC destination, clear stored identifiers on `UILeave` and request a host switch. |

## Reading the requirements

The specification uses **shall** for requirements, **should** for recommendations, **may** for permissions and **can** for possibilities or capabilities. Notes, examples and annexes are informative.

Details marked **unspecified** or **implementation-dependent** are intentionally left open to implementations. **Undefined behaviour** identifies cases for which the specification imposes no behavioural requirements.

## Branch naming

Use `<kind>/<topic>` for branch names. Choose the kind by the meaning of the change:

| Kind | Use | Example |
| --- | --- | --- |
| `spec` | Add or change requirements, APIs, defaults or behaviour, or resolve specification contradictions. | `spec/rpc-response-validation` |
| `docs` | Improve wording, terminology, structure or links without changing the specification's meaning. | `docs/utf8-terminology` |
| `chore` | Change repository maintenance rules or settings. | `chore/agent-instructions` |

- Write a short, descriptive topic using lowercase English words separated by hyphens. Use a single `/` between the kind and topic.
- Classify the change by its meaning, not by the file being edited. A typo correction in `specification.md` uses `docs`; an API change accompanied by README updates uses `spec`.
- An issue number may precede the topic, for example `spec/42-session-recovery`. Issue numbers are optional.

Create a branch from `main` for each change and delete it after merging.
