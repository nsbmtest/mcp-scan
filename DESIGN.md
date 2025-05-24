# Design Document

## 1. Introduction

MCP-Scan is a security scanning tool designed to both statically and dynamically scan and monitor your installed MCP servers and check them for common security vulnerabilities like prompt injections, tool poisoning and cross-origin escalations.

It can scan Claude, Cursor, Windsurf, and other file-based MCP client configurations. MCP-Scan also features runtime monitoring of MCP traffic using a proxy, enforces guardrailing policies (PII detection, secrets detection, tool restrictions, custom policies), detects cross-origin escalation attacks, and implements tool pinning to prevent MCP rug pull attacks.

MCP-Scan works by searching through configuration files to find MCP server configurations. It connects to these servers, retrieves tool descriptions, and then scans these descriptions for vulnerabilities. This scanning process involves both local checks and invoking Invariant Guardrailing via an API. For runtime monitoring, MCP-Scan can be used as a proxy server, temporarily injecting a local Invariant Gateway into MCP server configurations to intercept and analyze traffic.

The proxy functionality allows for real-time monitoring and guardrailing of system-wide MCP traffic. It can enforce various security policies, such as PII detection, secrets detection, tool restrictions, and custom guardrailing policies. This proxying and guardrailing operate entirely locally and do not require external API calls.

During scanning, tool names and descriptions are shared with invariantlabs.ai for security research purposes, as per the terms of use and privacy policy. However, MCP-Scan can be run in a local-only mode using the `--local-only` flag, which performs checks without invoking the Invariant Guardrailing API. This local mode requires an `OPENAI_API_KEY` environment variable. MCP-Scan does not store or log any usage data, such as the contents and results of MCP tool calls.

MCP-Scan provides several CLI commands for its operations: `scan` (default) for security vulnerability scanning, `proxy` for real-time traffic monitoring and guardrailing, `inspect` for printing descriptions without verification, `whitelist` for managing approved entities, and `help` for detailed information. Common options like `--storage-file`, `--base-url`, `--verbose`, `--print-errors`, and `--json` are available across commands.

## 2. Features

MCP-Scan offers a comprehensive suite of security functionalities for MCP environments. Key features include:

*   **Broad Configuration Scanning**: Scans various file-based MCP client configurations, including Claude, Cursor, and Windsurf.
*   **Vulnerability Detection**:
    *   Identifies prompt injection attacks in tools.
    *   Detects tool poisoning attacks by leveraging Guardrails.
*   **Runtime Monitoring and Auditing**: Audits MCP calls in real-time through runtime monitoring of MCP traffic using the `mcp-scan proxy` command.
*   **Policy Enforcement**: Enforces guardrailing policies on tool calls and responses. This includes:
    *   PII (Personally Identifiable Information) detection.
    *   Secrets detection.
    *   Tool restrictions.
    *   Support for entirely custom guardrailing policies.
*   **Attack Vector Mitigation**:
    *   Detects cross-origin escalation attacks, such as tool shadowing.
    *   Implements "tool pinning" by hashing MCP tools to detect changes and prevent MCP rug pull attacks.

## 3. Goals and Non-Goals

## 4. Architecture
## 4. Architecture

MCP-Scan is designed as a modular command-line tool with components for scanning, proxying, and policy enforcement. It also includes a server component, though the primary functionalities described in the README are CLI-driven.

The main components are:

*   **Command Line Interface (CLI - `src/mcp_scan/cli.py`):**
    *   Serves as the main entry point for the user.
    *   Parses command-line arguments (`scan`, `proxy`, `inspect`, `whitelist`, `help`).
    *   Orchestrates the workflow by invoking the appropriate modules based on the command.

*   **MCP Scanner (`src/mcp_scan/MCPScanner.py`):**
    *   Handles the core logic for the `scan` command.
    *   Identifies MCP server configurations.
    *   Utilizes `mcp_client.py` to connect to these servers and retrieve tool descriptions.
    *   Analyzes tool descriptions for vulnerabilities against defined policies (from `policy.gr`) and potentially via the Invariant Guardrailing API.

*   **MCP Client (`src/mcp_scan/mcp_client.py`):**
    *   Manages communication with the target MCP servers.
    *   Fetches tool descriptions and other necessary information from MCP instances.

