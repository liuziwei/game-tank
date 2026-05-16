# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

单文件 HTML5 Canvas 坦克大战游戏，零依赖，浏览器直接打开 `index.html` 即可运行。仓库地址：https://github.com/liuziwei/game-tank。

## 游戏架构

整个游戏在一个 60fps `requestAnimationFrame` 循环中运行，核心是状态机：

```
STATE_TITLE → STATE_PLAYING → STATE_VICTORY → (下一关) STATE_PLAYING
                   ↓
            STATE_PAUSED (P键切换)
                   ↓
            STATE_GAMEOVER → STATE_TITLE
```

游戏循环 `gameLoop()` 中每个 state 有独立的 update/render 分支，互不交叉。

## 类继承关系

```
Tank (基类: 移动、碰撞检测 canMoveTo、射击 shoot、渲染 draw)
├── PlayerTank (键盘控制，无敌帧 invincible，滑墙移动)
└── EnemyTank (AI 寻路：随机转向 + 40%概率朝向玩家，自动射击)
```

- `Bullet` — 子弹，每辆坦克同时最多1颗（由 shootCooldown 限制）
- `Particle` — 爆炸粒子特效，life 衰减到0后清理

## 地图系统

- 13×13 网格，每格 32px（`TILE=32`）
- 地形类型：`TILE_EMPTY=0`, `TILE_BRICK=1`, `TILE_STEEL=2`, `TILE_BASE=3`
- 地图数据在 `MAPS` 数组中，是一维数组（行优先，`row * COLS + col`）
- 添加新关卡：在 `MAPS` 数组末尾追加一个 169 个元素的数组即可
- 基地在最后一行（row=12, col=6），周围必须用砖墙保护

## 碰撞检测

- `Tank.canMoveTo(nx, ny, map, allTanks)` — 检查四个角点是否落在障碍物 tile 上（含2px内边距），以及与其他坦克的曼哈顿距离
- 子弹碰撞用 `Math.floor(bullet.x / TILE)` 定位到 tile，AABB 检测坦克
- 地图坐标：`(col, row)`，像素坐标：`(col*TILE + HALF_TILE, row*TILE + HALF_TILE)`

## 关键全局变量

- `map` — 当前关卡地图（可变，砖墙被打掉会写入 TILE_EMPTY）
- `player` — PlayerTank 实例，死亡重生时会重新 new
- `enemies` — EnemyTank 数组
- `bullets` / `particles` — 每帧 update + filter 清理
- `keys` — 按键状态对象（keydown 设 true，keyup 设 false）
- `enemiesRemaining` / `enemiesOnField` — 用于判断胜利条件

## 敌方 AI 要点

EnemyTank 直接访问全局变量 `player`、`enemies`、`map`（不通过构造函数注入，避免玩家重生后引用过期）。方向变换定时器 `dirChangeTimer` 在撞墙或到期时触发。

## 开发命令

没有构建/测试命令。修改后直接在浏览器中打开 `index.html` 测试。
