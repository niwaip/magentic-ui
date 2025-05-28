## Invocation Flow Summary: Agent Interaction with Web Pages via Playwright Components

This document outlines the typical step-by-step sequence an agent would follow to interact with web pages using the described Playwright components. This flow is based on the functionalities detailed in `browser_management.md`, `playwright_controller_analysis.md`, `page_script_analysis.md`, and `playwright_state_analysis.md`.

**Objective**: To programmatically control a web browser, perform actions on pages, retrieve information, and optionally persist browser state.

---

**Step 1: Instantiate a `PlaywrightBrowser` Implementation**

The agent first needs to decide how the browser will be run (locally, in Docker, headless, with VNC, etc.) and instantiate the corresponding `PlaywrightBrowser` subclass.

*   **Example (`LocalPlaywrightBrowser`)**:
    ```python
    from magentic_ui.tools.playwright import LocalPlaywrightBrowser

    # For a local, headless browser session
    browser_manager = LocalPlaywrightBrowser(headless=True)

    # Or, for a local, headful browser with a persistent context
    # browser_manager = LocalPlaywrightBrowser(headless=False, persistent_context_path="/path/to/user/data")
    ```
*   **Other options**: `HeadlessDockerPlaywrightBrowser`, `VncDockerPlaywrightBrowser`.
*   **Reference**: `docs/browser_management.md`

**Step 2: Start the Browser and Obtain a `BrowserContext`**

The browser instance needs to be started to host browser contexts and pages. This is typically done using an `async with` statement for proper lifecycle management, or by manually calling `_start()` and `_close()`. The `BrowserContext` is then obtained from the browser manager.

*   **Example (using `async with`)**:
    ```python
    async with browser_manager:
        # Browser is started automatically via __aenter__
        browser_context = await browser_manager.context
        # ... proceed with steps 3 onwards ...
    # Browser is closed automatically via __aexit__
    ```
*   **Example (manual start/close)**:
    ```python
    # await browser_manager._start() # If not using async with
    # browser_context = await browser_manager.context
    # ...
    # await browser_manager._close()
    ```
*   The `browser_manager.context` property handles the creation or retrieval of a `BrowserContext`.
*   **Reference**: `docs/browser_management.md` (Sections on `PlaywrightBrowser` ABC, lifecycle management, and context provision)

**Step 3: Create a `PlaywrightController` Instance**

Once a `BrowserContext` is available, the agent instantiates `PlaywrightController`, passing the context and any desired configuration options.

*   **Example**:
    ```python
    from magentic_ui.tools.playwright import PlaywrightController

    controller = PlaywrightController(
        browser_context=browser_context,
        animate_actions=False,       # Set to True for visual debugging
        single_tab_mode=True,        # Enforce single tab operation
        default_timeout=30.0         # Default timeout for operations
    )
    ```
*   The `PlaywrightController` will initialize its first page from the context or create one if none exist. It also injects `page_script.js` into the page(s).
*   **Reference**: `docs/playwright_controller_analysis.md` (Initialization Parameters)

**Step 4: (Implicit/Internal) `PlaywrightController` Page Initialization**

When `PlaywrightController` is initialized:
1.  It obtains its primary `Page` object (`self.page`) from the `browser_context`. If no pages exist, it creates one.
2.  It calls its internal `_initialize_page(self.page)` method. This method:
    *   Sets page viewport, default timeout, and extra HTTP headers as per controller configuration.
    *   **Crucially, injects `page_script.js` using `page.add_init_script()`**. This makes functions like `WebSurfer.getInteractiveRects()` available on the client-side.
    *   Sets up event listeners for new pages/popups based on `single_tab_mode`.
    *   Initializes `AnimationUtilsPlaywright` if `animate_actions` is true.

The agent does not explicitly call an `on_new_page()` method; this initialization is part of the `PlaywrightController`'s constructor and its internal handling of new pages (popups). If the agent uses `controller.create_new_tab()`, the new page also goes through this initialization.