*   **Proxy Engine (`src/mcp_scan/gateway.py` and `cli.py` proxy command):**
    *   Implements the `proxy` functionality.
    *   Intercepts HTTP/HTTPS traffic from MCP clients.
    *   Injects a local Invariant Gateway (`gateway.py`) to monitor and analyze MCP traffic in real-time.
    *   Enforces guardrailing policies (PII detection, secrets, tool restrictions) on the intercepted traffic.

*   **Policy and Guardrailing (`src/mcp_scan/policy.gr`, guardrailing logic):**
    *   Defines the security rules and policies used by both the scanner and the proxy.
    *   `policy.gr` likely contains static analysis policies.
    *   Runtime guardrailing policies (PII, secrets, etc.) are applied by the proxy engine.

*   **Storage:**
    *   Manages the persistence of data such as scan results, whitelisted items, and potentially configuration settings.
    *   The README mentions a `StorageFile` for this purpose.

*   **MCP Scan Server (`src/mcp_scan_server/`):**
    *   A separate server component (`server.py` with routes in `routes/`).
    *   Its exact role in conjunction with the CLI tool's `scan` and `proxy` features is not fully detailed in the README's overview but likely provides an API or a centralized management interface.
    *   Includes `guardrail_templates/` which might be used by the server for policy management or generation.

**Interaction Flow (Scan command):**
1.  User executes `mcp-scan scan ...` via the CLI.
2.  `cli.py` parses arguments and instantiates `MCPScanner.py`.
3.  `MCPScanner.py` discovers MCP server configurations.
4.  For each server, `MCPScanner.py` uses `mcp_client.py` to fetch tool descriptions.
5.  `MCPScanner.py` applies policies from `policy.gr` and/or Invariant Guardrailing API to the descriptions.
6.  Results are stored using the Storage component and presented to the user.

**Interaction Flow (Proxy command):**
1.  User executes `mcp-scan proxy ...` via the CLI.
2.  `cli.py` sets up the proxy server, configuring it to use `gateway.py` as the Invariant Gateway.
3.  MCP client applications are configured to route traffic through this local proxy.
4.  The proxy engine intercepts traffic, and `gateway.py` inspects it against runtime guardrailing policies.
5.  Actions (allow, block, modify, log) are taken based on policy violations.

## 5. Key Components

This section provides a more detailed description of each key component identified in the Architecture section, including its functionality and responsibilities.

*   **CLI (`src/mcp_scan/cli.py`)**:
    *   **Functionality**: Serves as the primary user interface for MCP-Scan. It's responsible for parsing command-line arguments, validating inputs, and initiating the appropriate actions based on the user's commands (e.g., `scan`, `proxy`, `inspect`, `whitelist`).
    *   **Responsibilities**:
        *   Manages different command modes and their specific options.
        *   Orchestrates the overall workflow by initializing and invoking other components like the `MCPScanner` or the proxy setup.
        *   Handles user feedback, outputting results, errors, and verbose information as specified by the user.
        *   Controls global configurations like storage file location, API base URLs, and output formats.

*   **MCP Scanner (`src/mcp_scan/MCPScanner.py`)**:
    *   **Functionality**: Core component for performing static and dynamic security assessments of MCP servers. It systematically identifies and analyzes MCP configurations and their associated tools.
    *   **Responsibilities**:
        *   Discovers MCP server instances by scanning configuration files (e.g., for Claude, Cursor, Windsurf).
        *   Utilizes the `MCPClient` to fetch tool descriptions and specifications from these servers.
        *   Applies a set of local security checks based on `policy.gr` to identify common vulnerabilities (e.g., overly permissive tool descriptions, known insecure patterns).
        *   Optionally interacts with the Invariant Guardrailing API (invariantlabs.ai) for more advanced vulnerability analysis and to leverage a broader knowledge base of threats.
        *   Manages the `--local-only` mode to perform checks without external API calls, relying on local policies and an `OPENAI_API_KEY` for certain local checks.
        *   Reports findings, including vulnerabilities and recommendations, to the user and saves them via the Storage component.

