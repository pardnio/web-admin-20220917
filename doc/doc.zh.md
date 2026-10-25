# web-admin-20220917 - 技術文件

最後更新：2026-10-25

> 返回 [README](./README.zh.md)

## 前置需求

- 支援 ES Module 的現代瀏覽器（`article-add`、`file-edit`、`json-edit`、`code-edit` 以 `<script type="module">` 載入）
- 靜態 HTTP 伺服器：頁面以 `:path` 載入 `view/` 局部視圖並以 ES Module 匯入套件，直接以 `file://` 開啟無法正常運作
- 可連線至 `cdn.jsdelivr.net`：PDRenderKit、PDMarkdownKit 與 Font Awesome 皆由 CDN 載入
- （選用）Node.js 與 Sass：僅在需要重新編譯樣式時使用

## 安裝

### 從原始碼

```bash
git clone https://github.com/pardnchiu/web-admin-20220917.git
cd web-admin-20220917
```

### 本機預覽

擇一啟動靜態伺服器後開啟 `http://localhost:8080/index.html`：

```bash
python3 -m http.server 8080
```

```bash
npx serve -l 8080 .
```

### 部署

專案沒有建置步驟，將整個目錄上傳至任何靜態主機（例如 GitHub Pages）即可。

### 重新編譯樣式（選用）

`package.json` 的 `sass` 腳本會監看 `sass/` 並壓縮輸出至 `css/`：

```bash
npm install --save-dev sass
npm run sass
```

`sass/` 目錄已列入 `.gitignore`，未包含在儲存庫中；`css/index.css` 為已編譯的成品，沒有原始 Sass 檔時直接修改它即可。

## 使用方式

### 基礎用法

啟動伺服器後由左側導覽列切換頁面：

| 導覽群組 | 項目 | 頁面 |
|----------|------|------|
| 資料庫管理 | 資料列表 | `database-list.html` |
| 資料庫管理 | 資料新增 | `database-add.html` |
| 資料庫管理 | 文章撰寫 | `article-add.html` |
| 檔案管理 | 檔案夾 | `image-folder.html` |
| 檔案管理 | 一般文檔 | `file-edit.html` |
| 檔案管理 | JSON | `json-edit.html` |

`code-edit.html` 不在導覽列中，需直接以網址開啟。

### 新增資料表單欄位

`database-add` 的表單由 `form.data` 渲染，新增一個鍵即會多一個欄位：

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

### 新增一個頁面

1. 建立 `report.html`，沿用既有頁面的登入區塊與主結構，並載入共用腳本與頁面腳本：

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

2. 建立 `js/report.js`，將 `default_events` 展開進 `event`，並提供導覽列需要的 `left` 與 `top_tab`：

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

3. 在 `view/body-left.html` 對應群組的 `<section data-row="N">` 中加入 `<a href="./report.html">`，並把 `data-row` 加一。

### 儲存 JSON 內容

`json-edit` 與 `code-edit` 的 `save` 會先將空白字元正規化再解析，格式錯誤時以 `alert` 提示；要串接後端時，把 `console.log` 換成實際的請求：

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

`/api/content` 為示意端點，本專案不含任何後端。

## 設定參考

### 頁面

| 頁面 | 腳本 | 導覽列選取旗標 | 內容 |
|------|------|----------------|------|
| `index.html` | 內嵌 `<script>` | - | 首頁佔位內容 |
| `database-list.html` | `js/database-list.js` | `left.is_database_list` | 篩選表單、10 筆範例資料表、分頁區塊 |
| `database-add.html` | `js/database-add.js` | `left.is_database_add` | 依 `form.data` 渲染的新增表單 |
| `article-add.html` | `js/article-add.js`（module） | `left.is_article_add` | Markdown 編輯器、即時預覽、圖片庫 |
| `image-folder.html` | `js/image-folder.js` | `left.is_folder_image` | 檔案夾列表、圖片預覽、搬移對話框 |
| `file-edit.html` | `js/file-edit.js`（module） | `left.is_file_edit` | 純文字編輯器，預設內容為 `robots.txt` 範例 |
| `json-edit.html` | `js/json-edit.js`（module） | `left.is_json_edit_input` | 樹狀欄位與原始碼雙模式 JSON 編輯 |
| `code-edit.html` | `js/code-edit.js`（module） | 設定 `left.is_json_edit`，導覽列未讀取 | 僅原始碼模式的 JSON 編輯 |

### PD 資料欄位

| 欄位 | 型別 | 用途 |
|------|------|------|
| `is_guest` | boolean | `true` 顯示登入區塊（`:if`），`false` 顯示主畫面（`:else`）；所有頁面預設 `false` |
| `login.title` | string | 登入區塊標題 |
| `login.password` | string | 以 `:model` 綁定密碼輸入框 |
| `title` | string | 頂部導覽列的系統名稱 |
| `left.is_body_left_min` | cookie 值 | 左側導覽列是否為收合模式，讀自 `$cookie("is_body_left_min")` |
| `left.is_*` | `0` / `1` | 左側導覽列目前選取的項目 |
| `top_tab` | `{ title, href }[]` | 內容區頂部的麵包屑連結 |
| `form.title` | string | 表單標題 |
| `form.data.<key>` | object | 每個鍵渲染為一個欄位，鍵名即欄位標籤 |
| `database.head` | string[] | 資料表欄位標題 |
| `database.list` | `{ class, is_deleted, body }[]` | 資料列；`body` 以 `:html` 輸出，可含 HTML；`is_deleted` 為 `true` 時隱藏刪除按鈕 |
| `is_folder_grid` | cookie 值 | 檔案夾是否為格狀顯示，讀自 `$cookie("is_folder_grid")` |

