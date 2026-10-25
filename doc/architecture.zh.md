# web-admin-20220917 - 架構

最後更新：2026-10-25

> 返回 [README](./README.zh.md)

## 概覽

```mermaid
graph TB
    Browser["瀏覽器"] --> Page["頁面 HTML（*.html）"]
    Page --> RK["PDRenderKit（CDN）"]
    Page --> FA["Font Awesome（CDN）"]
    Page --> CSS["樣式 css/index.css"]
    Page --> Shared["共用事件 js/index.js"]
    Page --> Script["頁面腳本 js/*.js"]
    Shared --> PD["PD 實例（全域 page）"]
    Script --> PD
    RK --> PD
    PD --> Top["view/body-top.html"]
    PD --> Left["view/body-left.html"]
    PD --> Right["view/body-right/*.html"]
    Script --> MK["PDMarkdownKit（CDN）"]
    PD --> Cookie[("Cookie 介面狀態")]
```

## Module: 頁面外殼

每個 `*.html` 載入相同的 CDN 資源與 `js/index.js`，以 `is_guest` 在登入區塊與主畫面間切換，主畫面由三個局部視圖組成。

```mermaid
graph TB
    subgraph Shell["頁面外殼（*.html）"]
        Head["head：css/index.css、Font Awesome、PDRenderKit、js/index.js、頁面腳本"]
        Body["body#body"]
        Login["登入區塊（:if is_guest）"]
        Main["主畫面（:else）"]
        Body --> Login
        Body --> Main
        Main --> Top[":path view/body-top.html"]
        Main --> Left[":path view/body-left.html"]
        Main --> Right["內容區（:path view/body-right/*.html 或內嵌）"]
    end
    Script["頁面腳本"] -->|"new PD({ id: 'body' })"| Body
```

| 頁面 | 內容區來源 |
|------|------------|
| `database-list`、`database-add`、`article-add`、`image-folder`、`file-edit` | `view/body-right/<頁面>.html` |
| `index`、`json-edit`、`code-edit` | 內嵌於頁面 HTML |

## Module: 共用事件與導覽

`js/index.js` 宣告全域 `page` 與 `default_events`，各頁以展開運算子併入自己的 `event`；導覽列依 `left.*` 旗標標示目前頁面。

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
        G1["資料庫管理：資料列表、資料新增、文章撰寫"]
        G2["檔案管理：檔案夾、一般文檔、JSON"]
        G3["版本資訊：Readme、License"]
    end
    BLT -->|"_cookie"| CookieMin[("is_body_left_min")]
    CookieMin -->|"$cookie"| Left["left.is_body_left_min"]
    Left --> Nav
    Flags["left.is_* 旗標"] -->|":data-selected"| Nav
    Tab --> Nav
    BLS --> Nav
```

## Module: 資料庫頁面

`database-list` 與 `database-add` 不寫 DOM，表單與資料表完全由 PD 資料渲染。

```mermaid
graph TB
    subgraph List["database-list"]
        LForm["form.data"] -->|":for (key, val)"| LInputs["篩選欄位（is_select == false 才渲染）"]
        Head["database.head"] -->|":for"| TH["表頭"]
        Rows["database.list"] -->|":for"| TR["資料列（:class）"]
        TR -->|":html"| TD["儲存格"]
        TR -->|"is_deleted == false"| Del["刪除按鈕"]
    end
    subgraph Add["database-add"]
        AForm["form.data"] -->|":for (key, val)"| Field["欄位"]
        Field -->|"is_input"| Input["input"]
        Field -->|"is_textarea"| Textarea["textarea"]
    end
