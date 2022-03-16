---
title: Angular-在Angular中導入字體
date: 2022-03-14 22:48:26
tags:
  - CSS
categories:
  - Angular
---

### 取得字體樣式文件
我們以常用的Google Noto Fonts作為舉例
https://fonts.google.com/noto/fonts
選擇任意自行後，下載得到附檔名為.otf的檔案(或是.ttf也可以)
將檔案放入/assets/fonts資料夾

### 建立css
在/assets/fonts中，建立一個.css，使用@font-face的at-rule描述字體。
* font-family: 未來引用的名稱
* src: 字體
```
@font-face {
    font-family: 'Orbitron';
    src: url('./Orbitron-VariableFont_wght.ttf');
}
```

### 現在你可以在任意文件中使用自訂字體並生效
```
.any-text{
    font-family: "Orbitron"
  }
```
