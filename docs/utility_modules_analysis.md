## Analysis of Utility Modules: `animation_utils.py` and `webpage_text_utils.py`

This document provides a brief explanation of the purpose of two utility modules used within the Playwright tooling: `animation_utils.py` and `webpage_text_utils.py`.

### `animation_utils.py`

**Purpose**:
The `animation_utils.py` module is responsible for providing visual feedback during automated browser interactions controlled by Playwright. This is primarily useful for debugging, demonstrations, or when a user wants to visually follow the automation's actions on a web page. It enhances the user experience by simulating a more human-like interaction through visual cues.

**Key Class**: `AnimationUtilsPlaywright`

**Functionality**:
-   **Cursor Simulation**: The class can create and display a custom visual cursor (a red circle) on the page.
-   **Element Highlighting**: It can highlight specific HTML elements (identified by an `__elementId`, typically assigned by `page_script.js`) by drawing a border around them.
-   **Animated Cursor Movement**: The `gradual_cursor_animation` method animates the custom cursor's movement from a starting coordinate to an ending coordinate in a series of small steps, simulating smooth mouse movement.
-   **Action Association**: These visual effects are intended to be tied to browser actions. For instance, before a click action, an element might be highlighted and the cursor animated towards it.
-   **Cleanup**: Provides methods to remove the cursor and any highlighting from the page, ensuring a clean state after animations are complete or no longer needed (`remove_cursor_box`, `cleanup_animations`).

The methods in this class (e.g., `add_cursor_box`, `gradual_cursor_animation`, `remove_cursor_box`) directly manipulate the DOM by injecting styles and elements via JavaScript evaluations executed by Playwright's `page.evaluate()`.

### `webpage_text_utils.py`

**Purpose**:
The `webpage_text_utils.py` module focuses on extracting textual content from web pages in various formats. This is essential for agents or systems that need to understand or process the information present on a page, whether it's rendered HTML, a PDF document, or requires conversion to Markdown.

**Key Class**: `WebpageTextUtilsPlaywright`

**Functionality**:
-   **Plain Text Extraction**:
    -   `get_all_webpage_text()`: Retrieves the `innerText` of the entire `document.body`, effectively getting all rendered text. It can limit the output to a specified number of lines.
    -   `get_visible_text()`: Leverages the injected `page_script.js` (specifically `WebSurfer.getVisibleText()`) to extract only the text content currently visible within the browser's viewport. This is more aligned with what a human user would see.
-   **Markdown Conversion**:
    -   `get_page_markdown()`: Converts the HTML content of the current page (`document.documentElement.outerHTML`) into Markdown format.
    -   It uses the `markitdown` library for this conversion.
    -   It can limit the output to a maximum number of tokens (using `tiktoken` for tokenization, specifically with "gpt-4o" model settings).
-   **PDF Content Handling**:
    -   The `get_page_markdown()` method automatically detects if the current page is a PDF (by checking URL, content type, or specific PDF viewer elements in the DOM via `_is_pdf_page()`).
    -   If a PDF is detected, `_extract_pdf_content()` is called. This method first attempts to extract text using browser-based techniques (evaluating JavaScript that interacts with PDF.js viewers or common PDF DOM structures via `_extract_pdf_browser()`).
    -   If browser-based extraction is insufficient, it downloads the PDF content, saves it to a temporary file, and then uses `markitdown` to extract text content from the PDF file.
    -   PDF content can also be token-limited.
-   **Initialization**: The class initializes an instance of `MarkItDown` for conversions and reads the `page_script.js` content for use in `get_visible_text()`.

This module provides a comprehensive suite of tools for obtaining textual data from web pages, adapting its strategy based on the content type (HTML vs. PDF) and offering options for simplification (visible text only) or structural representation (Markdown).
