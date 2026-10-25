# web-admin-20220917 - Documentation

Last updated: 2026-10-25

> Back to [README](../README.md)

## Prerequisites

- A modern browser with ES module support (`article-add`, `file-edit`, `json-edit`, and `code-edit` load through `<script type="module">`)
- A static HTTP server: pages load `view/` partials through `:path` and import packages as ES modules, so opening them via `file://` does not work
- Access to `cdn.jsdelivr.net`: PDRenderKit, PDMarkdownKit, and Font Awesome all load from the CDN
- (Optional) Node.js and Sass, only for recompiling styles

## Installation

### From Source

```bash
git clone https://github.com/pardnchiu/web-admin-20220917.git
cd web-admin-20220917
```

### Local Preview

Start either static server, then open `http://localhost:8080/index.html`:

```bash
python3 -m http.server 8080
```

```bash
npx serve -l 8080 .
```

### Deployment

The project has no build step. Upload the whole directory to any static host, such as GitHub Pages.

### Recompile Styles (Optional)

The `sass` script in `package.json` watches `sass/` and writes compressed output to `css/`:

```bash
npm install --save-dev sass
npm run sass
```

The `sass/` directory is listed in `.gitignore` and is not part of the repository. `css/index.css` is the compiled output; edit it directly when no Sass sources are available.

## Usage

### Basic

Start the server and switch pages from the side navigation:

| Group | Item | Page |
|-------|------|------|
| 資料庫管理 (Database) | 資料列表 (List) | `database-list.html` |
| 資料庫管理 (Database) | 資料新增 (Add) | `database-add.html` |
| 資料庫管理 (Database) | 文章撰寫 (Article) | `article-add.html` |
| 檔案管理 (Files) | 檔案夾 (Folder) | `image-folder.html` |
| 檔案管理 (Files) | 一般文檔 (Text) | `file-edit.html` |
| 檔案管理 (Files) | JSON | `json-edit.html` |

`code-edit.html` is not linked from the navigation; open it by URL.

### Add a Form Field

`database-add` renders its form from `form.data`. Each new key adds one field:

```javascript
form: {
    title: "新增資料範例",
    data: {
        author: {
            is_input: true,
            is_textarea: false,
            is_select: false,
            type: "text",
            size: "_33",
            placeholder: "作者",
            value: ""
        }
    }
}
```

### Add a Page

1. Create `report.html`, reuse the login block and main layout of an existing page, and load the shared and page scripts:

```html
<!DOCTYPE html>
<html lang="zh-hant">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>報表</title>
        <link rel="stylesheet" href="./css/index.css">
        <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@fortawesome/fontawesome-free@6.5.2/css/all.min.css">
        <script src="https://cdn.jsdelivr.net/gh/pardnchiu/PDRenderKit@1.0.0/dist/PDRenderKit.js"></script>
        <script src="./js/index.js"></script>
        <script src="./js/report.js"></script>
        <style>
            .dom-temp {
                display: none !important;
            }
        </style>
    </head>
    <body id="body">
        <section class="login dom-temp" :if="is_guest">
            <h1>{{ login.title }}</h1>
            <section>
                <label>
                    <input type="password" placeholder="密碼" :model="login.password">
                </label>
                <i class="fa-solid fa-eye-slash" @click="show"></i>
            </section>
            <button @click="login">
                前往
                <i class="fa-solid fa-arrow-right"></i>
            </button>
        </section>
        <section class="dom-temp" :else>
            <section :path="./view/body-top.html"></section>
            <section :path="./view/body-left.html"></section>
            <section class="body-right">
                <nav>
                    <button @click="body_left_show">
                        <i class="fa-solid fa-bars"></i>
                    </button>
                    <a :href="e.href" :for="e in top_tab">{{ e.title }}</a>
                </nav>
                <p class="hint">{{ title }}</p>
            </section>
        </section>
    </body>
</html>
```

2. Create `js/report.js`, spread `default_events` into `event`, and provide the `left` and `top_tab` data the navigation needs:

