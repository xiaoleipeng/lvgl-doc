



## 整体概念：Display 的屏幕层级模型

LVGL 的 display 采用一种**分层屏幕架构**。可以把它想象成一叠透明玻璃板，从下到上依次是：

```
绘制顺序（从底到顶）：
┌─────────────────────────────────┐
│         sys_layer               │  ← 最顶层，鼠标光标、调试信息
├─────────────────────────────────┤
│         top_layer               │  ← 弹窗、下拉菜单、全局遮罩
├─────────────────────────────────┤
│         act_scr                 │  ← 当前活动屏幕（用户看到的主界面）
│        (prev_scr)               │  ← 切屏动画期间，旧屏幕也参与绘制
├─────────────────────────────────┤
│         bottom_layer            │  ← 最底层，壁纸/背景
└─────────────────────────────────┘
```

下面逐个变量分析。

---

## 1. `screens` 和 `screen_cnt` — 屏幕注册表

```c
lv_obj_t ** screens;     // 动态数组，存储所有屏幕对象的指针
uint32_t screen_cnt;     // 数组中屏幕的数量
```

**含义**：这是 display 上所有屏幕对象的注册表。不仅包括用户创建的业务屏幕，也包括 `bottom_layer`、`top_layer`、`sys_layer` 这三个系统层（它们本质上也是"屏幕"——parent 为 NULL 的 lv_obj）。

**赋值时机**：每当调用 `lv_obj_create(NULL)` 创建一个无父对象时触发注册。

```c
// lv_obj_class.c:60 — 创建 screen 时
if(parent == NULL) {
    // 动态扩容 screens 数组
    lv_obj_t ** screens = lv_realloc(disp->screens,
                          sizeof(lv_obj_t *) * (disp->screen_cnt + 1));
    disp->screen_cnt++;
    disp->screens = screens;
    disp->screens[disp->screen_cnt - 1] = obj;  // 追加到末尾

    // 屏幕坐标覆盖整个 display
    obj->coords.x1 = 0;
    obj->coords.y1 = 0;
    obj->coords.x2 = hor_res - 1;
    obj->coords.y2 = ver_res - 1;
}
```

**删除时**（`lv_obj_tree.c:591`）：

```c
if(obj->parent == NULL) {
    // 从 screens 数组中找到并移除
    for(i = id; i < disp->screen_cnt - 1; i++) {
        disp->screens[i] = disp->screens[i + 1];  // 前移
    }
    disp->screen_cnt--;
    disp->screens = lv_realloc(disp->screens, disp->screen_cnt * sizeof(lv_obj_t *));
}
```

**`lv_display_create` 后的初始状态**：

```
screens[0] = bottom_layer
screens[1] = act_scr      (默认活动屏幕)
screens[2] = top_layer
screens[3] = sys_layer
screen_cnt = 4
```

用户每创建一个新屏幕（`lv_obj_create(NULL)`），`screen_cnt` 就 +1，数组就扩容一个槽位。

---

## 2. `act_scr` — 当前活动屏幕

```c
lv_obj_t * act_scr;   // 当前正在显示的主屏幕
```

**含义**：用户看到的主界面内容。所有通过 `lv_obj_create(lv_screen_active())` 创建的 widget 都挂在这个屏幕下。

**赋值时机**：

| 场景 | 代码位置 | 赋值方式 |
|------|---------|---------|
| display 创建时 | `lv_display_create` | `disp->act_scr = lv_obj_create(NULL)` |
| 无动画切屏 | `scr_load_internal` | `d->act_scr = scr` |
| 动画切屏开始 | `scr_load_anim_start` | `d->act_scr = a->var`（新屏幕） |

```c
// lv_display.c:1184 — 动画开始回调
static void scr_load_anim_start(lv_anim_t * a)
{
    d->prev_scr = lv_screen_active();  // 旧屏幕存入 prev_scr
    d->act_scr = a->var;               // 新屏幕成为 act_scr
}
```

---

## 3. `prev_scr` — 前一个屏幕（切屏动画专用）

```c
lv_obj_t * prev_scr;   // 切屏动画期间保存的旧屏幕
```

**含义**：只在切屏动画进行中有值。动画期间需要同时绘制新旧两个屏幕（一个滑入、一个滑出），`prev_scr` 就是那个"正在退场"的旧屏幕。

**生命周期**：

```
正常状态:     prev_scr = NULL
                  │
动画开始:     prev_scr = 旧的 act_scr    ← scr_load_anim_start()
                  │
动画进行中:   prev_scr 和 act_scr 同时参与渲染
                  │
动画结束:     prev_scr = NULL             ← scr_anim_completed()
              (如果 del_prev=1，旧屏幕被 delete)
```

