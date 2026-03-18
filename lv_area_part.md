

让我先读取 `refr_area_part` 的完整实现，以及它调用的关键函数。

Now let me also read `lv_refr_get_top_obj` and `wait_for_flushing`:

现在我有了所有需要的代码。下面来详细分析。

---

## `refr_area_part` 详细分析

`refr_area_part` 是 LVGL 渲染管线中"单次绘制"的核心函数。它负责将一个 clip 区域内的所有可见对象绘制到 draw buffer 中，然后触发 flush。

### 一、函数签名与调用上下文

```c
static void refr_area_part(lv_layer_t * layer)
```

由 `refr_area()` 调用，调用前 `refr_area` 已经设置好了 `layer` 的关键字段：
- `layer->draw_buf` — 指向当前活跃的 draw buffer（`buf_act`）
- `layer->buf_area` — draw buffer 覆盖的屏幕坐标范围
- `layer->_clip_area` — 本次需要绘制的脏区（裁剪区域）
- `layer->phy_clip_area` — 物理裁剪区域（旋转前）

在你的设备上（466×466, DIRECT 模式, 双缓冲），`refr_area` 中走的是 DIRECT 分支：

```c
// refr_area() 中 DIRECT 模式的调用
disp_refr->last_part = disp_refr->last_area;  // 只有最后一个脏区才 last_part=1
layer->_clip_area = *area_p;                    // clip 就是当前脏区本身
layer->phy_clip_area = *area_p;
refr_area_part(layer);
```

### 二、逐段分析

#### 2.1 记录刷新区域 + 重置 layer

```c
disp_refr->refreshed_area = layer->_clip_area;
lv_layer_reset(layer);
```

- `refreshed_area` 记录本次刷新的区域，后续 `flush_cb` 中会用到（NuttX 驱动里用它计算 `FBIO_UPDATE` 和 `FBIOPAN_DISPLAY` 的参数）
- `lv_layer_reset` 清空 layer 上的 draw task 链表和矩阵，为新一轮绘制做准备

#### 2.2 矩阵旋转处理（可选）

```c
#if LV_DRAW_TRANSFORM_USE_MATRIX
    if(lv_display_get_matrix_rotation(disp_refr)) {
        const lv_display_rotation_t rotation = lv_display_get_rotation(disp_refr);
        if(rotation != LV_DISPLAY_ROTATION_0) {
            lv_display_rotate_area(disp_refr, &layer->phy_clip_area);
            // 直接赋值旋转矩阵，避免浮点运算精度损失
            switch(rotation) {
                case LV_DISPLAY_ROTATION_90:
                    layer->matrix.m[0][0] = 0;  layer->matrix.m[0][1] = 1;  ...
```

如果开启了 `LV_DRAW_TRANSFORM_USE_MATRIX` 且 `lv_display_set_matrix_rotation(disp, true)`（你的设备在 `lv_nuttx_fbdev_set_file` 末尾有设置），则：
1. 将 `phy_clip_area` 旋转到物理坐标系
2. 直接给 `layer->matrix` 赋值 2D 仿射变换矩阵（不用 `lv_matrix_rotate` + `lv_matrix_translate`，减少精度损失）

注意：LVGL 定义的旋转方向和绘制角度是相反的，所以 90° 旋转对应的矩阵实际是 270° 的绘制变换。

#### 2.3 单缓冲等待 + 透明屏幕清空

```c
if(!lv_display_is_double_buffered(disp_refr)) {
    wait_for_flushing(disp_refr);
}
```

单缓冲模式下，必须等上一次 flush 完成才能往 buffer 里画新内容，否则会出现撕裂。你的设备是双缓冲，这里跳过。

```c
if(lv_color_format_has_alpha(disp_refr->color_format)) {
    lv_area_t a = disp_refr->refreshed_area;
    if(disp_refr->render_mode == LV_DISPLAY_RENDER_MODE_PARTIAL) {
        lv_area_move(&a, -disp_refr->refreshed_area.x1, -disp_refr->refreshed_area.y1);
    }
    lv_draw_buf_clear(layer->draw_buf, &a);
}
```

