# LVGL on NuttX：Framebuffer 架构与每帧渲染流程深度解析

## 1. 概述

本文基于 NuttX RTOS + LVGL 图形库的实际代码，从应用入口 `anim_demos.c` 的 `main()` 函数出发，深入分析 LVGL 如何初始化显示系统、管理 Framebuffer、以及每帧画面如何从"脏区域标记"到最终"像素写入屏幕"的完整流程。

文中涉及的核心概念包括：

| 术语 | 含义 |
|------|------|
| **Framebuffer (FB)** | 一块映射到显示控制器的连续内存区域，CPU/GPU 向其中写入像素数据，显示控制器从中读取并驱动屏幕 |
| **Double Buffering** | 使用两块 Framebuffer 交替工作：一块用于显示（front buffer），一块用于渲染（back buffer），避免撕裂 |
| **VSync (垂直同步)** | 显示控制器每刷新完一帧后产生的同步信号，用于协调 buffer 切换时机 |
| **Display (lv_display_t)** | LVGL 中对一个物理显示设备的抽象，包含分辨率、buffer、flush 回调、刷新定时器等 |
| **Flush Callback** | LVGL 渲染完成后调用的回调函数，负责将渲染结果提交给显示硬件 |
| **Invalidation (失效区域)** | UI 元素发生变化时，标记需要重绘的矩形区域（脏区域） |
| **Render Mode** | 渲染模式：`DIRECT`（直接写入 FB）、`PARTIAL`（分块渲染）、`FULL`（全屏重绘） |
| **Pan Display** | 通过修改 FB 的 y 偏移量来切换前后 buffer，实现无撕裂的 buffer 交换 |

## 2. 应用入口与初始化流程

### 2.1 入口函数 main()

**文件路径**: `frameworks/graphics/animengine/examples/anim_demos.c`

```c
int main(int argc, FAR char* argv[])
{
    lv_nuttx_dsc_t info;
    lv_nuttx_result_t result;

    // ① LVGL 核心初始化
    lv_init();

    // ② NuttX 设备描述符初始化（设置 FB 路径、输入设备路径等）
    lv_nuttx_dsc_init(&info);

    // ③ NuttX 平台初始化（打开 FB 设备、创建 display、注册输入设备）
    lv_nuttx_init(&info, &result);

    // ④ 主循环：周期性调用 lv_timer_handler() 驱动渲染
    while (1) {
        lv_timer_handler();
        usleep(10 * 1000);
    }

    // ⑤ 清理
    lv_disp_remove(result.disp);
    lv_deinit();
    return 0;
}
```

整体初始化分为三个阶段，下面逐一展开。

## 3. 阶段一：lv_init() — LVGL 核心初始化

**文件路径**: `apps/graphics/lvgl/lvgl/src/lv_init.c`

`lv_init()` 是 LVGL 的全局初始化入口，负责搭建整个图形引擎的基础设施。与渲染相关的关键步骤如下：

```c
void lv_init(void)
{
    if(lv_initialized) return;  // 防止重复初始化

    // 1. 初始化全局状态 lv_global_t
    LV_GLOBAL_INIT(LV_GLOBAL_DEFAULT());

    // 2. 内存管理初始化
    lv_mem_init();

    // 3. Draw buffer 处理器初始化
    _lv_draw_buf_init_handlers();

    // 4. 定时器核心初始化（渲染刷新依赖定时器驱动）
    _lv_timer_core_init();

    // 5. 动画核心初始化
    _lv_anim_core_init();

    // 6. 绘制引擎初始化（软件渲染 / 硬件加速）
    lv_draw_init();
    lv_draw_sw_init();  // 软件渲染单元

    // 7. 屏幕刷新子系统初始化
    _lv_refr_init();

    // 8. 图像解码器初始化
    _lv_image_decoder_init(LV_CACHE_DEF_SIZE, LV_IMAGE_HEADER_CACHE_DEF_CNT);

    lv_initialized = true;
}
```

**全局状态结构 `lv_global_t`** 在初始化时会创建 display 链表和 indev 链表：

```c
// lv_init.c -> lv_global_init()
_lv_ll_init(&(global->disp_ll), sizeof(lv_display_t));   // display 链表
_lv_ll_init(&(global->indev_ll), sizeof(lv_indev_t));    // 输入设备链表
```

此时还没有任何 display 被创建，也没有 Framebuffer 被分配。这些工作在后续的 NuttX 平台初始化中完成。

## 4. 阶段二：lv_nuttx_dsc_init() — 设备描述符初始化