```c
// lv_display.c:1209 — 动画完成回调
static void scr_anim_completed(lv_anim_t * a)
{
    lv_obj_send_event(d->act_scr, LV_EVENT_SCREEN_LOADED, NULL);
    lv_obj_send_event(d->prev_scr, LV_EVENT_SCREEN_UNLOADED, NULL);

    if(d->prev_scr && d->del_prev) lv_obj_delete(d->prev_scr);  // 自动删除旧屏幕
    d->prev_scr = NULL;           // 清空
    d->draw_prev_over_act = false; // 重置绘制顺序
    d->scr_to_load = NULL;
}
```

---

## 4. `scr_to_load` — 待加载屏幕（防重入）

```c
lv_obj_t * scr_to_load;   // 已经安排要加载但还没完成的屏幕
```

**含义**：当调用 `lv_screen_load_anim` 时，新屏幕先存入 `scr_to_load`，等动画真正开始时才变成 `act_scr`。这个变量主要用于**防止重复加载和处理动画中途被打断的情况**。

**赋值时机**：

```c
// lv_screen_load_anim() 中
d->scr_to_load = new_scr;   // 标记"这个屏幕即将被加载"

// 如果上一个动画还没完成就又触发了新的切屏：
if(d->scr_to_load && act_scr != d->scr_to_load) {
    // 强制完成上一个动画
    lv_anim_delete(d->scr_to_load, NULL);
    scr_load_internal(d->scr_to_load);  // 立即加载
}
```

```c
// scr_load_internal() 中
d->scr_to_load = NULL;   // 加载完成，清空

// scr_anim_completed() 中
d->scr_to_load = NULL;   // 动画完成，清空
```

---

## 5. `bottom_layer` / `top_layer` / `sys_layer` — 三个系统层

```c
lv_obj_t * bottom_layer;   // 最底层
lv_obj_t * top_layer;      // 顶层
lv_obj_t * sys_layer;      // 系统层（最顶）
```

**含义与用途**：

| 层 | Z 序 | 用途 | 可点击 | 有样式 |
|----|------|------|--------|--------|
| `bottom_layer` | 最底 | 壁纸、全局背景色 | 是（默认） | 无（被清除） |
| `top_layer` | 屏幕之上 | 弹窗、下拉菜单、全局遮罩层 | 否 | 无 |
| `sys_layer` | 最顶 | 鼠标光标、性能监视器、调试覆盖层 | 否 | 无 |

**赋值时机**：仅在 `lv_display_create` 中创建一次，之后不再改变：

```c
// lv_display.c:121-133
disp->bottom_layer = lv_obj_create(NULL);
disp->act_scr      = lv_obj_create(NULL);
disp->top_layer    = lv_obj_create(NULL);
disp->sys_layer    = lv_obj_create(NULL);

// 系统层去掉所有默认样式（透明背景）
lv_obj_remove_style_all(disp->bottom_layer);
lv_obj_remove_style_all(disp->top_layer);
lv_obj_remove_style_all(disp->sys_layer);

// top 和 sys 层不响应点击（事件穿透到下面的屏幕）
lv_obj_remove_flag(disp->top_layer, LV_OBJ_FLAG_CLICKABLE);
lv_obj_remove_flag(disp->sys_layer, LV_OBJ_FLAG_CLICKABLE);
```

**渲染时的绘制顺序**（`refr_area_part`，`lv_refr.c:970`）：

```c
// 1. 如果没有不透明对象完全覆盖脏区域，先画 bottom_layer
if(top_act_scr == NULL && top_prev_scr == NULL) {
    refr_obj_and_children(layer, bottom_layer);
}

// 2. 画 act_scr（和可能的 prev_scr，顺序取决于 draw_prev_over_act）
// ...

// 3. 无条件画 top_layer 和 sys_layer
refr_obj_and_children(layer, top_layer);
refr_obj_and_children(layer, sys_layer);
```

---

## 6. `draw_prev_over_act` — 新旧屏幕的绘制顺序

```c
uint8_t draw_prev_over_act : 1;   // 1: prev_scr 画在 act_scr 上面
```

**含义**：切屏动画期间，决定旧屏幕和新屏幕谁在上面。

**赋值时机**：

```c
// lv_screen_load_anim() 中
d->draw_prev_over_act = is_out_anim(anim_type);
```