如果颜色格式带 alpha 通道（如 ARGB8888），需要先把绘制区域清零（透明），否则残留数据会导致混合错误。你的设备是 `XRGB8888`（`FB_FMT_RGB32`, fmt=13），X 通道被忽略，所以这里也跳过。

#### 2.4 核心：Top Object 优化（最关键的性能优化）

```c
lv_obj_t * top_act_scr = NULL;
lv_obj_t * top_prev_scr = NULL;

top_act_scr = lv_refr_get_top_obj(&layer->_clip_area, lv_display_get_screen_active(disp_refr));
if(disp_refr->prev_scr) {
    top_prev_scr = lv_refr_get_top_obj(&layer->_clip_area, disp_refr->prev_scr);
}
```

`lv_refr_get_top_obj` 是一个递归搜索函数，它从 screen 对象开始，自顶向下找到能完全覆盖 `_clip_area` 的最上层不透明对象。核心逻辑：

```c
lv_obj_t * lv_refr_get_top_obj(const lv_area_t * area_p, lv_obj_t * obj)
{
    // 1. obj 的坐标必须完全包含 area_p
    if(_lv_area_is_in(area_p, &obj->coords, 0) == false) return NULL;
    // 2. 不能隐藏
    if(lv_obj_has_flag(obj, LV_OBJ_FLAG_HIDDEN)) return NULL;
    // 3. 不能是特殊 layer 类型
    if(_lv_obj_get_layer_type(obj) != LV_LAYER_TYPE_NONE) return NULL;
    // 4. 不能半透明
    if(lv_obj_get_style_opa(obj, LV_PART_MAIN) < LV_OPA_MAX) return NULL;
    // 5. 发送 COVER_CHECK 事件，检查 obj 是否真的能完全覆盖该区域
    //    （考虑圆角、背景图等因素）
    lv_cover_check_info_t info;
    info.res = LV_COVER_RES_COVER;
    info.area = area_p;
    lv_obj_send_event(obj, LV_EVENT_COVER_CHECK, &info);
    if(info.res == LV_COVER_RES_MASKED) return NULL;

    // 6. 从最上面的子对象开始递归查找
    for(i = child_cnt - 1; i >= 0; i--) {
        found_p = lv_refr_get_top_obj(area_p, child);
        if(found_p != NULL) break;  // 找到了就停
    }

    // 7. 如果子对象没有更好的，且自己能覆盖，就返回自己
    if(found_p == NULL && info.res == LV_COVER_RES_COVER) {
        found_p = obj;
    }
    return found_p;
}
```

这个优化的意义：如果 clip 区域被某个不透明对象完全覆盖，那么该对象下面的所有对象都不需要绘制。比如一个全屏不透明按钮覆盖了整个脏区，那么 screen 背景就不用画了。

返回值含义：
- `top_act_scr != NULL` → 找到了能覆盖 clip 区域的最上层对象，从它开始画就行
- `top_act_scr == NULL` → 没有任何对象能完全覆盖，需要从 screen 本身开始画

#### 2.5 绘制层级顺序（Z-order）

这是 `refr_area_part` 最核心的绘制调度逻辑，分两种情况：

```
绘制层级（从底到顶）：
┌─────────────────────────────────┐
│  sys_layer    (系统层，最顶)     │  ← 始终绘制
│  top_layer    (顶层)            │  ← 始终绘制
│  ─────────────────────────────  │
│  act_scr / prev_scr (屏幕层)    │  ← 顺序取决于 draw_prev_over_act
│  ─────────────────────────────  │
│  bottom_layer (底层，最底)       │  ← 仅当无 top_obj 时绘制
└─────────────────────────────────┘
```

**情况 A：`draw_prev_over_act == true`（OUT 类动画）**

```c
if(disp_refr->draw_prev_over_act) {
    // 先画 act_scr（新屏幕，在下面）
    if(top_act_scr == NULL) top_act_scr = disp_refr->act_scr;
    refr_obj_and_children(layer, top_act_scr);

    // 再画 prev_scr（旧屏幕，在上面，正在滑出/淡出）
    if(disp_refr->prev_scr) {
        if(top_prev_scr == NULL) top_prev_scr = disp_refr->prev_scr;
        refr_obj_and_children(layer, top_prev_scr);
    }
}
```

