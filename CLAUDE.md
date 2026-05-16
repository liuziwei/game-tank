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

- 17×17 网格，每格 32px（`TILE=32`）。游戏区 GW = 544px，右侧面板 PW = 176px，画布总宽 CW = 720px，高 CH = 544px
- 地形类型：`T_EMPTY=0`, `T_BRICK=1`, `T_STEEL=2`, `T_BASE=3`, `T_TREE=4`, `T_WATER=5`
  - 树木(TREE)：坦克可穿过，子弹可穿过（装饰/掩护）
  - 水域(WATER)：坦克不可穿过，子弹可穿过
- 地图数据在 `MAPS` 数组中，是一维数组（行优先，`row * COLS + col`）
- 添加新关卡：在 `MAPS` 数组末尾追加一个 289 个元素的数组即可
- 基地在最后一行（row=16, col=8），周围必须用砖墙保护

## 渲染系统

- 像素艺术风格，使用调色板 `P` 对象（~30种命名颜色，平面色块无渐变）
- `drawMap()` — 砖墙（4块子砖+砖缝）、钢墙（三层+铆钉）、鹰标（16×16 像素点阵）、树木（绿色块）、水域（蓝色+波纹）
- `Tank.draw()` — 用 `ctx.translate/rotate` 转向，履带条纹用 `frame` 计数器做交替动画
- `drawPanel()` — 右侧面板：STAGE 编号、20个敌人图标（红/灰两列）、玩家生命坦克图标
- 所有渲染在 720×544 Canvas 上完成，无 HTML 侧栏

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
- `enRemain` / `enOnField` — 判断胜利条件（`enRemain <= 0 && enOnField <= 0`）
- `difficulty` / `stage` — 难度随关卡递增，影响敌方数量、速度、射击频率
- `frame` — 全局帧计数器，用于履带动画

## 随机关卡生成

`generateRandomMap(diff)` 在预设关卡用尽后调用：
1. 放置基地 + 砖墙保护环
2. 散布水域（2+diff 簇，坦克不能过，子弹能过）
3. 散布树木（4+diff 簇，坦克/子弹都能过）
4. 砖墙簇（12+diff*4 个），避开出生点和基地区
5. 钢墙（3+diff 个，单点放置）
6. BFS 连通性校验（从上三路出生点能否到达底部），失败则移除障碍物重试

## 敌方 AI 要点

EnemyTank 直接访问全局变量 `player`、`enemies`、`map`（不通过构造函数注入，避免玩家重生后引用过期）。方向变换定时器 `dct` 在撞墙或到期时触发。30%概率生成快速坦克（橙色，速度更快）。当与玩家在同行/列且距离 < 8 格时，会主动朝玩家方向射击。

## 开发命令

没有构建/测试命令。修改后直接在浏览器中打开 `index.html` 测试。
