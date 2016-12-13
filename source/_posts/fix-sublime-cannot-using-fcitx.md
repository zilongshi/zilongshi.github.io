---
title: Linux下修复Sublime Text 3无法使用Fcitx搜狗拼音的方法
date: 2016-06-27 14:57:14
tags: "Linux"
---

通过启动sublime text 3时指定LD_PRELOAD来修复ubuntu 16.04中sublime text 3不支持fcitx中文输入法的问题。
Sublime Text 3安装路径为/opt/sublime_text/

### 1.安装C/C++的编译环境和gtk libgtk2.0-dev

    sudo apt-get install build-essential
    sudo apt-get install libgtk2.0-dev

### 2.编译共享内库

    gcc -shared -o libsublime-imfix.so sublime-imfix.c  `pkg-config --libs --cflags gtk+-2.0` -fPIC

#### sublime-imfix.c代码由OSChina Wuu[原帖](http://my.oschina.net/wugaoxing/blog/121281)给出，搬运如下：

```c
/*
sublime-imfix.c
Use LD_PRELOAD to interpose some function to fix sublime input method support for linux.
By Cjacker Huang <jianzhong.huang at i-soft.com.cn>
gcc -shared -o libsublime-imfix.so sublime_imfix.c  `pkg-config --libs --cflags gtk+-2.0` -fPIC
LD_PRELOAD=./libsublime-imfix.so sublime_text
*/
#include <gtk/gtk.h>
#include <gdk/gdkx.h>
typedef GdkSegment GdkRegionBox;

struct _GdkRegion
{
  long size;
  long numRects;
  GdkRegionBox *rects;
  GdkRegionBox extents;
};

GtkIMContext *local_context;

void
gdk_region_get_clipbox (const GdkRegion *region,
            GdkRectangle    *rectangle)
{
  g_return_if_fail (region != NULL);
  g_return_if_fail (rectangle != NULL);

  rectangle->x = region->extents.x1;
  rectangle->y = region->extents.y1;
  rectangle->width = region->extents.x2 - region->extents.x1;
  rectangle->height = region->extents.y2 - region->extents.y1;
  GdkRectangle rect;
  rect.x = rectangle->x;
  rect.y = rectangle->y;
  rect.width = 0;
  rect.height = rectangle->height; 
  //The caret width is 2; 
  //Maybe sometimes we will make a mistake, but for most of the time, it should be the caret.
  if(rectangle->width == 2 && GTK_IS_IM_CONTEXT(local_context)) {
        gtk_im_context_set_cursor_location(local_context, rectangle);
  }
}

//this is needed, for example, if you input something in file dialog and return back the edit area
//context will lost, so here we set it again.

static GdkFilterReturn event_filter (GdkXEvent *xevent, GdkEvent *event, gpointer im_context)
{
    XEvent *xev = (XEvent *)xevent;
    if(xev->type == KeyRelease && GTK_IS_IM_CONTEXT(im_context)) {
       GdkWindow * win = g_object_get_data(G_OBJECT(im_context),"window");
       if(GDK_IS_WINDOW(win))
         gtk_im_context_set_client_window(im_context, win);
    }
    return GDK_FILTER_CONTINUE;
}

void gtk_im_context_set_client_window (GtkIMContext *context,
          GdkWindow    *window)
{
  GtkIMContextClass *klass;
  g_return_if_fail (GTK_IS_IM_CONTEXT (context));
  klass = GTK_IM_CONTEXT_GET_CLASS (context);
  if (klass->set_client_window)
    klass->set_client_window (context, window);

  if(!GDK_IS_WINDOW (window))
    return;
  g_object_set_data(G_OBJECT(context),"window",window);
  int width = gdk_window_get_width(window);
  int height = gdk_window_get_height(window);
  if(width != 0 && height !=0) {
    gtk_im_context_focus_in(context);
    local_context = context;
  }
  gdk_window_add_filter (window, event_filter, context); 
}
```

### 3.将libsublime-imfix.so移动到/usr/lib/

    sudo mv libsublime-imfix.so /usr/lib/

### 4.启动 Sublime Text 3

    LD_PRELOAD=/usr/lib/libsublime-imfix.so subl

即可运行Fcitx的搜狗拼音输入法了。

为了避免每次都要在终端里面使用命令启动sublime text 3，还要通过修改subl以及sublime-text.desktop达到统一的启动效果。

### 1.打开sublime-text-3

    sudo vim /usr/bin/subl

### 2.修改subl为以下内容

    #!/bin/bash

    LD_PRELOAD=/usr/lib/libsublime-imfix.so /opt/sublime_text/sublime_text "$@"

这样就能使用subl等命令启动

### 3.打开sublime-text.desktop

    sudo vim /usr/share/applications/sublime-text.desktop

### 4.修改sublime-text.desktop

将

    Exec=/usr/bin/sublime-text-3 %F

修改为

    Exec=bash -c 'LD_PRELOAD=/usr/lib/libsublime-imfix.so /usr/bin/sublime-text-3' %F

将

    Exec=/usr/bin/sublime-text-3 --new-window

修改为

    Exec=bash -c 'LD_PRELOAD=/usr/lib/libsublime-imfix.so /usr/bin/sublime-text-3' --new-window

感谢Wuu[原帖](http://my.oschina.net/wugaoxing/blog/121281)给出解决方案。

