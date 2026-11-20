# Nauvim

A graphical user interface for Neovim, with persistent local and remote editing sessions.

The design keeps Neovim running independently of the GUI, so sessions can survive detachment and application restarts. Host-side Lua plugins can request client-side previews, port forwarding, temporary file copies and clipboard access.

**Status: working draft.** This repository describes Nauvim's intended behaviour. [specification.md](specification.md) contains the requirements; this README is an informative reading guide.

The project separates a Rust RPC layer (`nvim-rpc`), the application engine (`nauvim-core`), a host-side Lua interface (`nauvim-plugin`) and the GUI application (`nauvim`). Flutter is the initial UI backend.

## Where to start

- **Understand the design:** read [Architecture](specification.md#4-architecture) and [Session management](specification.md#6-session-management).
- **Configure a client:** start with [Configuration](specification.md#8-configuration), then follow the examples in [Annex A](specification.md#annex-a--configuration-examples).
- **Write or implement plugins:** read the [execution model](specification.md#7-plugin-interface-and-client-operations) and [common API rules](specification.md#112-common-rules), then use the module links below.

## Specification contents

| Section | What it covers |
| --- | --- |
| [1 Scope](specification.md#1-scope) | The features and components covered by the draft. |
| [2 Normative references](specification.md#2-normative-references) | External API and protocol documents used by specific requirements. |
| [3 Terms and definitions](specification.md#3-terms-and-definitions) | Host and client roles; sessions, connections, attachments, identifiers and resources. |
| [4 Architecture](specification.md#4-architecture) | Responsibilities of the four repositories and the boundary between the engine, plugin and UI backend. |
| [5 Environments and compatibility](specification.md#5-supported-environments-and-component-compatibility) | GNU/Linux and Apple Silicon macOS targets, Neovim support and core/plugin version compatibility. |
| [6 Session management](specification.md#6-session-management) | Startup, ownership, reattachment, host switching, shutdown, resource cleanup and connection recovery. |
| [7 Plugin interface and client operations](specification.md#7-plugin-interface-and-client-operations) | Client registration, operation without a selected destination, delegated client features and security recommendations. |
| [8 Configuration](specification.md#8-configuration) | Lua loading, host selection, SSH resolution, host information, program arguments, clipboard backends and UI-specific options. |
| [9 Build-time features and dependencies](specification.md#9-build-time-features-and-dependencies) | Cargo feature flags, optional functionality and dependencies assigned to each component. |
| [10 User interaction and rendering](specification.md#10-user-interaction-rendering-and-limitations) | Keyboard input, GUI mouse interaction, grid rendering and the scope of IME support. |
| [11 Nauvim plugin API](specification.md#11-nauvim-plugin-api) | Lua interfaces and RPC contracts, including asynchronous operations, resource handles, errors and cancellation. |
| [Annex A — Configuration examples](specification.md#annex-a--configuration-examples) | Example Lua configurations, shallow merging, module search paths, program specifications, clipboard settings and UI-specific options. |
| [Annex B — Open decisions](specification.md#annex-b--matters-requiring-further-specification) | The remaining decisions about host-specific security-policy configuration. |
| [Annex C — OpenSSH correspondences](specification.md#annex-c--openssh-directive-correspondences) | An informative mapping from native SSH fields to OpenSSH directives. |
| [Bibliography](specification.md#bibliography) | Supporting Neovim, protocol, Lua and operating-system documentation. |

## Plugin API shortcuts

The [common rules](specification.md#112-common-rules) define `ret, err` results, asynchronous handles, client-ID checks, path confinement and idempotent handle operations. Module-specific contracts follow:

| Module | Operations |
| --- | --- |
| [`local-forward`](specification.md#113-local-forwarding) | Open TCP or Unix domain socket forwards, obtain an assigned TCP port and clear a forward. |
| [`filesystem`](specification.md#114-filesystem-operations) | Copy host files into the client's temporary directory and cancel or remove copied resources. |
| [`http`](specification.md#115-http-services-and-gui-access) | Serve files over HTTP(S), query the bound port and open a GUI window for an HTTP(S) URL. |
| [`clipboard`](specification.md#116-clipboard-provider) | Supply a Neovim clipboard provider backed by the client system clipboard. |
| [`client`](specification.md#117-client-registration-rpc-destination-selection-and-session-switching) | Register the RPC destination, clear stored identifiers on `UILeave` and request a host switch. |

## Reading the requirements

The specification uses **shall** for requirements, **should** for recommendations, **may** for permissions and **can** for possibilities or capabilities. Notes, examples and the three annexes are informative.

Details marked **unspecified** or **implementation-dependent** are intentionally left open to implementations. They are distinct from the design decisions still awaiting specification in Annex B. **Undefined behaviour** identifies cases for which the specification imposes no behavioural requirements.

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
