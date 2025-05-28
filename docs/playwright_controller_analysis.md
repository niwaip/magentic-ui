## PlaywrightController: Detailed Analysis

The `PlaywrightController` class, found in `src/magentic_ui/tools/playwright/playwright_controller.py`, serves as the main high-level interface for programmatic interaction with web pages using the Playwright library. It abstracts many of Playwright's complexities, offering a more user-friendly API for common browsing tasks, element interactions, and data extraction.

### Role as Primary Interface

The `PlaywrightController` is designed to be the central point of control for web automation tasks. Instead of directly manipulating Playwright's `Page`, `BrowserContext`, or `ElementHandle` objects, users interact with the methods provided by this controller. This simplifies the process of writing automation scripts by providing a curated set of functionalities tailored for typical web interaction scenarios. It manages the current page, handles new pages according to its configuration, and integrates helper scripts for enhanced element detection.

### Initialization Parameters

The controller is instantiated with several parameters that dictate its behavior and the configuration of the Playwright page it manages:

- **`browser_context: BrowserContext`**: This is a mandatory Playwright `BrowserContext`. The controller operates within this context, meaning all pages and actions are scoped to this browser session. The controller does not manage the lifecycle (creation/destruction) of the `BrowserContext` itself.
- **`downloads_folder: Path | str | None = None`**: Specifies a directory where files downloaded during the session should be saved. If not provided, Playwright's default download behavior is used.
- **`animate_actions: bool = False`**: If set to `True`, the controller will use the `AnimationUtilsPlaywright` class to provide visual feedback for actions like clicking and typing. This is primarily for debugging or demonstration purposes.
- **`default_timeout: float = 30.0`**: Sets a default timeout (in seconds) for various Playwright operations, such as waiting for selectors to appear or for navigation events to complete. This is used to set `self.page.set_default_timeout(default_timeout * 1000)` and `self.browser_context.set_default_timeout(default_timeout * 1000)`.
- **`single_tab_mode: bool = True`**:
    - If `True` (default), the controller enforces a single-tab browsing experience. Any new tabs or windows (popups) opened by web pages are automatically closed, and the controller attempts to navigate the current tab to the URL of the intended popup.
    - If `False`, multiple tabs are permitted, and the controller provides methods to manage and switch between them.
- **`validate_urls: bool = True`**: When `True`, URLs passed to methods like `visit_page` are validated for correct formatting (e.g., presence of a scheme like `http` or `https`).
- **`viewport_size: dict[str, int] | None = None`**: Allows setting a custom viewport size for the page (e.g., `{"width": 1920, "height": 1080}`). If not specified, it defaults to `{"width": 1280, "height": 720}`. This impacts how web pages are rendered. This is applied using `self.page.set_viewport_size(self.viewport_size)`.
- **`user_agent: str | None = None`**: Allows overriding the default user agent string of the browser. This is passed to `self.browser_context.new_page()` if a new page has to be created, or set via `self.page.context.new_page(user_agent=self.user_agent)` if a page already exists but needs user_agent override.
- **`device_scale_factor: float | None = None`**: Emulates different screen pixel densities. This is part of the arguments to `self.browser_context.new_page()`.
- **`is_mobile: bool | None = None`**: Configures the page to emulate a mobile device environment. This is part of the arguments to `self.browser_context.new_page()`.
- **`color_scheme: Literal["light", "dark", "no-preference"] | None = None`**: Sets the preferred color scheme (e.g., light or dark mode) for the page. This is part of the arguments to `self.browser_context.new_page()`.
- **`timezone_id: str | None = None`**: Sets a specific timezone for the browser context. This is part of the arguments to `self.browser_context.new_page()`.
- **`locale: str | None = None`**: Sets a specific locale for the browser context. This is part of the arguments to `self.browser_context.new_page()`.
- **`geolocation: dict[str, float] | None = None`**: Allows setting a custom geolocation (latitude, longitude, accuracy). This is part of the arguments to `self.browser_context.new_page()`.
- **`permissions: list[str] | None = None`**: Allows granting specific browser permissions (e.g., "geolocation", "camera") to the context. This is part of the arguments to `self.browser_context.new_page()`.
- **`extra_http_headers: dict[str, str] | None = None`**: Defines extra HTTP headers that will be sent with every request made by the page. This is set using `self.page.set_extra_http_headers(self.extra_http_headers)`.

Upon initialization, the controller sets up its initial page (`self.page`) from the `browser_context`. If no pages exist in the context, it creates one using `self.browser_context.new_page()` with relevant parameters (`user_agent`, `device_scale_factor`, `is_mobile`, `color_scheme`, `timezone_id`, `locale`, `geolocation`, `permissions`). If pages do exist, it typically uses the first one.
It then configures the page's viewport, default timeout, and extra HTTP headers.
Critically, it injects a JavaScript file (`page_script.js`) into the page using `self.page.add_init_script(path=JS_SCRIPT_PATH)`. This script contains helper functions for tasks like identifying interactive elements.
If `animate_actions` is true, it initializes `AnimationUtilsPlaywright(self.page)`.
It sets up event listeners for page popups (`self.page.on("popup", ...)` or `self.browser_context.on("page", ...)` depending on whether an initial page existed) to handle them according to `single_tab_mode`.

