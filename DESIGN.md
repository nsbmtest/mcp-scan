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
