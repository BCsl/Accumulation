## 两种类型
__"stateXXX|adjustXXX"__

## stateXXX
控制当活动(Activity)成为用户关注的焦点时候，软键盘的状态是否它是隐藏或显示

| value | Description |
|-------|-------------|
|stateUnspecified| |
|stateUnchanged| 保持之前的状态|
|stateHidden|当用户明确导航到某个`Activity`而不是离开当前`Activity`而返回之前的某个`Activity`的时候，取得焦点的`Activity`就会隐藏软键盘 |
|stateAlwaysHidden|只要当前`Activity`焦点，就一定隐藏软键盘 |
|stateVisible|软键盘正常是可见的(当用户导航到Activity主窗口时)|
|stateAlwaysVisible|当用户明确导航到某个`Activity`而不是离开当前`Activity`而返回之前的某个`Activity`的时候，取得焦点的`Activity`就会显示软键盘|

## adjustXXX
活动的主窗口调整——是否减少活动主窗口大小以便腾出空间放软键盘或是否当活动窗口的部分被软键盘覆盖时它的内容的当前焦点是可见的

| value | Description |
|-------|-------------|
|adjustUnspecified| |
|adjustResize|该`Activity`主窗口会调整屏幕的大小以便留出软键盘的空间，对于非滚动布局来说，就是噩梦，呵呵|
|adjustPan| 该Activity主窗口并不调整屏幕的大小以便留出软键盘的空间。相反，当前窗口的内容将`自动`移动以便当前焦点不被键盘覆盖，使用户能总是看到输入内容的部分。相对`adjustResize`来说，通常取得的效果没那么好，因为只保证当前焦点不被覆盖，用户可能需要关闭软键盘以便和与被覆盖内容的交互|