*   **MCP Client (`src/mcp_scan/mcp_client.py`)**:
    *   **Functionality**: Acts as an abstraction layer for communicating with various MCP servers. It handles the specifics of connecting to different MCP implementations and retrieving necessary data.
    *   **Responsibilities**:
        *   Establishes connections to MCP servers based on discovered configurations.
        *   Retrieves tool descriptions, manifests, and other relevant metadata from the MCP servers.
        *   Handles authentication or any specific communication protocols required by the MCP servers.
        *   Provides a standardized interface for the `MCPScanner` to interact with diverse MCP environments.

*   **Proxy Engine (`src/mcp_scan/gateway.py` and `src/mcp_scan_server/server.py` for guardrailing aspects)**:
    *   **Functionality**: Implements the real-time monitoring and guardrailing of MCP traffic. It intercepts communication between MCP clients and servers to enforce security policies.
    *   **Responsibilities**:
        *   Sets up a local HTTP/HTTPS proxy server that MCP client applications can be configured to use.
        *   Intercepts all MCP-related traffic passing through it.
        *   Utilizes `gateway.py` (the Invariant Gateway) to inspect requests and responses in real-time.
        *   Applies guardrailing policies, which can be sourced from `src/mcp_scan_server/guardrail_templates/` and managed by `src/mcp_scan_server/server.py`. These policies include PII detection, secret detection, tool usage restrictions, and custom policy enforcement.
        *   Detects and potentially blocks cross-origin escalation attacks and rug pull attacks (tool pinning).
        *   Logs traffic and policy violations, providing visibility into runtime behavior.
        *   The `server.py` component likely provides the backend logic for managing and applying these dynamic guardrails, possibly serving policy configurations to the `gateway.py`.

    *   **Proxy Engine Code Flow (`mcp-scan proxy` command)**:
        This details the step-by-step execution when the `mcp-scan proxy` command is invoked:

        1.  **Execution of `mcp-scan proxy`**:
            *   The user runs `mcp-scan proxy` from the command line.
            *   `src/mcp_scan/cli.py` parses the command and its arguments, initiating the proxy setup sequence. This is primarily handled by the `proxy()` function within `cli.py`.

        2.  **The `install()` Sequence (via `cli.py#proxy` calling `MCPGatewayInstaller#install_all`)**:
            *   An `MCPGatewayInstaller` instance is created (from `src/mcp_scan/gateway.py`).
            *   The `install_all()` method is called, which iterates through discovered MCP configurations (e.g., Claude JSON files).
            *   For each supported MCP configuration, the `install_gateway()` method of `MCPGatewayInstaller` is invoked.
            *   **Configuration Modification**: `install_gateway()` modifies the MCP JSON configuration files. Specifically, it targets `stdio_server` commands.
            *   **Command Wrapping**: The original `stdio_server` command (e.g., `["python", "mcp_server.py"]`) is wrapped. The new command becomes `["invariant-gateway", "--exec", "python", "mcp_server.py"]` (or similar, depending on the original command structure). This ensures that when the MCP client starts its server, it actually starts `invariant-gateway`, which then executes the original server command as a subprocess.
            *   **`invariant-gateway` Configuration**: As part of the modification, `invariant-gateway` is configured to communicate with the local `MCPScanServer`. This is achieved by setting the `invariant_api_url` (or an equivalent mechanism for `invariant-gateway`) to point to the local address where `MCPScanServer` will listen (e.g., `http://localhost:PORT`). Original configuration files are backed up before modification.

        3.  **Startup of Local `MCPScanServer` (via `cli.py#server`)**:
            *   Concurrently with or after the installation, `cli.py`'s `proxy()` function calls its `server()` function (or a similar utility) to start the local `MCPScanServer`.
            *   `src/mcp_scan_server/server.py` is executed/imported.
            *   The `MCPScanServer` (an HTTP server, likely based on Flask or a similar framework) starts listening on a specific local port (e.g., 8080).
            *   On startup, the `MCPScanServer` loads local guardrail policies. These policies might be defined in `src/mcp_scan_server/guardrail_templates/` or other configuration files. These policies will be served to `invariant-gateway` instances upon request.
            *   The `MCPScanServer` also registers an `on_exit` hook (e.g., using `atexit` or signal handling) which will trigger the `uninstall()` sequence.

        4.  **Role of `invariant-gateway`**:
            *   When an MCP client application (e.g., Claude, Cursor) starts its associated MCP server, it now unknowingly executes `invariant-gateway` due to the modified configuration.
            *   `invariant-gateway` (which is an external tool, not part of MCP-Scan's direct codebase but relied upon) intercepts all standard input/output (stdin/stdout) communication between the MCP client and the (now child process) MCP server.
            *   Before passing data through, `invariant-gateway` communicates with the local `MCPScanServer` (at the configured `invariant_api_url`) to fetch the relevant guardrailing policies.
            *   Based on these policies, `invariant-gateway` inspects the MCP traffic (tool calls, responses). It can then allow, block, or modify the traffic according to the policies.
            *   It executes the original MCP server command (e.g., `python mcp_server.py`) as a subprocess, managing its lifecycle and proxying data to/from it.

        5.  **The `uninstall()` Sequence (via `cli.py#proxy` calling `MCPGatewayInstaller#uninstall_all` on exit)**:
            *   When the `mcp-scan proxy` command is terminated by the user (e.g., Ctrl+C), the `on_exit` hook registered by `MCPScanServer` (or `cli.py`) is triggered.
            *   This hook calls the `uninstall_all()` method of the `MCPGatewayInstaller` instance.
            *   `uninstall_gateway()` is invoked for each modified MCP configuration.
            *   This method reverts the changes made to the MCP JSON configuration files, restoring them from the backups created during the `install()` phase. This ensures that MCP clients will resume their normal operation without `invariant-gateway` once `mcp-scan proxy` is stopped.

    *   **Proxy Engine UML Diagrams**:

        ```plantuml
        @startuml
        title mcp-scan proxy Lifecycle and Request Interception

        actor User
        participant "mcp-scan CLI" as CLI
        participant "MCPGatewayInstaller" as Installer
        database "MCP JSON Configs" as Configs
        participant "MCPScanServer (local)" as LocalServer
        participant "invariant-gateway (proxy)" as IGProxy
        participant "Original MCP Server App" as MCPServerApp
        actor Client

        User -> CLI: Executes `mcp-scan proxy`
        CLI -> Installer: install()
        Installer -> Configs: Reads and modifies (wraps command with invariant-gateway)
        CLI -> LocalServer: Starts server
        LocalServer -> LocalServer: Loads guardrails
        User -> MCPServerApp: Starts/restarts MCP Server App (now runs via invariant-gateway)
        Client -> IGProxy: Sends request
        IGProxy -> LocalServer: Requests policies
        LocalServer --> IGProxy: Returns policies
        IGProxy -> IGProxy: Applies policies
        alt Request Allowed
          IGProxy -> MCPServerApp: Forwards request
          MCPServerApp --> IGProxy: Processes and responds
          IGProxy --> Client: Forwards response
        else Request Blocked/Modified
          IGProxy --> Client: Returns error or modified response
        end

        User -> CLI: Stops `mcp-scan proxy` (e.g., Ctrl+C)
        CLI -> Installer: uninstall() (via on_exit hook)
        Installer -> Configs: Reads and reverts to original
        @enduml
        ```

        ```plantuml
        @startuml
        title Proxy Mode Components

        package "User/Client System" {
          actor User
          actor ClientApp as "MCP Client Application"
        }

        package "MCP-Scan System" {
          component "mcp-scan CLI" as CLI {
            portin "proxy command"
          }
          component "MCPGatewayInstaller" as Installer
          database "MCP JSON Configs" as Configs
          component "MCPScanServer (local)" as LocalServer {
            port "Guardrail Policy API (e.g., 8129)" as PolicyAPI
          }
        }

        package "Proxied MCP Environment" {
          component "invariant-gateway (proxy)" as IGProxy {
            portin "Network Interception"
          }
          component "Original MCP Server App" as MCPServerApp
        }

        User --> CLI : "proxy command"
        CLI --> Installer : uses
        Installer ..> Configs : modifies
        CLI --> LocalServer : starts & manages

        ClientApp ..> IGProxy : (unawarely) connects to
        note on link
          Original connection
          to MCPServerApp
          is redirected by
          modified config
        end note

        IGProxy --> LocalServer : "Guardrail Policy API"
        IGProxy ..> MCPServerApp : manages/proxies (e.g. stdio/local network)

        @enduml
        ```

    *   **Proxy Engine Networking Details**:
        This subsection clarifies the networking aspects of how `mcp-scan`'s proxy mode operates, primarily by configuring and launching the `invariant-gateway` tool. `mcp-scan` itself does not perform direct network proxying of MCP traffic; instead, it sets up `invariant-gateway` to act as the intermediary.

        1.  **Connection Management for `invariant-gateway`**:
            *   **Primary Listener**: Once `mcp-scan proxy` modifies an MCP's configuration (e.g., a Claude JSON file) and that MCP server is started, `invariant-gateway` becomes the primary process that the MCP client application interacts with. The original server command (e.g., `["python", "mcp_server.py"]`) is replaced with a command that invokes `invariant-gateway` first, like `["uvx", "invariant-gateway", "--exec", "python", "mcp_server.py"]`.
            *   **Handling Original Communication Channel**: Because `invariant-gateway` is started via `uvx` (or `uv run`) and takes the original server command via its `--exec` argument, `invariant-gateway` takes control of the communication channel that the `Original MCP Server App` would have used. For many MCP tools that use `stdio_server` configurations, this means `invariant-gateway` intercepts the standard input (stdin) and standard output (stdout) streams. If the original server were designed to listen on a network socket, `invariant-gateway` would need to bind to that socket itself.
            *   **Connection to Original Server**: After intercepting the client-side communication, `invariant-gateway` then executes the `Original MCP Server App` as a child process. It establishes its own connection to this child process, typically by piping to the child's stdin and reading from its stdout.

        2.  **Proxy Positioning**:
            *   **Configuration Modification**: As detailed in the "Proxy Engine Code Flow", `mcp-scan` (specifically the `MCPGatewayInstaller`) modifies the MCP's JSON configuration file. The command to start the MCP server is altered to prepend `invariant-gateway`.
            *   **Transparent Interception**: When the MCP client (e.g., an IDE extension, a command-line tool that uses the MCP) attempts to start its configured MCP server, it unknowingly launches `invariant-gateway`.
            *   **Middleman Role**: `invariant-gateway` positions itself as a "man-in-the-middle."
                *   To the MCP Client: `invariant-gateway` appears to be the `Original MCP Server App`.
                *   To the `Original MCP Server App`: `invariant-gateway` appears to be the MCP client.

        3.  **Communication with Local `MCPScanServer`**:
            *   **API URLs for Policies**: When `mcp-scan proxy` launches `invariant-gateway`, it configures environment variables or command-line arguments for `invariant-gateway`. These include `INVARIANT_API_URL` and `GUARDRAILS_API_URL` (or similar, depending on `invariant-gateway`'s specific configuration options).
            *   **Local HTTP Requests**: These URLs are set to point to the `MCPScanServer` instance that `mcp-scan proxy` starts locally (e.g., `http://localhost:8129`, where 8129 is the default port for `MCPScanServer`).
            *   **Policy Fetching and Logging**: `invariant-gateway` uses these URLs to make local HTTP requests to the `MCPScanServer`. These requests are primarily to:
                *   Fetch the applicable guardrailing policies.
                *   Potentially send back logs or metadata about intercepted activity for `MCPScanServer` to record or display.

        4.  **Protocols Involved**:
            *   **MCP Client <-> `invariant-gateway`**: The protocol used here is dictated by what the `Original MCP Server App` expects.
                *   For `stdio_server` configurations: This is typically communication over stdin/stdout pipes.
                *   For MCP servers that listen on network sockets: This would be TCP (or UDP, though less common for MCPs). `invariant-gateway` would need to mimic the original server's network behavior.
            *   **`invariant-gateway` <-> `Original MCP Server App`**: Given the `--exec` model, this communication is almost always via **stdio** (stdin/stdout pipes). `invariant-gateway` starts the original server as a subprocess and communicates with it using these standard streams. Even if the original server was a network server, `invariant-gateway` effectively "localizes" its communication to stdio for the purpose of interception.
            *   **`invariant-gateway` <-> `MCPScanServer (local)`**: This communication is over **HTTP**. `MCPScanServer` runs an HTTP server, and `invariant-gateway` acts as an HTTP client to fetch policies and send data.

    *   **Proxy Interjection Mechanism**:
        This subsection details how `mcp-scan` modifies MCP configuration files to insert the `invariant-gateway` as a proxy. The core logic for this is found within `src/mcp_scan/gateway.py`, specifically the `MCPGatewayInstaller` class and its `install_gateway` and `uninstall_gateway` methods.

        1.  **Configuration Files Targeted**:
            *   `mcp-scan` targets MCP JSON configuration files that define how MCP clients launch their associated servers. Examples include `~/.cursor/cursor-settings.json`, VSCode's `settings.json` (for relevant extensions), and configuration files for Claude instances.
            *   The tool searches a default set of paths defined in `WELL_KNOWN_MCP_PATHS` (from `src/mcp_scan/paths.py`). Users can also specify custom paths to MCP configuration files via command-line arguments.

        2.  **Role of `MCPGatewayInstaller`**:
            *   The `MCPGatewayInstaller` class in `src/mcp_scan/gateway.py` is responsible for orchestrating the modification and restoration of these configuration files.
            *   It identifies compatible MCP server definitions within the JSON files and applies the necessary changes to inject `invariant-gateway`.

        3.  **Modification of `StdioServer` Entries**:
            *   The primary target for interjection are MCP servers defined as `StdioServer`. For these entries, `mcp-scan` alters the `command` and `args` fields to prepend the `invariant-gateway` execution.
            *   **Conceptual "Before" Example**:
                ```json
                {
                  "mcp_server_name": {
                    "type": "StdioServer",
                    "command": "python",
                    "args": ["my_mcp_server.py"],
                    "env": {}
                  }
                }
                ```
            *   **Conceptual "After" Example (actual arguments might vary slightly based on `invariant-gateway` version and `mcp-scan` configuration)**:
                ```json
                {
                  "mcp_server_name": {
                    "type": "StdioServer",
                    "command": "uvx",
                    "args": [
                      "invariant-gateway@latest",
                      "mcp",
                      // Example gateway configuration arguments:
                      // "--project-name", "some_project",
                      // "--api-key", "user_api_key_if_needed",
                      "--exec",
                      "python",
                      "my_mcp_server.py"
                    ],
                    "env": {
                      // Environment variables like INVARIANT_API_KEY,
                      // INVARIANT_API_URL (pointing to local MCPScanServer),
                      // and GUARDRAILS_API_URL are added here by MCPGatewayInstaller.
                    }
                  }
                }
                ```
            *   The `command` is changed to `uvx` (or `uv` if a specific `source_dir` for `invariant-gateway` is used) to invoke the `invariant-gateway` tool. The original command and its arguments are then appended after the `--exec` flag.
            *   Crucially, `MCPGatewayInstaller` also injects environment variables (`env` block) necessary for `invariant-gateway` to communicate with the locally running `MCPScanServer` (for policies) and potentially the Invariant platform (if not in local-only mode).

        4.  **Execution Flow Change**:
            *   When an MCP client (e.g., an IDE extension, a chatbot UI) reads this modified configuration, it will now execute the `uvx invariant-gateway ...` command instead of directly launching the original server (e.g., `python my_mcp_server.py`).
            *   `invariant-gateway`, upon starting, takes control and then, as per its `--exec` argument, launches the original `python my_mcp_server.py` as a child process. All communication (stdin/stdout for `StdioServer`) is now routed through `invariant-gateway`.

        5.  **Temporary Nature for `mcp-scan proxy` vs. Persistent for `mcp-scan install`**:
            *   When using the `mcp-scan proxy` command, these modifications to MCP configuration files are temporary. The `MCPGatewayInstaller`'s `uninstall_all()` method is registered to run when `mcp-scan proxy` exits (e.g., on Ctrl+C). This method restores the original configuration files from backups made during the installation phase.
            *   If the `mcp-scan install` command is used (assuming such a command exists for persistent installation, as implied by the need for `uninstall`), the changes would remain in place until a corresponding `mcp-scan uninstall` command is explicitly run.

        6.  **Non-Support for `SSEServer`**:
            *   Currently, the gateway interjection mechanism specifically targets `StdioServer` configurations. `SSEServer` (Server-Sent Events server) configurations are not supported for this type of proxying. `mcp-scan` will typically skip these entries during the installation process.

*   **Policy and Guardrailing (`src/mcp_scan/policy.gr`, `src/mcp_scan_server/guardrail_templates/`)**:
    *   **Functionality**: Defines the set of rules, checks, and security policies that MCP-Scan uses to identify vulnerabilities and enforce desired behavior.
    *   **Responsibilities**:
        *   `policy.gr`: Contains rules for static analysis performed by the `MCPScanner`. These rules likely define patterns of vulnerabilities in tool descriptions, dangerous function calls, or insecure configurations.
        *   `src/mcp_scan_server/guardrail_templates/`: Provides templates for dynamic guardrailing policies applied by the Proxy Engine. These templates might define how to detect PII, secrets, or restrict access to certain tools or functionalities at runtime.
        *   The overall system ensures that these policies are loaded and applied correctly during both scanning and proxying operations.

*   **Storage (`src/mcp_scan/StorageFile.py`)**:
    *   **Functionality**: Manages the persistence of data generated and used by MCP-Scan.
    *   **Responsibilities**:
        *   Saves and retrieves scan results, allowing users to review findings from previous scans.
        *   Manages whitelists of approved tools, servers, or specific configurations that should be excluded from certain checks.
        *   Potentially stores other application settings or cached data.
        *   Provides a consistent mechanism for reading from and writing to a designated storage file (e.g., a JSON file).

*   **MCP Scan Server (`src/mcp_scan_server/`)**:
    *   **Functionality**: Provides a backend API and logic that complements the CLI tool, particularly for advanced features like dynamic guardrailing and potentially centralized policy management.
    *   **Responsibilities**:
        *   Hosts an API (as defined in `src/mcp_scan_server/routes/`) that could be used by the CLI, the proxy, or external systems.
        *   Manages and serves guardrail templates (`src/mcp_scan_server/guardrail_templates/`) used by the Proxy Engine for runtime policy enforcement.
        *   May handle interactions with the Invariant Guardrailing API or implement a local version of its capabilities, especially for complex checks or policy evaluations that are better suited for a server environment.
        *   Could potentially support features like centralized reporting, user management, or more complex policy configurations in future extensions.

## 6. Proposed Design

## 7. Alternatives Considered

## 8. Future Work

This section outlines potential areas for future development and enhancement of MCP-Scan:

*   **Expanded Protocol and Platform Support**:
    *   Extend scanning capabilities beyond file-based MCP configurations to support other MCP protocols and platforms (e.g., database-backed configurations, proprietary MCP setups, cloud-based MCP services).
    *   Support for a wider range of MCP client applications and frameworks.

*   **More Sophisticated Static Analysis**:
    *   Incorporate more advanced static analysis techniques to detect a broader range of vulnerabilities in tool descriptions and configurations. This could include data flow analysis, taint analysis, and more complex pattern matching.
    *   Develop a more extensible framework for adding new static analysis rules and checks.

*   **Enhanced Machine Learning Models**:
    *   Improve the machine learning models used for threat detection (e.g., in the Invariant Guardrailing API or local equivalent) to identify novel and evolving attack patterns with greater accuracy.
    *   Explore unsupervised learning techniques to detect anomalies and suspicious activities in MCP traffic.

*   **Integration with Security Ecosystem**:
    *   Develop integrations with other security tools and platforms, such as SIEM (Security Information and Event Management) systems, SOAR (Security Orchestration, Automation and Response) platforms, and vulnerability management dashboards.
    *   Provide APIs or webhooks for easier integration into existing security workflows.

*   **Comprehensive Management Dashboard**:
    *   Create a web-based dashboard for visualizing scan results, managing policies, tracking remediation efforts, and viewing historical security posture.
    *   The existing `src/mcp_scan_server/` could be expanded to support this, providing a richer user experience beyond the CLI.

*   **Automated Remediation Assistance**:
    *   Introduce capabilities for automated or semi-automated remediation of certain detected vulnerabilities. This could include:
        *   Suggesting specific configuration changes.
        *   Automatically updating whitelists or policies based on trusted patterns.
        *   Generating patches or scripts to fix common issues.

*   **Improved Policy Management**:
    *   Enhance the flexibility and usability of policy definition and management, perhaps through a dedicated policy editor or a more expressive policy language.
    *   Support for versioning and auditing of policies.

*   **Community Contributions and Extensibility**:
    *   Develop a clearer plugin architecture to allow the community to contribute new scanner modules, policy checks, and integrations.