**文件路径**: `apps/graphics/lvgl/lvgl/src/drivers/nuttx/lv_nuttx_entry.c`

这是一个简单的结构体初始化函数，设置 NuttX 平台的设备路径：

```c
void lv_nuttx_dsc_init(lv_nuttx_dsc_t * dsc)
{
    lv_memzero(dsc, sizeof(lv_nuttx_dsc_t));
    dsc->fb_path    = "/dev/fb0";       // Framebuffer 设备节点
    dsc->input_path = "/dev/input0";    // 触摸屏输入设备
    dsc->utouch_path = "/dev/utouch";   // 用户空间触摸设备（可选）
    dsc->mouse_path  = "/dev/mouse0";   // 鼠标设备（可选）
}
```

**`lv_nuttx_dsc_t` 结构体定义**（`lv_nuttx_entry.h`）：

```c
typedef struct {
    const char * fb_path;       // Framebuffer 设备路径
    const char * input_path;    // 触摸屏设备路径
    const char * utouch_path;   // 用户空间触摸设备路径
    const char * mouse_path;    // 鼠标设备路径
    const char * trace_path;    // Profiler 追踪文件路径
} lv_nuttx_dsc_t;
```

`/dev/fb0` 是 NuttX 内核中 Framebuffer 字符设备驱动注册的标准路径，对应内核驱动文件 `nuttx/drivers/video/fb.c`。

## 5. 阶段三：lv_nuttx_init() — NuttX 平台初始化（核心）

**文件路径**: `apps/graphics/lvgl/lvgl/src/drivers/nuttx/lv_nuttx_entry.c`

这是整个初始化流程中最关键的函数，负责将 LVGL 与 NuttX 的硬件驱动连接起来：

```c
void lv_nuttx_init(const lv_nuttx_dsc_t * dsc, lv_nuttx_result_t * result)
{
    // 1. 分配 NuttX 上下文
    nuttx_ctx_p = lv_malloc_zeroed(sizeof(lv_nuttx_ctx_t));

    // 2. 注册日志回调（输出到 syslog）
    lv_log_register_print_cb(syslog_print);

    // 3. 注册 tick 回调（LVGL 时间源，基于 CLOCK_MONOTONIC）
    lv_tick_set_cb(millis);

    // 4. 检查栈大小是否满足最低要求
    check_stack_size();

    // 5. 缓存和图像缓存初始化
    lv_nuttx_cache_init();
    lv_nuttx_image_cache_init(LV_USE_NUTTX_INDEPENDENT_IMAGE_HEAP);

    // 6. 创建 Display（打开 FB 设备、mmap 映射、设置 buffer）
    if(dsc && dsc->fb_path) {
        lv_display_t * disp = lv_nuttx_fbdev_create();
        lv_nuttx_fbdev_set_file(disp, dsc->fb_path);  // 核心！
        result->disp = disp;
    }

    // 7. 创建输入设备
    if(dsc->input_path) {
        result->indev = lv_nuttx_touchscreen_create(dsc->input_path);
    }
}
```

**tick 回调** 是 LVGL 的时间基准，NuttX 实现使用 `CLOCK_MONOTONIC`：

```c
static uint32_t millis(void)
{
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000 + ts.tv_nsec / 1000000;
}
```

**`lv_nuttx_result_t`** 保存初始化结果，供后续主循环和清理使用：

```c
typedef struct {
    lv_display_t * disp;          // 创建的显示对象
    lv_indev_t * indev;           // 触摸屏输入设备
    lv_indev_t * utouch_indev;    // 用户空间触摸设备
    lv_indev_t * mouse_indev;    // 鼠标输入设备
} lv_nuttx_result_t;
```

### 5.1 Framebuffer 设备创建：lv_nuttx_fbdev_create()

**文件路径**: `apps/graphics/lvgl/lvgl/src/drivers/nuttx/lv_nuttx_fbdev.c`

```c
lv_display_t * lv_nuttx_fbdev_create(void)
{
    // 分配 NuttX FB 驱动私有数据
    lv_nuttx_fb_t * dsc = lv_malloc_zeroed(sizeof(lv_nuttx_fb_t));

    // 创建 LVGL display 对象（默认 800x480，后续会被实际分辨率覆盖）
    lv_display_t * disp = lv_display_create(800, 480);

    dsc->fd = -1;
    lv_display_set_driver_data(disp, dsc);                              // 绑定私有数据
    lv_display_add_event_cb(disp, display_release_cb, LV_EVENT_DELETE, disp); // 注册释放回调
    lv_display_set_flush_cb(disp, flush_cb);                            // 注册 flush 回调
    return disp;
}
```

