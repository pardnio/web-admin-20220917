# web-admin-20220917 - Architecture

Last updated: 2026-10-25

> Back to [README](../README.md)

## Overview

```mermaid
graph TB
    Browser["Browser"] --> Page["Page HTML (*.html)"]
    Page --> RK["PDRenderKit (CDN)"]
    Page --> FA["Font Awesome (CDN)"]
    Page --> CSS["Styles css/index.css"]
    Page --> Shared["Shared events js/index.js"]
    Page --> Script["Page script js/*.js"]
    Shared --> PD["PD instance (global page)"]
    Script --> PD
    RK --> PD
    PD --> Top["view/body-top.html"]
    PD --> Left["view/body-left.html"]
    PD --> Right["view/body-right/*.html"]
    Script --> MK["PDMarkdownKit (CDN)"]
    PD --> Cookie[("Cookie UI state")]
```

## Module: Page Shell

Every `*.html` loads the same CDN assets plus `js/index.js`, switches between the login block and the main layout through `is_guest`, and assembles the main layout from three view partials.

```mermaid
graph TB
    subgraph Shell["Page shell (*.html)"]
        Head["head: css/index.css, Font Awesome, PDRenderKit, js/index.js, page script"]
        Body["body#body"]
        Login["Login block (:if is_guest)"]
        Main["Main layout (:else)"]
        Body --> Login
        Body --> Main
        Main --> Top[":path view/body-top.html"]
        Main --> Left[":path view/body-left.html"]
        Main --> Right["Content (:path view/body-right/*.html or inline)"]
    end
    Script["Page script"] -->|"new PD({ id: 'body' })"| Body
```

| Page | Content source |
|------|----------------|
| `database-list`, `database-add`, `article-add`, `image-folder`, `file-edit` | `view/body-right/<page>.html` |
| `index`, `json-edit`, `code-edit` | Inline in the page HTML |

## Module: Shared Events and Navigation

`js/index.js` declares the global `page` and `default_events`; each page spreads them into its own `event`. The navigation marks the current page from the `left.*` flags.

```mermaid
graph LR
    subgraph IndexJS["js/index.js"]
        PageVar["let page"]
        DE["default_events"]
        DE --> Show["show"]
        DE --> LoginE["login / logout"]
        DE --> BLS["body_left_show"]
        DE --> BLT["body_left_type"]
        DE --> Tab["tab_show"]
        DE --> AR["auto_resize"]
    end
    subgraph Nav["view/body-left.html"]
        G1["Database: list, add, article"]
        G2["Files: folder, text, JSON"]
        G3["Version: Readme, License"]
    end
    BLT -->|"_cookie"| CookieMin[("is_body_left_min")]
    CookieMin -->|"$cookie"| Left["left.is_body_left_min"]
    Left --> Nav
    Flags["left.is_* flags"] -->|":data-selected"| Nav
    Tab --> Nav
    BLS --> Nav
```

## Module: Database Pages

`database-list` and `database-add` never touch the DOM directly; PD data renders every form field and table cell.

```mermaid
graph TB
    subgraph List["database-list"]
        LForm["form.data"] -->|":for (key, val)"| LInputs["Filter fields (rendered when is_select == false)"]
        Head["database.head"] -->|":for"| TH["Table header"]
        Rows["database.list"] -->|":for"| TR["Rows (:class)"]
        TR -->|":html"| TD["Cells"]
        TR -->|"is_deleted == false"| Del["Delete button"]
    end
    subgraph Add["database-add"]
        AForm["form.data"] -->|":for (key, val)"| Field["Field"]
        Field -->|"is_input"| Input["input"]
        Field -->|"is_textarea"| Textarea["textarea"]
    end
```

## Module: Editors

The article, text, and JSON pages import PDMarkdownKit as an ES module and mount the editor into `section.markdown` in the PD `next` callback.

```mermaid
graph TB
    MK["PDMarkdownKit@1.0.0"] --> Editor["editor (MDEditor)"]
    MK --> Viewer["viewer (MDViewer)"]
    subgraph Article["article-add"]
        Toolbar["Toolbar events add_*"] --> Editor
        Photo["show_photo_library / add_photo"] -->|"addImage"| Editor
        Editor <-->|"editor.viewer / viewer.editor"| Viewer
        Editor -->|"onwheel scroll sync"| Viewer
    end
    subgraph File["file-edit"]
        Editor2["editor.init(robots.txt sample)"]
    end
    subgraph Json["json-edit / code-edit"]
        Editor3["editor.init(JSON.stringify(...))"]
        Save["save: getMdTxt, normalize whitespace, JSON.parse"]
        Mode["change_json_mode (json-edit only)"]
        Tree["section.json field tree"]
        Code["section.markdown raw code"]
        Mode --> Tree
        Mode --> Code
        Editor3 --> Code
        Code --> Save
    end
    Editor --> Editor2
    Editor --> Editor3
```

## Module: File Folder

`image-folder` renders folders as `<details>` elements; events resolve files through `data-src` and drive the preview and move dialogs.

```mermaid
graph TB
    subgraph Folder["image-folder"]
        Toggle["change_to_list / change_to_grid"] -->|"data-grid"| ListSec["section.list"]
        Toggle -->|"_cookie"| CG[("is_folder_grid")]
        CG -->|"$cookie"| Grid["is_folder_grid"] --> ListSec
        Item["File button (data-src)"] -->|"get_image_viewer"| Viewer["section.folder-viewer"]
        Viewer -->|"jpg/jpeg/png/svg/webp"| Img["img preview"]
        Rename["set_file_name"] --> Prompt["prompt"]
        Move["set_file_folder"] --> PathDlg["section.folder-path"]
        PathDlg -->|"set_file_folder_selected"| Layer["data-selected layer"]
        DelF["delete_file"] -->|"confirm"| RemoveF["Remove file node"]
        DelD["delete_folder"] -->|"confirm"| RemoveD["Remove folder node"]
    end
```

## Data Flow

```mermaid
sequenceDiagram
    participant B as Browser
    participant H as Page HTML
    participant RK as PDRenderKit
    participant I as js/index.js
    participant S as Page script
    participant PD as PD instance
    participant V as view/ partials
    participant MK as PDMarkdownKit
    B->>H: Request *.html
    H->>RK: Load PDRenderKit.js
    H->>I: Load default_events
    H->>S: Load page script (some as module)
    S->>MK: import editor / viewer (editor pages)
    B->>S: DOMContentLoaded
    S->>PD: new PD({ id, data, event, next })
    PD->>V: Load partials via :path
    PD->>B: Render :if / :for / {{ }} and bind @click
    PD->>S: Call next()
    S->>MK: Mount editor.body and init()
```

## State Machine

### Login

```mermaid
stateDiagram-v2
    [*] --> Main: is_guest = false (default)
    Main --> Login: logout (is_guest = true)
    Login --> Transition: login (add show class)
    Transition --> Main: is_guest = false after 1 second
```

### JSON Edit Mode (json-edit)

```mermaid
stateDiagram-v2
    [*] --> Tree: section.json visible
    Tree --> Code: change_json_mode
    Code --> Tree: change_json_mode
```

### Image Preview (image-folder)

```mermaid
stateDiagram-v2
    [*] --> Hidden
    Hidden --> Shown: get_image_viewer (supported image extension, data-show = 1)
    Shown --> Hidden: get_image_viewer (data-show = 0)
```

***

©️ 2022 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