*   **Reference**: `docs/playwright_controller_analysis.md` (Initialization, Interaction with `page_script.js`) and `docs/page_script_analysis.md`.

**Step 5: Use `PlaywrightController` Methods for Actions and Information Retrieval**

The agent now uses the various methods of the `PlaywrightController` to interact with the web page.

*   **Navigation**:
    ```python
    await controller.visit_page("https://example.com")
    ```
*   **Understanding the Page**:
    The agent typically calls `describe_page()` to get a comprehensive view of the current page, including interactive elements (which are identified by `page_script.js` and given `__elementId`s).
    ```python
    page_description = await controller.describe_page()
    # Agent processes page_description["interactive_elements"] to find target elements
    # and their element_ids (referred to as __elementId in page_script.js output)
    ```
*   **Performing Actions** (using `element_id` obtained from `describe_page`):
    ```python
    target_element_id = "15" # Example ID from page_description
    await controller.click_id(target_element_id)
    await controller.fill_id("some_input_id", "text to fill")
    ```
    If `animate_actions` is true, these actions will be visually animated using `AnimationUtilsPlaywright`.
*   **Retrieving Text/Markdown**:
    ```python
    markdown_content = await controller.get_page_markdown()
    visible_text = await controller.get_visible_text() # Uses page_script.js
    ```
*   **Other actions**: `hover_id`, `select_option`, `keypress`, `get_screenshot`, etc.
*   **Reference**: `docs/playwright_controller_analysis.md` (Key Methods), `docs/page_script_analysis.md` (Data Extraction Capabilities for how element IDs and details are sourced).

**Step 6: (Optional) Use `save_browser_state()` and `load_browser_state()` for Persistence**

If the agent needs to persist the browser session (open tabs, cookies, local storage) to resume later:

*   **Saving State**:
    ```python
    from magentic_ui.tools.playwright.playwright_state import save_browser_state

    # controlled_page is the current self.page of the PlaywrightController
    browser_state_data = await save_browser_state(
        context=browser_context,
        controlled_page=controller.page,
        simplified=False # Set to False to save cookies/localStorage
    )
    # Agent can now store browser_state_data (e.g., serialize to JSON and save to file)
    ```
*   **Loading State** (typically at the start of a new session, after Step 2):
    ```python
    from magentic_ui.tools.playwright.playwright_state import load_browser_state

    # browser_state_data would be loaded from where it was persisted
    # await load_browser_state(
    #     context=browser_context,
    #     state=loaded_browser_state_data,
    #     load_only_active_tab=False
    # )
    # After loading state, a new PlaywrightController would be instantiated (Step 3)
    # or the existing controller's page might need to be updated if the active tab changed.
    ```
*   **Note**: As per `docs/playwright_state_analysis.md`, the `load_browser_state` function in the analyzed code does not explicitly restore cookies/localStorage from `browser_state_data.state`. This would need to be handled for full state restoration.
*   **Reference**: `docs/playwright_state_analysis.md`

**Step 7: Close the Browser**

Properly closing the browser is essential to free up resources.

*   If using an `async with` statement for the `PlaywrightBrowser` instance (as shown in Step 2), the browser is closed automatically when the block exits.
*   If managing manually:
    ```python
    await browser_manager._close()
    ```
*   This calls the appropriate `_close_browser_if_needed()` method in the specific `PlaywrightBrowser` implementation, which handles shutting down the Playwright browser instance and, for Docker implementations, stopping the container.
*   **Reference**: `docs/browser_management.md` (Lifecycle Management)

---

This flow demonstrates how the different components work in concert: `PlaywrightBrowser` manages the browser's existence, `page_script.js` provides in-page intelligence, `PlaywrightController` orchestrates actions and data flow, and `playwright_state.py` offers persistence. Utility modules like `animation_utils.py` and `webpage_text_utils.py` are used internally by `PlaywrightController` or its associated components.