**`lv_nuttx_fb_t`** 是 NuttX FB 驱动的私有数据结构：

```c
typedef struct {
    int fd;                          // FB 设备文件描述符
    struct fb_videoinfo_s vinfo;     // 视频信息（分辨率、像素格式）
    struct fb_planeinfo_s pinfo;     // 平面信息（内存地址、stride、bpp）
    void * mem;                      // mmap 映射的 FB 内存（buffer 1）
    void * mem2;                     // 第二块 buffer 内存（双缓冲时使用）
    uint32_t mem2_yoffset;           // 第二块 buffer 的 y 偏移量
    lv_draw_buf_t buf1;              // LVGL draw buffer 1
    lv_draw_buf_t buf2;              // LVGL draw buffer 2
} lv_nuttx_fb_t;
```

### 5.2 lv_display_create() — Display 对象创建

**文件路径**: `apps/graphics/lvgl/lvgl/src/display/lv_display.c`

这是 LVGL 核心层创建 display 的函数，与渲染密切相关：

```c
lv_display_t * lv_display_create(int32_t hor_res, int32_t ver_res)
{
    lv_display_t * disp = _lv_ll_ins_head(disp_ll_p);  // 插入 display 链表
    lv_memzero(disp, sizeof(lv_display_t));

    disp->hor_res = hor_res;
    disp->ver_res = ver_res;
    disp->color_format = LV_COLOR_FORMAT_NATIVE;

    // 创建根 layer（所有绘制操作的起点）
    disp->layer_head = lv_malloc(sizeof(lv_layer_t));
    lv_layer_init(disp->layer_head);

    // 初始化同步区域链表（双缓冲同步用）
    _lv_ll_init(&disp->sync_areas, sizeof(lv_area_t));

    // ★ 创建刷新定时器，周期性触发渲染
    disp->refr_timer = lv_timer_create(
        _lv_display_refr_timer,    // 定时器回调 = 渲染入口
        LV_DEF_REFR_PERIOD,        // 默认刷新周期（通常 33ms ≈ 30fps）
        disp                        // user_data = display 指针
    );

    // 创建屏幕层级结构
    disp->bottom_layer = lv_obj_create(NULL);  // 底层
    disp->act_scr      = lv_obj_create(NULL);  // 活动屏幕
    disp->top_layer    = lv_obj_create(NULL);   // 顶层（弹窗等）
    disp->sys_layer    = lv_obj_create(NULL);   // 系统层（鼠标光标等）

    // 标记活动屏幕为"需要重绘"
    lv_obj_invalidate(disp->act_scr);

    // 立即标记定时器就绪，确保首帧立即渲染
    lv_timer_ready(disp->refr_timer);

    return disp;
}
```

**`lv_display_t` 核心字段**（`lv_display_private.h`）：

```c
struct _lv_display_t {
    int32_t hor_res, ver_res;           // 分辨率

    lv_draw_buf_t * buf_1;              // 绘制 buffer 1
    lv_draw_buf_t * buf_2;              // 绘制 buffer 2（双缓冲时非 NULL）
    lv_draw_buf_t * buf_act;            // 当前活动 buffer（渲染目标）

    lv_display_flush_cb_t flush_cb;     // flush 回调函数指针
    lv_display_flush_wait_cb_t flush_wait_cb; // flush 等待回调

    volatile int flushing;              // 是否正在 flush
    volatile int flushing_last;         // 是否是最后一块 flush

    lv_display_render_mode_t render_mode; // 渲染模式

    lv_area_t inv_areas[LV_INV_BUF_SIZE]; // 失效（脏）区域数组
    uint32_t inv_p;                       // 当前脏区域数量

    lv_ll_t sync_areas;                 // 双缓冲同步区域链表
    lv_layer_t * layer_head;            // 根绘制层
    lv_timer_t * refr_timer;            // 刷新定时器
    lv_area_t refreshed_area;           // 当前正在刷新的区域
    uint32_t vsync_count;               // VSync 计数器
    void * driver_data;                 // 驱动私有数据（lv_nuttx_fb_t）
};
```

### 5.3 Framebuffer 映射与 Buffer 配置：lv_nuttx_fbdev_set_file()

**文件路径**: `apps/graphics/lvgl/lvgl/src/drivers/nuttx/lv_nuttx_fbdev.c`

这是将 LVGL display 与 NuttX Framebuffer 硬件绑定的核心函数：

