## RecyclerView的滚动事件有 

recyclerView.scrollToPosition(position) 没有平滑滚动效果 (**不再屏幕上** (包括初始化时候没有绘制到屏幕上)才会生效，如果已经显示在屏幕上就无效)

recyclerView.smoothScrollToPosition(position) 平滑滚动 (**不再屏幕上** (包括初始化时候没有绘制到屏幕上)才会生效，如果已经显示在屏幕上就无效)

((GridLayoutManager)recyclerView.getLayoutManager()).scrollToPositionWithOffset(position, 0)没有平滑滚动效果 指定position滚动到指定（0）的位置。