```c
static bool is_out_anim(lv_screen_load_anim_t anim_type)
{
    return anim_type == LV_SCR_LOAD_ANIM_FADE_OUT  ||
           anim_type == LV_SCR_LOAD_ANIM_OUT_LEFT  ||
           anim_type == LV_SCR_LOAD_ANIM_OUT_RIGHT ||
           anim_type == LV_SCR_LOAD_ANIM_OUT_TOP   ||
           anim_type == LV_SCR_LOAD_ANIM_OUT_BOTTOM;
}
```

**逻辑**：

- `OVER_LEFT`（新屏幕从右滑入覆盖旧屏幕）→ `draw_prev_over_act = 0`
  - 绘制顺序：先画 prev_scr（底），再画 act_scr（顶）→ 新屏幕盖住旧屏幕
- `OUT_LEFT`（旧屏幕向左滑出露出新屏幕）→ `draw_prev_over_act = 1`
  - 绘制顺序：先画 act_scr（底），再画 prev_scr（顶）→ 旧屏幕盖住新屏幕

对应渲染代码：

```c
// lv_refr.c:976
if(disp_refr->draw_prev_over_act) {
    // OUT 类动画：先画新屏幕，再画旧屏幕（旧的在上面滑走）
    refr_obj_and_children(layer, act_scr);
    refr_obj_and_children(layer, prev_scr);
} else {
    // OVER 类动画：先画旧屏幕，再画新屏幕（新的在上面滑入）
    refr_obj_and_children(layer, prev_scr);
    refr_obj_and_children(layer, act_scr);
}
```

动画结束后重置：

```c
// scr_anim_completed()
d->draw_prev_over_act = false;
```

---

## 7. `del_prev` — 动画结束后是否自动删除旧屏幕

```c
uint8_t del_prev : 1;   // 1: 动画完成后自动 delete prev_scr
```

**含义**：由 `lv_screen_load_anim` 的 `auto_del` 参数控制。如果为 1，动画结束后旧屏幕会被自动释放内存。

**赋值时机**：

```c
// lv_screen_load_anim() 中
d->del_prev = auto_del;   // 用户传入的参数
```

**消费时机**：

```c
// scr_anim_completed() — 动画完成
if(d->prev_scr && d->del_prev) lv_obj_delete(d->prev_scr);
d->prev_scr = NULL;
```

以及在连续切屏时的防护：

```c
// lv_screen_load_anim() 开头 — 如果上一个动画的旧屏幕还在
if(d->prev_scr && d->del_prev) {
    lv_obj_delete(d->prev_scr);   // 立即删除
    d->prev_scr = NULL;
}
```

---

## 完整状态机：一次带动画的切屏

以 `lv_screen_load_anim(scr_B, LV_SCR_LOAD_ANIM_OVER_LEFT, 300, 0, true)` 为例：

```
时刻 T0: 调用 lv_screen_load_anim
┌──────────────────────────────────────────────────┐
│ act_scr     = scr_A（当前屏幕）                    │
│ prev_scr    = NULL                                │
│ scr_to_load = scr_B                               │
│ del_prev    = 1（auto_del=true）                   │
│ draw_prev_over_act = 0（OVER_LEFT 不是 out 动画）   │
│ screens = [bottom, scr_A, scr_B, top, sys]        │
│ screen_cnt = 5                                    │
└──────────────────────────────────────────────────┘

时刻 T1: 动画开始（scr_load_anim_start 回调）
┌──────────────────────────────────────────────────┐
│ act_scr     = scr_B  ← 新屏幕成为活动屏幕          │
│ prev_scr    = scr_A  ← 旧屏幕保存到 prev_scr       │
│ scr_to_load = scr_B                               │
│                                                    │
│ 渲染顺序（draw_prev_over_act=0）：                  │
│   bottom → scr_A(prev) → scr_B(act) → top → sys  │
│   scr_B 从右侧 x=466 滑到 x=0，逐渐覆盖 scr_A     │
└──────────────────────────────────────────────────┘

时刻 T2: 动画进行中（每帧渲染）
┌──────────────────────────────────────────────────┐
│ scr_B 的 x 坐标从 466 逐渐变为 0                   │
│ 两个屏幕同时可见，scr_B 在 scr_A 上面               │
└──────────────────────────────────────────────────┘

时刻 T3: 动画完成（scr_anim_completed 回调）
┌──────────────────────────────────────────────────┐
│ act_scr     = scr_B                               │
│ prev_scr    = NULL   ← 清空                       │
│ scr_to_load = NULL   ← 清空                       │
│ del_prev    = 1 → lv_obj_delete(scr_A)            │
│ draw_prev_over_act = false ← 重置                  │
│ screens = [bottom, scr_B, top, sys]               │
│ screen_cnt = 4  ← scr_A 被删除，数组缩小            │
└──────────────────────────────────────────────────┘
```