```c
int lv_nuttx_fbdev_set_file(lv_display_t * disp, const char * file)
{
    lv_nuttx_fb_t * dsc = lv_display_get_driver_data(disp);

    // ① 打开 FB 设备
    dsc->fd = open(file, O_RDWR | O_CLOEXEC);  // "/dev/fb0"

    // ② 获取视频信息（分辨率、像素格式）
    ioctl(dsc->fd, FBIOGET_VIDEOINFO, &dsc->vinfo);
    // vinfo.xres, vinfo.yres, vinfo.fmt

    // ③ 获取平面信息（内存地址、stride、bpp）
    ioctl(dsc->fd, FBIOGET_PLANEINFO, &dsc->pinfo);
    // pinfo.fbmem, pinfo.fblen, pinfo.stride, pinfo.bpp

    // ④ 将 FB 内存映射到用户空间
    dsc->mem = mmap(NULL, dsc->pinfo.fblen,
                    PROT_READ | PROT_WRITE,
                    MAP_SHARED | MAP_FILE, dsc->fd, 0);

    // ⑤ 初始化 LVGL draw buffer（直接指向 mmap 的 FB 内存）
    uint32_t w = dsc->vinfo.xres;
    uint32_t h = dsc->vinfo.yres;
    uint32_t stride = dsc->pinfo.stride;
    lv_draw_buf_init(&dsc->buf1, w, h, color_format, stride, dsc->mem, h * stride);

    // ⑥ 检测双缓冲：yres_virtual == yres * 2 表示硬件支持双 buffer
    bool double_buffer = dsc->pinfo.yres_virtual == (dsc->vinfo.yres * 2);
    if(double_buffer) {
        fbdev_init_mem2(dsc);  // mmap 第二块 buffer
        lv_draw_buf_init(&dsc->buf2, w, h, color_format, stride, dsc->mem2, h * stride);
    }

    // ⑦ 配置 display
    lv_display_set_draw_buffers(disp, &dsc->buf1, double_buffer ? &dsc->buf2 : NULL);
    lv_display_set_color_format(disp, color_format);       // RGB565/RGB888/XRGB8888 等
    lv_display_set_render_mode(disp, LV_DISPLAY_RENDER_MODE_DIRECT); // 直接渲染模式
    lv_display_set_resolution(disp, dsc->vinfo.xres, dsc->vinfo.yres);

    // ⑧ 替换刷新定时器回调为 NuttX 专用版本（支持 VSync 等待）
    lv_timer_set_cb(disp->refr_timer, display_refr_timer_cb);
}
```

**关键点解析**：

- **mmap 映射**：LVGL 的 draw buffer 直接指向 Framebuffer 的物理内存（通过 mmap），渲染时 CPU 直接写入 FB 内存，无需额外的内存拷贝。这就是 `DIRECT` 渲染模式的核心优势。

- **双缓冲检测**：通过比较 `yres_virtual`（虚拟分辨率）与 `yres`（物理分辨率）来判断。如果 `yres_virtual == yres * 2`，说明 FB 驱动分配了两倍高度的内存，上半部分是 buffer 1，下半部分是 buffer 2。

- **stride**：每行像素数据占用的字节数（可能因对齐要求大于 `width * bpp/8`）。

- **颜色格式映射**：

```c
static lv_color_format_t fb_fmt_to_color_format(int fmt)
{
    switch(fmt) {
        case FB_FMT_RGB16_565:  return LV_COLOR_FORMAT_RGB565;
        case FB_FMT_RGB24:      return LV_COLOR_FORMAT_RGB888;
        case FB_FMT_RGB32:      return LV_COLOR_FORMAT_XRGB8888;
        case FB_FMT_RGBA32:     return LV_COLOR_FORMAT_ARGB8888;
    }
}
```

## 6. 每帧渲染流程

初始化完成后，主循环通过 `lv_timer_handler()` 周期性驱动渲染。以下是一帧画面从"脏区域标记"到"像素上屏"的完整流程：

```
lv_timer_handler()
  └─ display_refr_timer_cb()          [NuttX 专用，带 VSync 等待]
       └─ _lv_display_refr_timer()    [LVGL 核心渲染入口]
            ├─ lv_obj_update_layout()  [更新布局]
            ├─ lv_refr_join_area()     [合并重叠的脏区域]
            ├─ refr_sync_areas()       [双缓冲同步]
            ├─ refr_invalid_areas()    [遍历脏区域逐个渲染]
            │    └─ refr_area()        [渲染单个区域]
            │         └─ refr_area_part()  [渲染区域的一部分]
            │              ├─ refr_obj_and_children()  [递归绘制对象树]
            │              └─ draw_buf_flush()         [提交渲染结果]
            │                   └─ call_flush_cb()     [调用驱动 flush 回调]
            │                        └─ flush_cb()     [NuttX FB flush 实现]
            │                             ├─ FBIO_UPDATE    [通知硬件更新区域]
            │                             └─ FBIOPAN_DISPLAY [切换前后 buffer]
            └─ [buffer swap]           [交换 buf_act 指针]
```