### Key Methods for Page Manipulation

#### Navigation
- **`visit_page(url: str) -> str`**: Navigates the current page (`self.page`) to the specified URL. It validates the URL format if `self.validate_urls` is enabled. It waits for the page to load using `self.page.goto(url, wait_until="domcontentloaded")`. Returns a success or error message.
- **`go_back() -> str`**: Navigates the current page back in its history using `self.page.go_back()`.
- **`go_forward() -> str`**: Navigates the current page forward in its history using `self.page.go_forward()`.
- **`refresh_page() -> str`**: Reloads the current page using `self.page.reload()`.

#### Element Interaction
The controller uses an `element_id` (typically a "data-id" attribute assigned by `page_script.js` or an existing HTML "id") to identify interactive elements. It forms a Playwright locator using `self.page.locator(f"#{element_id}, [data-id='{element_id}']")`.

- **`click_id(element_id: str) -> str`**: Clicks the element identified by `element_id`. Uses the locator's `click()` method. If `animate_actions` is true, it also calls `self.animation_utils.animate_click()` with the selector.
- **`fill_id(element_id: str, text: str) -> str`**: Fills the input field identified by `element_id` with the given `text`. Uses the locator's `fill(text)` method. If `animate_actions` is true, it also calls `self.animation_utils.animate_type()` with the selector and text.
- **`hover_id(element_id: str) -> str`**: Hovers over the element identified by `element_id`. Uses the locator's `hover()` method.
- **`select_option(element_id: str, option_value: str) -> str`**: Selects an option by its value within a `<select>` element identified by `element_id`. Uses the locator's `select_option(value=option_value)` method.
- **`scroll_id(element_id: str, direction: Literal["up", "down", "left", "right"]) -> str`**: Scrolls the element identified by `element_id` into view using `element.scroll_into_view_if_needed()`. The `direction` parameter is not directly used by the Playwright call but might be for logging or future use.
- **`upload_file(element_id: str, file_path: str | Path) -> str`**: Uploads a file to an input element (e.g., `<input type="file">`) identified by `element_id`. Uses the locator's `set_input_files(file_path)` method.

#### Coordinate-Based Actions
- **`click_coords(x: float, y: float) -> str`**: Clicks at the specified (x, y) coordinates on the page using `self.page.mouse.click(x, y)`.
- **`type_direct(text: str) -> str`**: Types the given text as if a user were typing directly into the currently focused element (or the page body if no element is focused). Uses `self.page.keyboard.type(text)`.
- **`keypress(key_combination: str) -> str`**: Simulates pressing a key or key combination (e.g., "Enter", "Control+C"). Uses `self.page.keyboard.press(key_combination)`.
- **`drag_coords(start_x: float, start_y: float, end_x: float, end_y: float) -> str`**: Performs a drag-and-drop operation from the start coordinates to the end coordinates. Uses `self.page.mouse.move()`, `self.page.mouse.down()`, and `self.page.mouse.up()`.

#### Page Scrolling
- **`page_down() -> str`**: Scrolls the page down by one viewport height. It calculates viewport height from `self.page.viewport_size` and uses `self.page.mouse.wheel(0, viewport_height)`.
- **`page_up() -> str`**: Scrolls the page up by one viewport height. Uses `self.page.mouse.wheel(0, -viewport_height)`.

### Key Methods for Information Retrieval

#### Screenshots
- **`get_screenshot(output_path: Path | str | None = None, image_format: Literal["png", "jpeg"] = "png", quality: int | None = None, full_page: bool = False) -> Path | bytes`**: Captures a screenshot of the current page.
    - If `output_path` is provided, saves the screenshot to that file using `self.page.screenshot(path=output_path, ...)`.
    - Otherwise, returns the screenshot as `bytes` using `self.page.screenshot(path=None, ...)`.
    - `image_format`, `quality`, and `full_page` are options passed to Playwright's `page.screenshot()`.

#### Page Details
- **`get_current_url_title() -> tuple[str, str]`**: Returns the current page's URL (`self.page.url`) and title (`self.page.title()`).
- **`get_visual_viewport() -> tuple[float, float]`**: Returns the width and height of the page's visual viewport by evaluating JavaScript: `self.page.evaluate("[window.visualViewport.width, window.visualViewport.height]")`.
- **`get_focused_rect_id() -> tuple[dict[str, float] | None, str | None]`**: Returns the bounding box (`dict` with x, y, width, height) and `element_id` of the currently focused interactive element. This relies on the injected `page_script.js` function `window.getFocusedRectAndDataId()`.
- **`get_page_metadata() -> dict[str, str]`**: Returns a dictionary of page metadata, including OpenGraph and Twitter card information, extracted from `<meta>` tags by evaluating `window.getPageMetadata()` from `page_script.js`.

