一、帧数和内存显示
![[Pasted image 20260817150515.png]]

// 自定义UI界面
// 1、获取当前硬件整个屏幕
lv_obj_t *src = lv_screen_active();

// 2、创建一个新的屏幕 => 会有默认大小和样式
lv_obj_t *obj = lv_obj_create(src);

// 3、创建一个按钮
lv_obj_t *btn = lv_btn_create(obj);