### 6.1 NuttX 刷新定时器回调：display_refr_timer_cb()

**文件路径**: `apps/graphics/lvgl/lvgl/src/drivers/nuttx/lv_nuttx_fbdev.c`

NuttX 版本的刷新定时器回调在调用 LVGL 核心渲染之前，先通过 `poll()` 检查 FB 设备是否就绪（即前一帧是否已经显示完毕）：

```c
static void display_refr_timer_cb(lv_timer_t * tmr)
{
    lv_display_t * disp = lv_timer_get_user_data(tmr);
    lv_nuttx_fb_t * dsc = lv_display_get_driver_data(disp);
    struct pollfd pfds[1];

    lv_memzero(pfds, sizeof(pfds));
    pfds[0].fd = dsc->fd;
    pfds[0].events = POLLOUT;  // 查询 FB 是否可写

    if(poll(pfds, 1, 0) < 0) return;  // 非阻塞 poll

    // 只有当 FB 设备就绪（POLLOUT）时才执行渲染
    if(pfds[0].revents & POLLOUT) {
        _lv_display_refr_timer(tmr);  // 调用 LVGL 核心渲染
    }
}
```

这里的 `poll(POLLOUT)` 实现了一种**软 VSync 机制**：NuttX FB 驱动在前一帧的 pan buffer 被显示控制器消费后，才会将 fd 标记为可写。这避免了在显示控制器还在读取 buffer 时就开始写入新数据。

### 6.2 LVGL 核心渲染入口：_lv_display_refr_timer()

**文件路径**: `apps/graphics/lvgl/lvgl/src/core/lv_refr.c`

```c
void _lv_display_refr_timer(lv_timer_t * tmr)
{
    disp_refr = tmr->user_data;  // 获取当前 display

    // 检查 draw buffer 是否有效
    lv_draw_buf_t * buf_act = disp_refr->buf_act;
    if(!(buf_act && buf_act->data)) return;

    // 发送 REFR_START 事件
    lv_display_send_event(disp_refr, LV_EVENT_REFR_START, NULL);

    // ① 更新所有屏幕层的布局
    lv_obj_update_layout(disp_refr->act_scr);
    lv_obj_update_layout(disp_refr->top_layer);
    lv_obj_update_layout(disp_refr->sys_layer);

    // ② 合并重叠的脏区域（减少重复绘制）
    lv_refr_join_area();

    // ③ 双缓冲同步：将上一帧的渲染区域从 on-screen buffer 复制到 off-screen buffer
    refr_sync_areas();

    // ④ 遍历所有脏区域，逐个渲染
    refr_invalid_areas();

    // ⑤ 双缓冲模式下，记录本帧渲染区域供下一帧同步
    if(lv_display_is_double_buffered(disp_refr)) {
        for(i = 0; i < disp_refr->inv_p; i++) {
            if(!disp_refr->inv_area_joined[i]) {
                lv_area_t * sync_area = _lv_ll_ins_tail(&disp_refr->sync_areas);
                *sync_area = disp_refr->inv_areas[i];
            }
        }
    }

    // ⑥ 清空脏区域数组
    lv_memzero(disp_refr->inv_areas, sizeof(disp_refr->inv_areas));
    disp_refr->inv_p = 0;

    lv_display_send_event(disp_refr, LV_EVENT_REFR_READY, NULL);
}
```

### 6.3 脏区域合并：lv_refr_join_area()

**文件路径**: `apps/graphics/lvgl/lvgl/src/core/lv_refr.c`

当多个 UI 元素同时变化时，可能产生多个重叠的脏区域。合并算法会将重叠区域合并为一个更大的区域，前提是合并后的面积小于两个区域面积之和（否则合并反而增加绘制量）：

```c
static void lv_refr_join_area(void)
{
    for(join_in = 0; join_in < disp_refr->inv_p; join_in++) {
        for(join_from = 0; join_from < disp_refr->inv_p; join_from++) {
            if(join_in == join_from) continue;

            // 检查两个区域是否重叠
            if(!_lv_area_is_on(&disp_refr->inv_areas[join_in],
                               &disp_refr->inv_areas[join_from])) continue;

            _lv_area_join(&joined_area, &inv_areas[join_in], &inv_areas[join_from]);

            // 只有合并后面积更小时才合并
            if(lv_area_get_size(&joined_area) <
               lv_area_get_size(&inv_areas[join_in]) + lv_area_get_size(&inv_areas[join_from])) {
                lv_area_copy(&inv_areas[join_in], &joined_area);
                inv_area_joined[join_from] = 1;  // 标记为已合并
            }
        }
    }
}
```