OUT 类动画（如 `LV_SCR_LOAD_ANIM_OUT_LEFT`）：旧屏幕在上面滑出，新屏幕在下面露出。所以先画新屏幕，再画旧屏幕覆盖在上面。

**情况 B：`draw_prev_over_act == false`（OVER/MOVE/FADE_IN 类动画，或无动画）**

```c
else {
    // 先画 prev_scr（旧屏幕，在下面）
    if(disp_refr->prev_scr) {
        if(top_prev_scr == NULL) top_prev_scr = disp_refr->prev_scr;
        refr_obj_and_children(layer, top_prev_scr);
    }

    // 再画 act_scr（新屏幕，在上面，正在滑入/淡入）
    if(top_act_scr == NULL) top_act_scr = disp_refr->act_scr;
    refr_obj_and_children(layer, top_act_scr);
}
```

OVER 类动画（如 `LV_SCR_LOAD_ANIM_OVER_LEFT`）：新屏幕在上面滑入，旧屏幕在下面不动。所以先画旧屏幕，再画新屏幕覆盖。

**无动画时**：`prev_scr == NULL`，只画 `act_scr`。

**bottom_layer 的条件**：

```c
if(top_act_scr == NULL && top_prev_scr == NULL) {
    refr_obj_and_children(layer, lv_display_get_layer_bottom(disp_refr));
}
```

只有当 act_scr 和 prev_scr 都没有找到能完全覆盖 clip 区域的对象时，才需要画 bottom_layer。因为如果有对象能完全覆盖，bottom_layer 的内容反正看不见，画了也白画。

**top_layer 和 sys_layer 始终绘制**：

```c
refr_obj_and_children(layer, lv_display_get_layer_top(disp_refr));
refr_obj_and_children(layer, lv_display_get_layer_sys(disp_refr));
```

这两层在所有屏幕之上，无条件绘制。典型用途：
- `top_layer`：弹窗、下拉菜单、Toast 等
- `sys_layer`：鼠标光标、性能监视器等系统级 UI

#### 2.6 `refr_obj_and_children` 的绘制策略

```c
static void refr_obj_and_children(lv_layer_t * layer, lv_obj_t * top_obj)
{
    lv_obj_t * parent;
    lv_obj_t * border_p = top_obj;
    parent = lv_obj_get_parent(top_obj);

    // 先画 top_obj 自身及其所有子对象
    lv_obj_refr(layer, top_obj);

    // 然后向上遍历父链，画 top_obj 的"弟弟"们（z-order 更高的兄弟）
    while(parent != NULL) {
        for(i = 0; i < child_cnt; i++) {
            lv_obj_t * child = parent->spec_attr->children[i];
            if(!go) {
                if(child == border_p) go = true;  // 找到 border_p，之后的兄弟才画
            } else {
                lv_obj_refr(layer, child);  // 画"弟弟"
            }
        }
        // 父对象的 post-draw 事件
        lv_obj_send_event(parent, LV_EVENT_DRAW_POST_BEGIN, layer);
        lv_obj_send_event(parent, LV_EVENT_DRAW_POST, layer);
        lv_obj_send_event(parent, LV_EVENT_DRAW_POST_END, layer);

        border_p = parent;
        parent = lv_obj_get_parent(parent);
    }
}
```

这个函数的逻辑很精妙：
1. 先画 `top_obj` 及其子树（`lv_obj_refr` 会递归画子对象）
2. 然后向上回溯到父对象，画 `top_obj` 之后的兄弟（children 数组中 index 更大的 = z-order 更高的）
3. 继续向上，画父对象的兄弟中 z-order 更高的
4. 一直到 screen 根节点

这样就实现了"从 top_obj 开始，只画它和它上面的对象"的效果，跳过了 top_obj 下面的所有对象。

#### 2.7 Flush 到显示设备

```c
draw_buf_flush(disp_refr);
```

`draw_buf_flush` 的关键逻辑：

