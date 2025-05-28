## Playwright Browser Instance and Context Management

This document details how browser instances and contexts are managed across different Playwright browser implementations within the `magentic_ui` toolset.

### 1. `PlaywrightBrowser` Abstract Base Class

The `PlaywrightBrowser` class serves as the abstract base class (ABC) for all Playwright browser implementations. It defines the core interface and lifecycle methods that concrete browser classes must implement.

**Key Responsibilities:**

- **Defines the contract:** It establishes the methods and properties that all browser types will have, such as `_start()`, `_close()`, `__aenter__`, `__aexit__`, and the `browser` & `context` properties.
- **Abstract methods:** It declares `_start_browser_if_needed()` and `_close_browser_if_needed()` as abstract methods, forcing subclasses to provide specific implementations for starting and stopping the browser.
- **Asynchronous context management:** It implements `__aenter__` and `__aexit__` to allow the browser instances to be used as asynchronous context managers (e.g., `async with Browser() as browser:`).
    - `__aenter__`: Calls `_start_browser_if_needed()` to ensure the browser is running and returns the instance itself.
    - `__aexit__`: Calls `_close_browser_if_needed()` to clean up resources when the context is exited.
- **`browser` property:** This property is responsible for returning an active Playwright `Browser` instance. It calls `_start_browser_if_needed()` if the browser isn't already running.
- **`context` property:** This property is responsible for returning an active Playwright `BrowserContext` instance. It ensures the browser is started and then creates or returns an existing context. The management of this context (e.g., persistence) is delegated to subclasses.

### 2. `LocalPlaywrightBrowser`

The `LocalPlaywrightBrowser` class manages a browser instance that runs directly on the local machine.

**Key Features and Management:**

- **Initialization:**
    - `headless`: A boolean indicating whether to run the browser in headless mode (no UI) or headful mode.
    - `persistent_context_path`: An optional `Path` to a directory where the browser context (cookies, local storage, etc.) should be persisted. If provided, a persistent context is created; otherwise, a new incognito-like context is created each time.
    - `channel`: An optional string specifying the browser channel to use (e.g., "chrome", "msedge", "chrome-beta"). This allows launching specific browser executables installed on the system.
- **Lifecycle Management:**
    - `_start_browser_if_needed()`:
        - Checks if `_browser` is already initialized. If not:
            - It retrieves the Playwright browser type (chromium, firefox, webkit) based on the `browser_type` attribute (defaults to chromium).
            - If a `channel` is specified, it uses `browser_type.launch_persistent_context()` if `persistent_context_path` is also given, or `browser_type.launch()` otherwise.
            - If no `channel` is specified, it uses `browser_type.launch_persistent_context()` with the `persistent_context_path` (if provided) and `headless` status, or `browser_type.launch()` with the `headless` status.
            - If a persistent context is launched, the `_browser` is set to the browser instance returned by the context, and `_context` is set to this persistent context.
            - If a regular browser is launched, `_browser` is set, and `_context` remains `None` initially.
    - `_close_browser_if_needed()`:
        - If `_context` exists and is persistent (has `_owner == _browser`), it closes the context first. This is important for persistent contexts launched with `launch_persistent_context`.
        - If `_browser` exists, it closes the browser.
        - Sets `_browser` and `_context` to `None`.
- **`context` Property:**
    - If `_context` is not already set (i.e., not a persistent context launched directly), it creates a new context using `self.browser.new_context()`. This new context is *not* persistent by default unless `persistent_context_path` was used during the initial browser launch.

### 3. `DockerPlaywrightBrowser`

The `DockerPlaywrightBrowser` class is an abstract base class for browsers that run Playwright inside a Docker container. It handles the common Docker-related setup and teardown.

**Key Features and Management:**

- **Initialization:**
    - Takes a `docker_image` string, which is the name of the Docker image to use.
    - `port`: An optional integer for mapping a specific port from the container to the host (e.g., for VNC).
- **Lifecycle Management:**
    - `_start_browser_if_needed()`:
        - Checks if `_container` (the Docker container instance) is already running.
        - If not, it pulls the specified `docker_image` using `docker.images.pull()`.
        - It defines `container_ports` based on whether a `port` was provided during initialization. If `port` is given, it maps the container's port 5900 (typically for VNC) to the host's specified `port` and also maps port 7900 (typically for noVNC web access) to another host port.
        - It runs the Docker container using `docker.containers.run()`:
            - The image is `self.docker_image`.
            - `detach=True` runs the container in the background.
            - `auto_remove=True` ensures the container is removed when stopped.
            - `init=True` runs an init process in the container for better signal handling.
            - `shm_size="2gb"` provides a larger shared memory size, often needed by browsers.
            - `ports=container_ports` sets up the port mapping.
            - `command=["sleep", "infinity"]` keeps the container running until explicitly stopped.
        - It then waits for the container to be in a running state.
        - It determines the WebSocket endpoint for Playwright by inspecting the container's logs for a "Listening on ws://" message. This endpoint is crucial for the Playwright client to connect to the browser server running inside Docker.
        - It connects to the browser server using `playwright.chromium.connect_over_cdp(ws_endpoint)` (assuming Chromium, though this could be generalized).
        - `_browser` is set to the connected browser instance.
    - `_close_browser_if_needed()`:
        - If `_browser` exists, it closes the connection.
        - If `_container` exists, it stops and removes the container.
        - Sets `_browser`, `_container`, and `_ws_endpoint` to `None`.
