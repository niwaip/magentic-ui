## PlaywrightController Detailed Explanation

The `PlaywrightController` class serves as the primary high-level interface for interacting with web pages using Playwright. It encapsulates page objects, provides methods for navigation, element interaction, information retrieval, and tab management, abstracting away many of the lower-level Playwright API calls.

### Role as Primary Interface

The `PlaywrightController` is designed to be the main entry point for any agent or system that needs to control a web browser programmatically. It offers a simplified and more targeted set of commands compared to the raw Playwright API, focusing on common web interaction tasks. It manages the state of the current page and browser context, allowing users to perform actions without directly handling Playwright's `Page` or `BrowserContext` objects in most cases.

### Initialization Parameters

The controller is initialized with several parameters that configure its behavior:

- **`browser_context: BrowserContext`**: A Playwright `BrowserContext` object. This is a mandatory parameter and represents the browser session in which all actions will take place. The controller does not manage the lifecycle of the `BrowserContext` itself but operates within it.
- **`downloads_folder: Path | str | None = None`**: Specifies the directory where downloaded files should be saved. If `None`, a default behavior (likely Playwright's or the browser's default) will be used.
- **`animate_actions: bool = False`**: If `True`, enables visual animations for certain actions (like clicks and typing) using `AnimationUtilsPlaywright`. This is useful for debugging or when visually demonstrating the automation.
- **`default_timeout: float = 30.0`**: A default timeout in seconds for Playwright operations (e.g., waiting for selectors, navigation).
- **`single_tab_mode: bool = True`**:
    - If `True`, the controller restricts operations to a single tab. New tabs created by web pages (e.g., `target="_blank"`) will be automatically closed, and the controller will attempt to redirect the current page to the new URL if possible.
    - If `False`, multiple tabs are allowed, and the controller provides methods to manage and switch between them.
- **`validate_urls: bool = True`**: If `True`, URLs provided to methods like `visit_page` will be validated for correct format and scheme.
- **`viewport_size: dict[str, int] | None = None`**: Sets the viewport size of the page. Defaults to `{"width": 1280, "height": 720}`. This affects how web pages are rendered and can influence element visibility and layout.
- **`user_agent: str | None = None`**: Allows overriding the browser's default user agent string.
- **`device_scale_factor: float | None = None`**: Sets the device scale factor, emulating different screen densities.
- **`is_mobile: bool | None = None`**: Emulates a mobile device environment.
- **`color_scheme: Literal["light", "dark", "no-preference"] | None = None`**: Sets the preferred color scheme (light or dark mode).
- **`timezone_id: str | None = None`**: Sets the timezone for the browser context.
- **`locale: str | None = None`**: Sets the locale for the browser context.
- **`geolocation: dict[str, float] | None = None`**: Sets the geolocation (latitude, longitude, accuracy).
- **`permissions: list[str] | None = None`**: Sets default permissions for the browser context (e.g., "geolocation", "camera").
- **`extra_http_headers: dict[str, str] | None = None`**: Sets extra HTTP headers to be sent with every request.

During initialization (`__init__` and `_initialize_page`):
- It stores these parameters.
- It sets up the initial page (`self.page`) from the `browser_context`. If no pages exist, it creates one. If multiple pages exist and `single_tab_mode` is true, it uses the first page.
- It configures the page's viewport and other settings passed during initialization.
- It injects a JavaScript file (`page_script.js`) into the page, which contains helper functions for tasks like identifying interactive elements.
- If `animate_actions` is true, it initializes `AnimationUtilsPlaywright`.
- It sets up event listeners, particularly for handling new pages (popups) according to `single_tab_mode`.

### Key Methods for Page Manipulation

#### Navigation
- **`visit_page(url: str) -> str`**: Navigates the current page to the specified URL. It validates the URL if `validate_urls` is enabled. It waits for the page to load (specifically, the "domcontentloaded" event). Returns a success or error message.
- **`go_back() -> str`**: Navigates the current page back in its history.
- **`go_forward() -> str`**: Navigates the current page forward in its history.
- **`refresh_page() -> str`**: Reloads the current page.

#### Element Interaction
The controller uses a system of `element_id` (typically "id" or "data-id" attributes from `page_script.js`) to identify interactive elements.