```c
static void draw_buf_flush(lv_display_t * disp)
{
    // 1. 等待所有 draw task 完成（GPU 异步绘制的情况）
    while(layer->draw_task_head) {
        lv_draw_dispatch_wait_for_request();
        lv_draw_dispatch();
    }

    // 2. 双缓冲模式下等待上一次 flush 完成
    if(lv_display_is_double_buffered(disp)) {
        wait_for_flushing(disp_refr);
    }

    // 3. 设置 flushing 标志
    disp->flushing = 1;

    // 4. 关键：设置 flushing_last 标志
    if(disp->last_area && disp->last_part) disp->flushing_last = 1;
    else disp->flushing_last = 0;

    // 5. 调用 flush_cb
    if(disp->flush_cb) {
        call_flush_cb(disp, &disp->refreshed_area, layer->draw_buf->data);
    }

    // 6. 双缓冲交换（DIRECT 模式只在最后一个区域交换）
    if(lv_display_is_double_buffered(disp) &&
       (disp->render_mode != LV_DISPLAY_RENDER_MODE_DIRECT || flushing_last)) {
        if(disp->buf_act == disp->buf_1)
            disp->buf_act = disp->buf_2;
        else
            disp->buf_act = disp->buf_1;
    }
}
```

在你的设备上（DIRECT + 双缓冲），`flush_cb` 就是 `lv_nuttx_fbdev.c` 中的：

```c
static void flush_cb(lv_display_t * disp, const lv_area_t * area, uint8_t * color_p)
{
    // 非最后一次 flush，直接跳过（DIRECT 模式的关键优化）
    if(!lv_display_flush_is_last(disp)) {
        lv_display_flush_ready(disp);
        return;
    }
    // 最后一次 flush：ioctl(FBIOPAN_DISPLAY) 切换前后缓冲
    if(dsc->mem2 != NULL) {
        dsc->pinfo.yoffset = (disp->buf_act == disp->buf_1) ? 0 : dsc->mem2_yoffset;
        ioctl(dsc->fd, FBIOPAN_DISPLAY, &dsc->pinfo);
    }
    lv_display_flush_ready(disp);
}
```

### 三、你的设备上的完整执行流程

以一帧有 3 个脏区（inv_p=3, 其中 area[1] 被 join 标记）为例：

```
refr_invalid_areas()
  ├── last_i = 2 (最后一个未 join 的区域)
  │
  ├── i=0: inv_area_joined[0]=0, last_area=0
  │   └── refr_area(area[0])
  │       ├── DIRECT 模式: last_part = last_area = 0
  │       └── refr_area_part(layer)
  │           ├── 找 top_obj → 绘制对象树
  │           └── draw_buf_flush()
  │               ├── flushing_last = (0 && 0) = 0
  │               ├── flush_cb → flush_is_last()=false → 直接 return（不 pan）
  │               └── 不交换 buf（DIRECT 模式 + 非 last）
  │
  ├── i=1: inv_area_joined[1]=1 → 跳过
  │
  └── i=2: inv_area_joined[2]=0, last_area=1
      └── refr_area(area[2])
          ├── DIRECT 模式: last_part = last_area = 1
          └── refr_area_part(layer)
              ├── 找 top_obj → 绘制对象树
              └── draw_buf_flush()
                  ├── flushing_last = (1 && 1) = 1  ← 最后一次！
                  ├── flush_cb → flush_is_last()=true
                  │   └── ioctl(FBIOPAN_DISPLAY) → 切换前后缓冲
                  └── 交换 buf_act: buf_1 ↔ buf_2
```

### 四、总结

`refr_area_part` 的核心职责可以概括为 5 步：

1. 旋转处理（如果启用了矩阵旋转）
2. 同步等待（单缓冲等 flush 完成；透明格式清空 buffer）
3. Top Object 优化（`lv_refr_get_top_obj` 找到最上层不透明覆盖对象，跳过其下方所有绘制）
4. 按 Z-order 绘制对象树（bottom_layer → screen(s) → top_layer → sys_layer），屏幕转场时通过 `draw_prev_over_act` 控制新旧屏幕的绘制顺序
5. 触发 flush（`draw_buf_flush` → `flush_cb` → NuttX `FBIOPAN_DISPLAY`），DIRECT 模式下只有最后一个脏区才真正执行 pan 操作和缓冲交换