#### Text Content
- **`get_all_webpage_text() -> str`**: Returns all text content from the page's body by getting `self.page.content()` and then parsing it with `BeautifulSoup(html_content, "html.parser").get_text()`.
- **`get_visible_text() -> str`**: Returns only the visible text content from the page. This relies on `window.getVisibleText()` from `page_script.js`.
- **`get_page_markdown(simplify: bool = True) -> str`**: Converts the page's DOM into Markdown format using the `Html2Markdown` utility class. If `simplify` is true, a simplified version is attempted.

#### Comprehensive Description
- **`describe_page(screenshot_format: Literal["png", "jpeg"] | None = "png") -> dict[str, Any]`**: Provides a comprehensive description of the current page state. It includes:
    - Current URL and title.
    - Interactive elements (buttons, inputs, links, etc.) with their IDs, types, bounding boxes, and ARIA labels/text content. This information is retrieved by executing `window.getInteractiveElements()` from `page_script.js`.
    - The focused element's ID and bounding box from `get_focused_rect_id()`.
    - A screenshot of the page (as bytes, obtained via `get_screenshot(image_format=screenshot_format)`).
    - Markdown representation of the page content from `get_page_markdown()`.

### Tab Management Methods

- **`get_tabs_information() -> list[dict[str, Any]]`**: Returns a list of dictionaries, each containing information about an open tab (URL, title, and an `internal_tab_id` which is the Playwright page's internal `guid`). It iterates through `self.browser_context.pages`.
- **`switch_tab(internal_tab_id: str) -> str`**: Switches the controller's active page (`self.page`) to the tab identified by `internal_tab_id`. It finds the page in `self.browser_context.pages` matching the ID.
- **`close_tab(internal_tab_id: str) -> str`**: Closes the tab identified by `internal_tab_id`. It handles switching to another tab (usually the first available) if the closed tab was active.
- **`create_new_tab(url: str | None = None) -> str`**: Creates a new tab using `self.browser_context.new_page()`. If `url` is provided, it navigates the new tab to that URL. The new tab becomes the active page (`self.page`). This method raises an error if used in `single_tab_mode`.

### Handling `single_tab_mode`

- If `single_tab_mode` is `True` (the default):
    - The controller sets up a handler (`_handle_new_page_single_tab`) for new pages. This handler is attached either to `self.page.on("popup", ...)` if an initial page exists, or `self.browser_context.on("page", ...)` if the controller creates the first page.
    - `_handle_new_page_single_tab(new_page)`:
        - Gets the URL the `new_page` was trying to load (by waiting for its load event or using its main frame URL).
        - Closes the `new_page` immediately.
        - Navigates the original `self.page` to the captured URL.
    - This ensures all navigations effectively occur within the single main tab.
- If `single_tab_mode` is `False`:
    - The `_handle_new_page_multi_tab(new_page)` handler is used.
    - This handler simply sets `self.page = new_page`, making the newly opened tab the active one. It also ensures the new page has the init script and viewport settings applied.

### How it uses `AnimationUtilsPlaywright`

- If `animate_actions` is `True` during initialization, an instance of `AnimationUtilsPlaywright` is created: `self.animation_utils = AnimationUtilsPlaywright(self.page)`.
- Before performing actions like `click_id` and `fill_id`, the controller calls the corresponding animation methods from `self.animation_utils`:
    - `self.animation_utils.animate_click(selector)`: Visually highlights the element or shows a cursor animation before the click.
    - `self.animation_utils.animate_type(selector, text_to_type)`: Visually indicates the input field and simulates typing.
- These animations are for visual feedback and do not affect the actual Playwright action's outcome.

### Interaction with `page_script.js`

- The JavaScript file (`JS_SCRIPT_PATH`, which points to `js/page_script.js`) is injected into every page managed by the controller via `page.add_init_script()`. This makes its functions available globally (on the `window` object) in the page's context.
- The `PlaywrightController` then calls these JavaScript functions using `page.evaluate("window.functionName()")`:
    - **`window.getInteractiveElements()`**: Called by `describe_page` to find all elements deemed interactive (buttons, links, inputs, etc.). The script assigns them `data-id` attributes if they lack an HTML `id`, and collects details like type, bounding box, ARIA label, and text.
    - **`window.getFocusedRectAndDataId()`**: Called by `get_focused_rect_id` to get the bounding box and `data-id` of the currently focused element.
    - **`window.getPageMetadata()`**: Called by `get_page_metadata` to extract metadata like OpenGraph and Twitter tags.
    - **`window.getVisibleText()`**: Called by `get_visible_text` to get only the text content of currently visible elements.
- This interaction allows for complex DOM querying and information extraction logic to be executed efficiently in the browser's context, providing data that is then used by the controller's Python methods. The `data-id` attribute serves as a stable identifier for elements between Python and JavaScript.
