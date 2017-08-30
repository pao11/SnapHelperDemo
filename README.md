## 概述

RecyclerView在24.2.0版本中新增了SnapHelper这个辅助类，用于辅助RecyclerView在滚动结束时将Item对齐到某个位置。特别是列表横向滑动时，很多时候不会让列表滑到任意位置，而是会有一定的规则限制，这时候就可以通过SnapHelper来定义对齐规则了。

SnapHelper是一个抽象类，官方提供了一个LinearSnapHelper的子类，可以让RecyclerView滚动停止时相应的Item停留中间位置。25.1.0版本中官方又提供了一个PagerSnapHelper的子类，可以使RecyclerView像ViewPager一样的效果，一次只能滑一页，而且居中显示。

这两个子类使用方式也很简单，只需要创建对象之后调用attachToRecyclerView()附着到对应的RecyclerView对象上就可以了。

```
new LinearSnapHelper().attachToRecyclerView(mRecyclerView);
//或者
new PagerSnapHelper().attachToRecyclerView(mRecyclerView);

```
## 自定义SnapHelper
    一个类似Gallery的横向列表滑动控件，滚动后的ItemView是对齐RecyclerView的左边缘位置。
#### 创建一个GallerySnapHelper继承SnapHelper实现它的三个抽象方法：

    1、calculateDistanceToFinalSnap（）：计算SnapView当前位置与目标位置的距离
    2、findSnapView（）：找到当前时刻的SnapView。
    3、findTargetSnapPosition()： 在触发fling时找到targetSnapPosition。
    

## Copyright Notice
```
Copyright (C) 2017 pao11

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.