### 6.4 双缓冲同步：refr_sync_areas()

**文件路径**: `apps/graphics/lvgl/lvgl/src/core/lv_refr.c`

在双缓冲 DIRECT 模式下，每帧只渲染脏区域。但 buffer 切换后，新的 back buffer（上一帧的 front buffer）中可能缺少上一帧渲染的内容。`refr_sync_areas()` 负责将这些区域从 on-screen buffer 复制到 off-screen buffer：

```c
static void refr_sync_areas(void)
{
    if(disp_refr->render_mode != LV_DISPLAY_RENDER_MODE_DIRECT) return;
    if(!lv_display_is_double_buffered(disp_refr)) return;

    // off_screen = 当前渲染目标（back buffer）
    // on_screen  = 当前显示中的（front buffer）
    lv_draw_buf_t * off_screen = disp_refr->buf_act;
    lv_draw_buf_t * on_screen = (disp_refr->buf_act == disp_refr->buf_1)
                                 ? disp_refr->buf_2 : disp_refr->buf_1;

    // 从 sync_areas 中去除本帧将要重绘的区域（不需要同步）
    // 剩余区域从 on_screen 复制到 off_screen
    for(sync_area = ...) {
        lv_draw_buf_copy(off_screen, sync_area, on_screen, sync_area);
    }
    _lv_ll_clear(&disp_refr->sync_areas);
}
```

### 6.5 渲染提交：draw_buf_flush()

**文件路径**: `apps/graphics/lvgl/lvgl/src/core/lv_refr.c`

每个脏区域渲染完成后，调用 `draw_buf_flush()` 将结果提交给显示驱动：

```c
static void draw_buf_flush(lv_display_t * disp)
{
    lv_layer_t * layer = disp->layer_head;

    // 等待所有绘制任务完成
    while(layer->draw_task_head) {
        lv_draw_dispatch_wait_for_request();
        lv_draw_dispatch();
    }

    // 双缓冲模式下，等待上一次 flush 完成
    if(lv_display_is_double_buffered(disp)) {
        wait_for_flushing(disp_refr);
    }

    disp->flushing = 1;
    disp->flushing_last = (disp->last_area && disp->last_part) ? 1 : 0;

    // 调用驱动注册的 flush 回调
    if(disp->flush_cb) {
        call_flush_cb(disp, &disp->refreshed_area, layer->draw_buf->data);
    }

    // ★ 双缓冲 buffer 交换
    if(lv_display_is_double_buffered(disp) && ...) {
        disp->buf_act = (disp->buf_act == disp->buf_1) ? disp->buf_2 : disp->buf_1;
    }
}
```

`call_flush_cb()` 会加上 display 的偏移量后调用实际的 flush 回调：

```c
static void call_flush_cb(lv_display_t * disp, const lv_area_t * area, uint8_t * px_map)
{
    lv_area_t offset_area = {
        .x1 = area->x1 + disp->offset_x,
        .y1 = area->y1 + disp->offset_y,
        .x2 = area->x2 + disp->offset_x,
        .y2 = area->y2 + disp->offset_y
    };

    lv_display_send_event(disp, LV_EVENT_FLUSH_START, &offset_area);
    disp->flush_cb(disp, &offset_area, px_map);   // → NuttX flush_cb
    lv_display_send_event(disp, LV_EVENT_FLUSH_FINISH, &offset_area);
}
```

### 6.6 NuttX Framebuffer Flush 实现：flush_cb()

**文件路径**: `apps/graphics/lvgl/lvgl/src/drivers/nuttx/lv_nuttx_fbdev.c`

这是整个渲染管线的最后一环，负责通知硬件更新显示：