```javascript
document.addEventListener("DOMContentLoaded", function () {
    page = new PD({
        id: "body",
        data: {
            is_guest: false,
            login: {
                title: "管理後台",
                password: "",
            },
            title: "報表",
            left: {
                is_body_left_min: $cookie("is_body_left_min"),
                is_database_list: 0,
                is_database_add: 0,
                is_article_add: 0,
                is_folder_image: 0,
                is_file_edit: 0,
                is_json_edit_input: 0
            },
            top_tab: [
                { title: "首頁", href: "./index.html" },
                { title: "報表", href: "" }
            ],
        },
        event: {
            ...default_events,
        },
        next: () => {
        }
    });
});
```

3. Add `<a href="./report.html">` inside the matching group's `<section data-row="N">` in `view/body-left.html`, and increase `data-row` by one.

### Save JSON Content

`save` in `json-edit` and `code-edit` normalizes whitespace, parses the text, and alerts on malformed JSON. To connect a backend, replace the `console.log` with a real request:

```javascript
save: function () {
    let txt = editor.getMdTxt();
    txt = txt.replace(/[\u0009\u000A\u000B\u000C\u000D  ​-‍⁠ 　]/g, " ");

    let json;
    try {
        json = JSON.parse(txt);
    }
    catch (err) {
        alert("錯誤: JSON解析失敗, 請檢查格式.");
        return;
    }

    fetch("/api/content", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(json)
    })
        .then(res => {
            if (!res.ok) {
                throw new Error(`HTTP ${res.status}`);
            }
        })
        .catch(err => alert(`儲存失敗: ${err.message}`));
}
```

`/api/content` is a placeholder endpoint; the project ships no backend.

## Configuration Reference

### Pages

| Page | Script | Navigation flag | Content |
|------|--------|-----------------|---------|
| `index.html` | Inline `<script>` | - | Home placeholder |
| `database-list.html` | `js/database-list.js` | `left.is_database_list` | Filter form, 10-row sample table, pagination block |
| `database-add.html` | `js/database-add.js` | `left.is_database_add` | Create form rendered from `form.data` |
| `article-add.html` | `js/article-add.js` (module) | `left.is_article_add` | Markdown editor, live preview, photo library |
| `image-folder.html` | `js/image-folder.js` | `left.is_folder_image` | Folder list, image preview, move dialog |
| `file-edit.html` | `js/file-edit.js` (module) | `left.is_file_edit` | Plain text editor seeded with a `robots.txt` sample |
| `json-edit.html` | `js/json-edit.js` (module) | `left.is_json_edit_input` | Dual-mode JSON editing: field tree and raw code |
| `code-edit.html` | `js/code-edit.js` (module) | Sets `left.is_json_edit`, which the navigation does not read | Raw-code-only JSON editing |

### PD Data Fields

| Field | Type | Purpose |
|-------|------|---------|
| `is_guest` | boolean | `true` shows the login block (`:if`), `false` shows the main layout (`:else`); every page defaults to `false` |
| `login.title` | string | Login block heading |
| `login.password` | string | Bound to the password input through `:model` |
| `title` | string | System name in the top bar |
| `left.is_body_left_min` | cookie value | Whether the side navigation is collapsed, read from `$cookie("is_body_left_min")` |
| `left.is_*` | `0` / `1` | The currently selected navigation item |
| `top_tab` | `{ title, href }[]` | Breadcrumb links at the top of the content area |
| `form.title` | string | Form heading |
| `form.data.<key>` | object | Each key renders one field; the key is the field label |
| `database.head` | string[] | Table column headings |
| `database.list` | `{ class, is_deleted, body }[]` | Table rows; `body` renders through `:html` and may contain HTML; `is_deleted: true` hides the delete button |
| `is_folder_grid` | cookie value | Whether the folder uses grid layout, read from `$cookie("is_folder_grid")` |

### `form.data` Field Properties

