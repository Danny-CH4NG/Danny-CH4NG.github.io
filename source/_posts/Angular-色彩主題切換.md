---
title: Angular-色彩主題切換
date: 2021-08-30 20:40:22
tags:
  - Material-UI
categories:
  - Angular
---

簡單介紹一下色彩切換功能
![](https://i.imgur.com/aNhBD8b.gif)

### 建立2個獨立的主題
```
$dark-primary: mat-palette($dark-primary, 900);
$dark-accent:  mat-palette($mat-cyan);
$dark-warn:    mat-palette($mat-red);

$dark-theme: mat-dark-theme($dark-primary, $dark-accent, $dark-warn);
// class包裹其他樣式
.dark-theme {
  @include angular-material-theme($dark-theme);
}

$light-primary: mat-palette($mat-indigo);
$light-accent:  mat-palette($mat-lime);
$light-warn:    mat-palette($mat-pink);

$light-theme: mat-light-theme($light-primary, $light-accent, $light-warn);
// class包裹其他樣式
.light-theme {
  @include angular-material-theme($light-theme);
}
```

### 在app.component建立切換功能
material-UI中，menu、dialog等元件在OverlayContainer，與app.component平級，所以額外改變其樣式

#### html
```
<div [class]="theme">
  <router-outlet></router-outlet>
</div>
```

#### ts
```
import { Component, OnInit } from '@angular/core';

import { OverlayContainer } from '@angular/cdk/overlay';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.scss']
})
export class AppComponent implements OnInit {

  constructor(private overlayContainer: OverlayContainer) {}

  ngOnInit() {
    this.overlayContainer.getContainerElement().classList.add(this.theme);
    toggleTheme('light-theme') // 看想要怎麼觸發
  }

  theme = 'dark-theme';

  toggleTheme(newTheme) {
    this.overlayContainer.getContainerElement().classList.remove(this.theme);
    this.overlayContainer.getContainerElement().classList.add(newTheme);
    this.theme = newTheme;
  }
}
```