- **`click_id(element_id: str) -> str`**: Clicks the element identified by `element_id`. Uses Playwright's `locator.click()`. If `animate_actions` is true, it also calls `self.animation_utils.animate_click()`.
- **`fill_id(element_id: str, text: str) -> str`**: Fills the input field identified by `element_id` with the given `text`. Uses `locator.fill()`. If `animate_actions` is true, it also calls `self.animation_utils.animate_type()`.
- **`hover_id(element_id: str) -> str`**: Hovers over the element identified by `element_id`. Uses `locator.hover()`.
- **`select_option(element_id: str, option_value: str) -> str`**: Selects an option by its value within a `<select>` element identified by `element_id`. Uses `locator.select_option()`.
- **`scroll_id(element_id: str, direction: Literal["up", "down", "left", "right"]) -> str`**: Scrolls the element identified by `element_id` into view. While the name suggests scrolling the element itself, it typically scrolls the page to make the element visible using `element.scroll_into_view_if_needed()`.
- **`upload_file(element_id: str, file_path: str | Path) -> str`**: Uploads a file to an input element (e.g., `<input type="file">`) identified by `element_id`. Uses `locator.set_input_files()`.

#### Coordinate-Based Actions
These methods interact with the page based on x/y coordinates, often relative to the viewport.

- **`click_coords(x: float, y: float) -> str`**: Clicks at the specified (x, y) coordinates on the page. Uses `page.mouse.click()`.
- **`type_direct(text: str) -> str`**: Types the given text as if a user were typing directly into the currently focused element (or the page body if no element is focused). Uses `page.keyboard.type()`.
- **`keypress(key_combination: str) -> str`**: Simulates pressing a key or key combination (e.g., "Enter", "Control+C"). Uses `page.keyboard.press()`.
- **`drag_coords(start_x: float, start_y: float, end_x: float, end_y: float) -> str`**: Performs a drag-and-drop operation from the start coordinates to the end coordinates. Uses `page.mouse.move()` and `page.mouse.down()/up()`.

#### Page Scrolling
- **`page_down() -> str`**: Scrolls the page down by one viewport height. Uses `page.mouse.wheel(0, viewport_height)`.
- **`page_up() -> str`**: Scrolls the page up by one viewport height. Uses `page.mouse.wheel(0, -viewport_height)`.

### Key Methods for Information Retrieval

#### Screenshots
- **`get_screenshot(output_path: Path | str | None = None, image_format: Literal["png", "jpeg"] = "png", quality: int | None = None, full_page: bool = False) -> Path | bytes`**: Captures a screenshot of the current page.
    - If `output_path` is provided, saves the screenshot to that file and returns the `Path`.
    - Otherwise, returns the screenshot as `bytes`.
    - `image_format`, `quality`, and `full_page` are options passed to Playwright's `page.screenshot()`.

#### Page Details
- **`get_current_url_title() -> tuple[str, str]`**: Returns the current page's URL and title.
- **`get_visual_viewport() -> tuple[float, float]`**: Returns the width and height of the page's visual viewport.
- **`get_focused_rect_id() -> tuple[dict[str, float] | None, str | None]`**: Returns the bounding box (`dict` with x, y, width, height) and `element_id` of the currently focused interactive element. This relies on `page_script.js`.
- **`get_page_metadata() -> dict[str, str]`**: Returns a dictionary of page metadata, including OpenGraph and Twitter card information, extracted from `<meta>` tags.

#### Text Content
- **`get_all_webpage_text() -> str`**: Returns all text content from the page's body (`page.content()`, then parsed).
- **`get_visible_text() -> str`**: Returns only the visible text content from the page. This relies on `page_script.js` to identify visible elements and extract their text.
- **`get_page_markdown(simplify: bool = True) -> str`**: Converts the page's DOM into Markdown format.
    - If `simplify` is `True`, it uses a simplified Markdown representation (likely focusing on structure and key content).
    - Otherwise, it might produce a more detailed Markdown. This uses an external library or custom logic for HTML-to-Markdown conversion.