```c
static void flush_cb(lv_display_t * disp, const lv_area_t * area, uint8_t * color_p)
{
    lv_nuttx_fb_t * dsc = lv_display_get_driver_data(disp);

    // 只在最后一次 flush 时执行硬件操作（DIRECT 模式可能分多次 flush）
    if(!lv_display_flush_is_last(disp)) {
        lv_display_flush_ready(disp);
        return;
    }

    // --- FBIO_UPDATE：通知显示控制器哪个区域需要更新 ---
#if defined(CONFIG_FB_UPDATE)
    int yoffset = (disp->buf_act == disp->buf_1) ? 0 : dsc->mem2_yoffset;

    lv_area_t dirty_area;
    lv_display_get_dirty_area(disp, &dirty_area);

    struct fb_area_s fb_area;
    fb_area.x = dirty_area.x1;
    fb_area.y = dirty_area.y1 + yoffset;
    fb_area.w = lv_area_get_width(&dirty_area);
    fb_area.h = lv_area_get_height(&dirty_area);
    ioctl(dsc->fd, FBIO_UPDATE, &fb_area);  // 通知硬件刷新指定区域
#endif

    // --- FBIOPAN_DISPLAY：双缓冲 buffer 切换 ---
    if(dsc->mem2 != NULL) {
        if(disp->buf_act == disp->buf_1) {
            dsc->pinfo.yoffset = 0;           // 显示 buffer 1（y=0 开始）
        } else {
            dsc->pinfo.yoffset = dsc->mem2_yoffset; // 显示 buffer 2（y=yres 开始）
        }
        ioctl(dsc->fd, FBIOPAN_DISPLAY, &dsc->pinfo);  // 切换显示起始地址
    }

    lv_display_flush_ready(disp);  // 标记 flush 完成
}
```

**两个关键 ioctl 的作用**：

| ioctl | 作用 | NuttX 内核处理 |
|-------|------|---------------|
| `FBIO_UPDATE` | 通知显示控制器指定矩形区域的像素数据已更新，需要刷新到屏幕 | 调用 `fb->vtable->updatearea()` |
| `FBIOPAN_DISPLAY` | 修改显示控制器读取 FB 内存的起始 y 偏移，实现双缓冲切换 | 调用 `fb->vtable->pandisplay()` 并将 paninfo 加入环形队列 |

**NuttX 内核侧处理**（`nuttx/drivers/video/fb.c`）：

```c
// FBIO_UPDATE 处理
case FBIO_UPDATE:
    struct fb_area_s *area = (struct fb_area_s *)arg;
    ret = fb->vtable->updatearea(fb->vtable, area);  // 调用硬件驱动
    break;

// FBIOPAN_DISPLAY 处理
case FBIOPAN_DISPLAY:
    struct fb_planeinfo_s *pinfo = (struct fb_planeinfo_s *)arg;
    if(fb->vtable->pandisplay != NULL) {
        fb->vtable->pandisplay(fb->vtable, pinfo);   // 通知硬件切换 buffer
    }
    ret = fb_add_paninfo(fb, &paninfo, FB_NO_OVERLAY); // 加入 pan 队列
    break;
```

## 7. lv_deinit() — 清理与资源释放

**文件路径**: `apps/graphics/lvgl/lvgl/src/lv_init.c`

`lv_deinit()` 按照初始化的逆序释放所有资源：

```c
void lv_deinit(void)
{
    if(!lv_initialized) return;
    lv_deinit_in_progress = true;

    lv_display_set_default(NULL);

    // 删除所有输入设备和显示设备（触发 display_release_cb）
    _lv_ll_clear_custom(&(global->indev_ll), lv_indev_delete);
    _lv_ll_clear_custom(&(global->disp_ll), lv_display_delete);
    // display_delete 会触发 LV_EVENT_DELETE → display_release_cb()
    //   → munmap(fb_mem) + close(fd)  释放 FB 资源

    // 逆序释放各子系统
    _lv_image_decoder_deinit();
    _lv_refr_deinit();           // 停止刷新
    lv_draw_deinit();            // 释放绘制引擎
    _lv_anim_core_deinit();      // 停止动画
    _lv_timer_core_deinit();     // 停止定时器
    lv_mem_deinit();             // 释放内存池

    lv_initialized = false;
}
```

Display 删除时触发的 FB 资源释放回调：

```c
// lv_nuttx_fbdev.c
static void display_release_cb(lv_event_t * e)
{
    lv_display_t * disp = lv_event_get_user_data(e);
    lv_nuttx_fb_t * dsc = lv_display_get_driver_data(disp);

    if(dsc->mem != NULL && dsc->mem != MAP_FAILED)
        munmap(dsc->mem, dsc->pinfo.fblen);     // 解除 buffer 1 映射

    if(dsc->mem2 != NULL && dsc->mem2 != MAP_FAILED)
        munmap(dsc->mem2, dsc->pinfo.fblen);    // 解除 buffer 2 映射

    if(dsc->fd >= 0)
        close(dsc->fd);                          // 关闭 FB 设备

    lv_free(dsc);                                // 释放私有数据
}
```