| Property | Value | Description |
|----------|-------|-------------|
| `is_input` | boolean | Renders an `<input>` (`database-add`) |
| `is_textarea` | boolean | Renders a `<textarea>` (`database-add`) |
| `is_select` | boolean | `database-list` renders an `<input>` only when `false` |
| `type` | `text`, `date`, ... | The `<input>` `type` |
| `size` | `""`, `_33`, `_25` | Applied as the `<label>` class to control field width |
| `placeholder` | string | Placeholder text |
| `value` | string | Value bound through `:model` |
| `row` | number | `<textarea>` row count |

### Shared Events (`default_events` in `js/index.js`)

| Event | Description |
|-------|-------------|
| `show` | Toggles password visibility and the eye icon |
| `login` | Adds the `show` class to the login block, then sets `is_guest` to `false` after 1 second; the password is not validated |
| `logout` | Sets `is_guest` to `true` |
| `body_left_show` | Toggles `data-show` on `section.body-left` (mobile open/close) |
| `body_left_type` | Toggles `data-min` on `section.body-left` and writes the `is_body_left_min` cookie (desktop collapse) |
| `tab_show` | Toggles `data-show` on a navigation group heading to expand or collapse it; a group containing the selected item collapses on the first click |
| `auto_resize` | Strips newlines from a `<textarea>` and mirrors the value to the parent `data-text` for auto-height |

### Page Events

| Page | Events |
|------|--------|
| `article-add` | `go_back`, `go_forward`, `add_h2`-`add_h6`, `add_bold`, `add_italic`, `add_strikethrough`, `add_underline`, `add_marker`, `add_blockquote`, `add_ul`, `add_ol`, `add_code`, `add_link`, `show_photo_library`, `add_photo` |
| `file-edit` | `go_back`, `go_forward` |
| `json-edit` | `go_back`, `go_forward`, `save`, `change_json_mode` |
| `code-edit` | `go_back`, `go_forward`, `save` |
| `image-folder` | `change_to_list`, `change_to_grid`, `get_image_viewer`, `set_file_name`, `set_file_folder`, `set_file_folder_selected`, `delete_folder`, `delete_file` |

The `article-add` toolbar only has H2-H5 buttons; `add_h6` is defined but unbound. Both the publish and photo-library buttons bind to `show_photo_library`.

### Template Syntax (PDRenderKit Directives Used Here)

| Syntax | Purpose |
|--------|---------|
| `:path="./view/..."` | Load a view partial |
| `:if` / `:else` | Conditional rendering |
| `:for="e in list"`, `:for="(key, val) in obj"` | Loop over an array or object |
| `:model` | Two-way form binding |
| `:html` | Render HTML content |
| `:href`, `:class`, `:type`, `:placeholder`, `:row`, `:data-*` | Attribute binding |
| `@click`, `@input` | Bind a function from `event` |
| `{{ expr }}` | Text interpolation |

### Cookies

| Name | Value | Written by |
|------|-------|------------|
| `is_body_left_min` | `0` / `1` | `body_left_type` |
| `is_folder_grid` | `"0"` / `"1"` | `change_to_list` / `change_to_grid` |

### External Dependencies

| Package | Version | Source |
|---------|---------|--------|
| PDRenderKit | 1.0.0 | `cdn.jsdelivr.net/gh/pardnchiu/PDRenderKit@1.0.0/dist/PDRenderKit.js` |
| PDMarkdownKit | 1.0.0 | `cdn.jsdelivr.net/gh/pardnchiu/PDMarkdownKit@1.0.0/dist/PDMarkdownKit.js` |
| Font Awesome Free | 6.5.2 | `cdn.jsdelivr.net/npm/@fortawesome/fontawesome-free@6.5.2/css/all.min.css` |

### Template Limitations

- No backend or persistence: table rows, folder contents, and editor defaults are hard-coded in HTML and scripts
- Search, export, create, clear, upload, new folder, download, and move buttons are UI only with no bound behavior; the `file-edit` save button has no event
- `set_file_name` only opens a `prompt` and never applies the new name; `delete_file` and `delete_folder` only remove nodes from the page
- The home breadcrumb in `top_tab` and the top-bar logo link point to `/management-1-web`; adjust them when deploying under a different path

***

©️ 2022 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