```

## Module: 編輯器

文章、文檔與 JSON 頁面以 ES Module 匯入 PDMarkdownKit，在 PD 的 `next` 回呼中把編輯器掛進 `section.markdown`。

```mermaid
graph TB
    MK["PDMarkdownKit@1.0.0"] --> Editor["editor（MDEditor）"]
    MK --> Viewer["viewer（MDViewer）"]
    subgraph Article["article-add"]
        Toolbar["工具列事件 add_*"] --> Editor
        Photo["show_photo_library / add_photo"] -->|"addImage"| Editor
        Editor <-->|"editor.viewer / viewer.editor"| Viewer
        Editor -->|"onwheel 同步捲動"| Viewer
    end
    subgraph File["file-edit"]
        Editor2["editor.init(robots.txt 範例)"]
    end
    subgraph Json["json-edit / code-edit"]
        Editor3["editor.init(JSON.stringify(...))"]
        Save["save：getMdTxt → 正規化空白 → JSON.parse"]
        Mode["change_json_mode（僅 json-edit）"]
        Tree["section.json 樹狀欄位"]
        Code["section.markdown 原始碼"]
        Mode --> Tree
        Mode --> Code
        Editor3 --> Code
        Code --> Save
    end
    Editor --> Editor2
    Editor --> Editor3
```

## Module: 檔案夾

`image-folder` 以 `<details>` 呈現資料夾，事件透過 `data-src` 對應檔案並操作預覽與搬移對話框。

```mermaid
graph TB
    subgraph Folder["image-folder"]
        Toggle["change_to_list / change_to_grid"] -->|"data-grid"| ListSec["section.list"]
        Toggle -->|"_cookie"| CG[("is_folder_grid")]
        CG -->|"$cookie"| Grid["is_folder_grid"] --> ListSec
        Item["檔案按鈕（data-src）"] -->|"get_image_viewer"| Viewer["section.folder-viewer"]
        Viewer -->|"副檔名 jpg/jpeg/png/svg/webp"| Img["img 預覽"]
        Rename["set_file_name"] --> Prompt["prompt"]
        Move["set_file_folder"] --> PathDlg["section.folder-path"]
        PathDlg -->|"set_file_folder_selected"| Layer["data-selected 層級"]
        DelF["delete_file"] -->|"confirm"| RemoveF["移除檔案節點"]
        DelD["delete_folder"] -->|"confirm"| RemoveD["移除資料夾節點"]
    end
```

## 資料流

```mermaid
sequenceDiagram
    participant B as 瀏覽器
    participant H as 頁面 HTML
    participant RK as PDRenderKit
    participant I as js/index.js
    participant S as 頁面腳本
    participant PD as PD 實例
    participant V as view/ 局部視圖
    participant MK as PDMarkdownKit
    B->>H: 請求 *.html
    H->>RK: 載入 PDRenderKit.js
    H->>I: 載入 default_events
    H->>S: 載入頁面腳本（部分為 module）
    S->>MK: import editor / viewer（編輯器頁面）
    B->>S: DOMContentLoaded
    S->>PD: new PD({ id, data, event, next })
    PD->>V: 依 :path 載入局部視圖
    PD->>B: 渲染 :if / :for / {{ }} 並綁定 @click
    PD->>S: 呼叫 next()
    S->>MK: 掛載 editor.body 並 init()
```

## 狀態機

### 登入

```mermaid
stateDiagram-v2
    state "主畫面" as Main
    state "登入區塊" as Login
    state "轉場中" as Transition
    [*] --> Main: is_guest = false（預設）
    Main --> Login: logout（is_guest = true）
    Login --> Transition: login（加上 show class）
    Transition --> Main: 1 秒後 is_guest = false
```

### JSON 編輯模式（json-edit）

```mermaid
stateDiagram-v2
    state "樹狀欄位" as Tree
    state "原始碼" as Code
    [*] --> Tree: section.json 顯示
    Tree --> Code: change_json_mode
    Code --> Tree: change_json_mode
```

### 圖片預覽（image-folder）

```mermaid
stateDiagram-v2
    state "隱藏" as Hidden
    state "顯示" as Shown
    [*] --> Hidden
    Hidden --> Shown: get_image_viewer（支援的圖片副檔名，data-show = 1）
    Shown --> Hidden: get_image_viewer（data-show = 0）
```

***

©️ 2022 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