## 8. 架构总览图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Application Layer                            │
│  anim_demos.c::main()                                               │
│  lv_init() → lv_nuttx_dsc_init() → lv_nuttx_init() → main loop    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ lv_timer_handler()
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     LVGL Core Rendering Engine                      │
│                                                                     │
│  lv_refr.c                                                          │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ _lv_display_refr_timer()                                    │    │
│  │   ├─ lv_obj_update_layout()    ← 布局计算                   │    │
│  │   ├─ lv_refr_join_area()       ← 脏区域合并                 │    │
│  │   ├─ refr_sync_areas()         ← 双缓冲同步                 │    │
│  │   ├─ refr_invalid_areas()      ← 遍历脏区域                 │    │
│  │   │    └─ refr_area()                                       │    │
│  │   │         └─ refr_area_part()                             │    │
│  │   │              ├─ refr_obj_and_children() ← 对象树绘制     │    │
│  │   │              └─ draw_buf_flush()        ← 提交渲染结果   │    │
│  │   └─ [buffer swap]             ← buf_act 指针切换            │    │
│  └─────────────────────────────────────────────────────────────┘    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ flush_cb()
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    NuttX Display Driver (User Space)                 │
│                                                                     │
│  lv_nuttx_fbdev.c                                                   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ flush_cb()                                                  │    │
│  │   ├─ ioctl(FBIO_UPDATE, &fb_area)    ← 通知硬件更新区域     │    │
│  │   ├─ ioctl(FBIOPAN_DISPLAY, &pinfo)  ← 切换前后 buffer      │    │
│  │   └─ lv_display_flush_ready()        ← 标记 flush 完成      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ display_refr_timer_cb()                                     │    │
│  │   └─ poll(fd, POLLOUT) → _lv_display_refr_timer()           │    │
│  │      (VSync 等待：FB 就绪后才开始渲染)                        │    │
│  └─────────────────────────────────────────────────────────────┘    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ ioctl / mmap
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    NuttX Kernel FB Driver                            │
│                                                                     │
│  nuttx/drivers/video/fb.c                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ fb_ioctl()                                                  │    │
│  │   ├─ FBIOGET_VIDEOINFO  → 返回分辨率、像素格式               │    │
│  │   ├─ FBIOGET_PLANEINFO → 返回内存地址、stride、bpp          │    │
│  │   ├─ FBIO_UPDATE       → vtable->updatearea()               │    │
│  │   ├─ FBIOPAN_DISPLAY   → vtable->pandisplay()               │    │
│  │   └─ FBIO_WAITFORVSYNC → 等待垂直同步信号                    │    │
│  │                                                             │    │
│  │ fb_mmap()  → 将 FB 物理内存映射到用户空间                     │    │
│  │ fb_poll()  → 返回 FB 是否可写（POLLOUT）                      │    │
│  └─────────────────────────────────────────────────────────────┘    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Display Hardware   │
                    │   (LCD Controller)   │
                    └─────────────────────┘
```

## 9. 核心文件索引

| 文件路径 | 职责 |
|---------|------|
| `frameworks/graphics/animengine/examples/anim_demos.c` | 应用入口，调用初始化和主循环 |
| `apps/graphics/lvgl/lvgl/src/lv_init.c` | `lv_init()` / `lv_deinit()` LVGL 全局初始化与清理 |
| `apps/graphics/lvgl/lvgl/src/drivers/nuttx/lv_nuttx_entry.h` | `lv_nuttx_dsc_t` / `lv_nuttx_result_t` 类型定义 |
| `apps/graphics/lvgl/lvgl/src/drivers/nuttx/lv_nuttx_entry.c` | `lv_nuttx_dsc_init()` / `lv_nuttx_init()` / `lv_nuttx_deinit()` |
| `apps/graphics/lvgl/lvgl/src/drivers/nuttx/lv_nuttx_fbdev.c` | FB 设备驱动：`lv_nuttx_fbdev_create()` / `flush_cb()` / `display_refr_timer_cb()` |
| `apps/graphics/lvgl/lvgl/src/display/lv_display.c` | `lv_display_create()` / `lv_display_set_flush_cb()` 等 display API |
| `apps/graphics/lvgl/lvgl/src/display/lv_display_private.h` | `lv_display_t` 结构体完整定义 |
| `apps/graphics/lvgl/lvgl/src/core/lv_refr.c` | 核心渲染引擎：`_lv_display_refr_timer()` / `draw_buf_flush()` / `call_flush_cb()` |
| `nuttx/drivers/video/fb.c` | NuttX 内核 FB 字符设备驱动 |