### `form.data` 欄位屬性

| 屬性 | 值 | 說明 |
|------|----|------|
| `is_input` | boolean | 渲染為 `<input>`（`database-add`） |
| `is_textarea` | boolean | 渲染為 `<textarea>`（`database-add`） |
| `is_select` | boolean | `database-list` 僅在 `false` 時渲染 `<input>` |
| `type` | `text`、`date`... | `<input>` 的 `type` |
| `size` | `""`、`_33`、`_25` | 套用為 `<label>` 的 class，控制欄位寬度 |
| `placeholder` | string | 佔位文字 |
| `value` | string | 以 `:model` 綁定的值 |
| `row` | number | `<textarea>` 的列數 |

### 共用事件（`js/index.js` 的 `default_events`）

| 事件 | 說明 |
|------|------|
| `show` | 切換密碼欄位的顯示／隱藏與眼睛圖示 |
| `login` | 為登入區塊加上 `show` class，1 秒後將 `is_guest` 設為 `false`；不驗證密碼 |
| `logout` | 將 `is_guest` 設為 `true` |
| `body_left_show` | 切換 `section.body-left` 的 `data-show`（行動版開合） |
| `body_left_type` | 切換 `section.body-left` 的 `data-min` 並寫入 Cookie `is_body_left_min`（桌面版收合） |
| `tab_show` | 切換導覽列群組標題的 `data-show` 以展開或收合；群組內含選取項目時會先收合 |
| `auto_resize` | 移除 `<textarea>` 換行並同步到父層 `data-text`，用於自動撐高 |

### 頁面事件

| 頁面 | 事件 |
|------|------|
| `article-add` | `go_back`、`go_forward`、`add_h2`～`add_h6`、`add_bold`、`add_italic`、`add_strikethrough`、`add_underline`、`add_marker`、`add_blockquote`、`add_ul`、`add_ol`、`add_code`、`add_link`、`show_photo_library`、`add_photo` |
| `file-edit` | `go_back`、`go_forward` |
| `json-edit` | `go_back`、`go_forward`、`save`、`change_json_mode` |
| `code-edit` | `go_back`、`go_forward`、`save` |
| `image-folder` | `change_to_list`、`change_to_grid`、`get_image_viewer`、`set_file_name`、`set_file_folder`、`set_file_folder_selected`、`delete_folder`、`delete_file` |

`article-add` 的工具列只放了 H2～H5 按鈕，`add_h6` 已定義但未綁定；「發布」與「圖片庫」按鈕都綁定 `show_photo_library`。

### 樣板語法（本專案用到的 PDRenderKit 指令）

| 語法 | 用途 |
|------|------|
| `:path="./view/..."` | 載入局部視圖 |
| `:if` / `:else` | 條件渲染 |
| `:for="e in list"`、`:for="(key, val) in obj"` | 迴圈渲染陣列或物件 |
| `:model` | 表單雙向綁定 |
| `:html` | 輸出 HTML 內容 |
| `:href`、`:class`、`:type`、`:placeholder`、`:row`、`:data-*` | 屬性綁定 |
| `@click`、`@input` | 綁定 `event` 中的函式 |
| `{{ expr }}` | 文字插值 |

### Cookie

| 名稱 | 值 | 寫入時機 |
|------|----|----------|
| `is_body_left_min` | `0` / `1` | `body_left_type` |
| `is_folder_grid` | `"0"` / `"1"` | `change_to_list` / `change_to_grid` |

### 外部相依

| 套件 | 版本 | 來源 |
|------|------|------|
| PDRenderKit | 1.0.0 | `cdn.jsdelivr.net/gh/pardnchiu/PDRenderKit@1.0.0/dist/PDRenderKit.js` |
| PDMarkdownKit | 1.0.0 | `cdn.jsdelivr.net/gh/pardnchiu/PDMarkdownKit@1.0.0/dist/PDMarkdownKit.js` |
| Font Awesome Free | 6.5.2 | `cdn.jsdelivr.net/npm/@fortawesome/fontawesome-free@6.5.2/css/all.min.css` |

### 範本限制

- 沒有後端與資料持久化：資料列表、檔案夾內容與編輯器預設值皆寫死在 HTML 與腳本中
- 搜尋、匯出、新增、清除、上傳、新增資料夾、下載、搬移等按鈕僅有介面，沒有綁定行為；`file-edit` 的「儲存」按鈕未綁定事件
- `set_file_name` 只跳出 `prompt`，不會套用新名稱；`delete_file`／`delete_folder` 僅移除畫面上的節點
- `top_tab` 的「首頁」與頂部 Logo 連結指向 `/management-1-web`，部署路徑不同時需自行調整

***

©️ 2022 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