#### Comprehensive Description
- **`describe_page(screenshot_format: Literal["png", "jpeg"] | None = "png") -> dict[str, Any]`**: Provides a comprehensive description of the current page state. This is a key method for agents needing to "understand" the page. It includes:
    - Current URL and title.
    - Interactive elements (buttons, inputs, links, etc.) with their IDs, types, bounding boxes, and ARIA labels/text content. This information is retrieved by executing functions from the injected `page_script.js`.
    - The focused element's ID and bounding box.
    - A screenshot of the page (as bytes or a path, depending on how it's called internally, though the method signature implies bytes if no path is given to an underlying screenshot function).
    - Markdown representation of the page content.

### Tab Management Methods

- **`get_tabs_information() -> list[dict[str, Any]]`**: Returns a list of dictionaries, each containing information about an open tab (URL, title, and an `internal_tab_id` which is the Playwright page's internal ID).
- **`switch_tab(internal_tab_id: str) -> str`**: Switches the controller's active page to the tab identified by `internal_tab_id`.
- **`close_tab(internal_tab_id: str) -> str`**: Closes the tab identified by `internal_tab_id`. It handles switching to another tab if the closed tab was active.
- **`create_new_tab(url: str | None = None) -> str`**: Creates a new tab. If `url` is provided, it navigates the new tab to that URL. The new tab becomes the active page.

### Handling `single_tab_mode`

- If `single_tab_mode` is `True` (the default):
    - The controller listens for 'popup' events (new tabs/windows opened by the page).
    - When a popup occurs, the `_handle_new_page_single_tab` method is triggered.
    - This method attempts to get the URL the popup was trying to load.
    - It then closes the new popup page immediately.
    - Finally, it navigates the *original* page to the popup's URL. This effectively prevents new tabs from opening and keeps the interaction within the single active tab.
    - The `create_new_tab` method will likely raise an error or be a no-op if called in `single_tab_mode`.
- If `single_tab_mode` is `False`:
    - New tabs opened by web pages are allowed to exist.
    - The `_handle_new_page_multi_tab` method is triggered on 'popup' events, which simply adds the new page to the `browser_context.pages` list and switches to it, making it the active page.
    - Tab management methods (`get_tabs_information`, `switch_tab`, `close_tab`, `create_new_tab`) can be used effectively.

### Interaction with `AnimationUtilsPlaywright`

- If `animate_actions` is `True` during initialization, an instance of `AnimationUtilsPlaywright` is created, associated with the current `self.page`.
- Methods like `click_id` and `fill_id` will then call corresponding animation methods from `self.animation_utils`:
    - `self.animation_utils.animate_click(selector)` before performing the actual click.
    - `self.animation_utils.animate_type(selector, text_to_type)` before filling an input.
- These animation utilities typically inject visual cues (like highlighting or simulated cursor movement) into the page to make the automation visually traceable.

### Interaction with `page_script.js`

- During initialization (`_initialize_page`), the content of `page_script.js` (located in `src/magentic_ui/tools/playwright/js/page_script.js`) is read and injected into every new page created or controlled using `page.add_init_script()`.
- This script runs in the browser's context and provides several helper functions that the `PlaywrightController` calls using `page.evaluate()` or `locator.evaluate_all()`:
    - **Identifying Interactive Elements**: The script contains logic to find elements deemed interactive (buttons, links, inputs, textareas, selects, elements with `role` attributes like button/link, etc.). It assigns a unique `data-id` to these elements if they don't have a suitable existing ID. The `describe_page` method heavily relies on this to list interactive elements.
    - **Getting Element Properties**: Functions in `page_script.js` are used to extract properties like ARIA labels, text content, visibility status, and bounding boxes of these interactive elements.
    - **Determining Focused Element**: `get_focused_rect_id` uses a function from this script to find which interactive element currently has focus and retrieve its `data-id` and bounding box.
    - **Extracting Visible Text**: `get_visible_text` likely uses functions from this script to determine which text nodes or elements are currently visible in the viewport.
    - **Scrolling**: While Playwright has built-in scroll functions, `page_script.js` might contain helpers for more complex scroll behaviors or for ensuring elements are scrolled into view in a specific way, though `scroll_id` primarily uses `scroll_into_view_if_needed`.

By injecting `page_script.js`, the `PlaywrightController` can perform complex queries and manipulations of the page content that might be cumbersome or less efficient using only the standard Playwright API. It allows for a more semantic understanding of page elements, especially regarding their interactivity.