- **`context` Property:**
    - If `_context` is not set, it creates a new context using `self.browser.new_context()`.

### 4. `HeadlessDockerPlaywrightBrowser`

This class runs Playwright in a standard headless Docker container.

**Key Features and Management:**

- **Inheritance:** It inherits from `DockerPlaywrightBrowser`.
- **Initialization:**
    - It calls the parent constructor with a `docker_image` that defaults to `"mcr.microsoft.com/playwright:latest"`. This is the official Playwright Docker image.
    - It does not require any special port configurations beyond what `DockerPlaywrightBrowser` might handle if a port were passed (though typically not needed for headless).
- **Functionality:** It relies entirely on the `DockerPlaywrightBrowser`'s implementation for starting the container, connecting to the browser via WebSocket, and managing the lifecycle. The browser inside the Docker container runs headlessly by default as configured in the official Playwright image.

### 5. `VncDockerPlaywrightBrowser`

This class runs Playwright in a Docker container that includes a VNC server, allowing for visual inspection and interaction with the browser.

**Key Features and Management:**

- **Inheritance:** It inherits from `DockerPlaywrightBrowser`.
- **Custom Docker Image:**
    - It uses a custom Docker image named `"douglasbastos/playwright-vnc:latest"` by default. This image is presumably built with Playwright, a VNC server (like x11vnc or TigerVNC), and a window manager.
- **Port Configuration:**
    - During initialization, it calls the parent `DockerPlaywrightBrowser` constructor, passing the `docker_image` and explicitly setting `port=vnc_port` (defaulting to 5900).
    - `DockerPlaywrightBrowser` then maps this:
        - Container port 5900 (VNC server) to `host_vnc_port` (e.g., 5900 on the host).
        - Container port 7900 (noVNC web client, if the image includes it) to `host_novnc_port` (e.g., 7900 on the host).
    - This allows users to connect a VNC client to `localhost:host_vnc_port` or access a web-based VNC client via `http://localhost:host_novnc_port`.
- **Lifecycle and Context:**
    - The lifecycle (`_start`, `_close`) and context provision (`context` property) are handled by the parent `DockerPlaywrightBrowser`. The key difference is the Docker image used and the port mapping, which enables VNC access. The browser itself, once connected via Playwright's WebSocket, operates similarly to the headless Docker version from Playwright's perspective, but its UI is rendered within the VNC session in the container.

### 6. Lifecycle Management (`_start`, `_close`, `__aenter__`, `__aexit__`)

- **`_start()` and `_close()` (Public Methods):**
    - These are the user-facing methods to manually start and stop the browser.
    - `_start()`: Calls `_start_browser_if_needed()`.
    - `_close()`: Calls `_close_browser_if_needed()`.
- **`__aenter__` and `__aexit__` (Asynchronous Context Managers):**
    - Defined in the `PlaywrightBrowser` ABC.
    - `__aenter__`: Ensures the browser is started by calling `_start_browser_if_needed()` (which is implemented by subclasses) and returns the browser instance itself (`self`).
    - `__aexit__`: Ensures the browser is cleaned up by calling `_close_browser_if_needed()` (implemented by subclasses).
- **`_start_browser_if_needed()` (Protected, Implemented by Subclasses):**
    - `LocalPlaywrightBrowser`: Launches a local browser instance (headful/headless, persistent/incognito) using Playwright's `launch` or `launch_persistent_context` methods.
    - `DockerPlaywrightBrowser` (and its children): Starts a Docker container, waits for it to be ready, finds the Playwright WebSocket endpoint from container logs, and connects to it using `connect_over_cdp`.
- **`_close_browser_if_needed()` (Protected, Implemented by Subclasses):**
    - `LocalPlaywrightBrowser`: Closes the Playwright browser instance and any persistent context it owns.
    - `DockerPlaywrightBrowser` (and its children): Closes the Playwright browser connection and stops the Docker container.

### 7. `BrowserContext` Provision

A Playwright `BrowserContext` represents an isolated "session" within a browser instance. It's where pages are created, and it can manage its own cookies, permissions, and local storage.

- **`LocalPlaywrightBrowser`:**
    - If `persistent_context_path` is provided during initialization:
        - `launch_persistent_context()` is used, which directly returns a `BrowserContext`. This context (`self._context`) is then used, and its browser (`self._browser`) is associated.
    - If `persistent_context_path` is *not* provided:
        - `launch()` is used, which returns a `Browser` instance (`self._browser`).
        - The `context` property will then create a new, non-persistent (incognito-like) context on demand using `self.browser.new_context()`. A new one is created each time `context` is accessed if `_context` is not already populated.
- **`DockerPlaywrightBrowser` (and its children `HeadlessDockerPlaywrightBrowser`, `VncDockerPlaywrightBrowser`):**
    - After connecting to the browser running inside Docker (`self._browser` is set), these classes do not inherently create a persistent context.
    - The `context` property, when accessed, will create a new, non-persistent context using `self.browser.new_context()` if `self._context` is not already set. Like the local non-persistent case, a new context would be generated on first access within a session.

In summary, the system provides a flexible way to manage Playwright browsers, whether running locally with various configurations or within Docker containers (headless or with VNC). The `PlaywrightBrowser` ABC ensures a consistent interface, while subclasses handle the specifics of each environment. Context management allows for both persistent sessions (primarily with `LocalPlaywrightBrowser`) and isolated, incognito-like sessions.
