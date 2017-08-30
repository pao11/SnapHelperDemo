## 概述

RecyclerView在24.2.0版本中新增了SnapHelper这个辅助类，用于辅助RecyclerView在滚动结束时将Item对齐到某个位置。特别是列表横向滑动时，很多时候不会让列表滑到任意位置，而是会有一定的规则限制，这时候就可以通过SnapHelper来定义对齐规则了。

SnapHelper是一个抽象类，官方提供了一个LinearSnapHelper的子类，可以让RecyclerView滚动停止时相应的Item停留中间位置。25.1.0版本中官方又提供了一个PagerSnapHelper的子类，可以使RecyclerView像ViewPager一样的效果，一次只能滑一页，而且居中显示。

这两个子类使用方式也很简单，只需要创建对象之后调用attachToRecyclerView()附着到对应的RecyclerView对象上就可以了。

```
new LinearSnapHelper().attachToRecyclerView(mRecyclerView);
//或者
new PagerSnapHelper().attachToRecyclerView(mRecyclerView);

```
#### attachToRecyclerView()

现在来看attachToRecyclerView()这个方法，SnapHelper正是通过该方法附着到RecyclerView上，从而实现辅助RecyclerView滚动对齐操作。源码如下：
```
   public void attachToRecyclerView(@Nullable RecyclerView recyclerView)
            throws IllegalStateException {
      //如果SnapHelper之前已经附着到此RecyclerView上，不用进行任何操作
        if (mRecyclerView == recyclerView) {
            return;
        }
      //如果SnapHelper之前附着的RecyclerView和现在的不一致，清理掉之前RecyclerView的回调
        if (mRecyclerView != null) {
            destroyCallbacks();
        }
      //更新RecyclerView对象引用
        mRecyclerView = recyclerView;
        if (mRecyclerView != null) {
          //设置当前RecyclerView对象的回调
            setupCallbacks();
          //创建一个Scroller对象，用于辅助计算fling的总距离，后面会涉及到
            mGravityScroller = new Scroller(mRecyclerView.getContext(),
                    new DecelerateInterpolator());
          //调用snapToTargetExistingView()方法以实现对SnapView的对齐滚动处理
            snapToTargetExistingView();
        }
    }
```
可以看到，在attachToRecyclerView()方法中会清掉SnapHelper之前保存的RecyclerView对象的回调(如果有的话)，对新设置进来的RecyclerView对象设置回调,然后初始化一个Scroller对象,最后调用snapToTargetExistingView()方法对SnapView进行对齐调整。
#### snapToTargetExistingView()

该方法的作用是对SnapView进行滚动调整，以使得SnapView达到对齐效果。源码如下：

    void snapToTargetExistingView() {
        if (mRecyclerView == null) {
            return;
        }
        LayoutManager layoutManager = mRecyclerView.getLayoutManager();
        if (layoutManager == null) {
            return;
        }
      //找出SnapView
        View snapView = findSnapView(layoutManager);
        if (snapView == null) {
            return;
        }
      //计算出SnapView需要滚动的距离
        int[] snapDistance = calculateDistanceToFinalSnap(layoutManager, snapView);
      //如果需要滚动的距离不是为0，就调用smoothScrollBy（）使RecyclerView滚动相应的距离
        if (snapDistance[0] != 0 || snapDistance[1] != 0) {
            mRecyclerView.smoothScrollBy(snapDistance[0], snapDistance[1]);
        }
    }

可以看到，snapToTargetExistingView()方法就是先找到SnapView，然后计算SnapView当前坐标到目的坐标之间的距离，然后调用RecyclerView.smoothScrollBy()方法实现对RecyclerView内容的平滑滚动，从而将SnapView移到目标位置，达到对齐效果。RecyclerView.smoothScrollBy()这个方法的实现原理这里就不展开了 ，它的作用就是根据参数平滑滚动RecyclerView的中的ItemView相应的距离。
#### setupCallbacks()和destroyCallbacks()

再看下SnapHelper对RecyclerView设置了哪些回调:

    private void setupCallbacks() throws IllegalStateException {
        if (mRecyclerView.getOnFlingListener() != null) {
            throw new IllegalStateException("An instance of OnFlingListener already set.");
        }
        mRecyclerView.addOnScrollListener(mScrollListener);
        mRecyclerView.setOnFlingListener(this);
    }

    private void destroyCallbacks() {
        mRecyclerView.removeOnScrollListener(mScrollListener);
        mRecyclerView.setOnFlingListener(null);
    }

可以看出RecyclerView设置的回调有两个：一个是OnScrollListener对象mScrollListener.还有一个是OnFlingListener对象。由于SnapHelper实现了OnFlingListener接口,所以这个对象就是SnapHelper自身了.

先看下mScrollListener这个变量在怎样实现的.

    private final RecyclerView.OnScrollListener mScrollListener =
            new RecyclerView.OnScrollListener() {
                boolean mScrolled = false;
                @Override
                public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
                    super.onScrollStateChanged(recyclerView, newState);
                  //mScrolled为true表示之前进行过滚动.
                  //newState为SCROLL_STATE_IDLE状态表示滚动结束停下来
                    if (newState == RecyclerView.SCROLL_STATE_IDLE && mScrolled) {
                        mScrolled = false;
                        snapToTargetExistingView();
                    }
                }

                @Override
                public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
                    if (dx != 0 || dy != 0) {
                        mScrolled = true;
                    }
                }
            };

该滚动监听器的实现很简单,只是在正常滚动停止的时候调用了snapToTargetExistingView()方法对targetView进行滚动调整，以确保停止的位置是在对应的坐标上，这就是RecyclerView添加该OnScrollListener的目的。

除了OnScrollListener这个监听器，还对RecyclerView还设置了OnFlingListener这个监听器，而这个监听器就是SnapHelper自身。因为SnapHelper实现了RecyclerView.OnFlingListener接口。我们先来看看RecyclerView.OnFlingListener这个接口。

    public static abstract class OnFlingListener {
            /**
         * Override this to handle a fling given the velocities in both x and y directions.
         * Note that this method will only be called if the associated {@link LayoutManager}
         * supports scrolling and the fling is not handled by nested scrolls first.
         *
         * @param velocityX the fling velocity on the X axis
         * @param velocityY the fling velocity on the Y axis
         *
         * @return true if the fling washandled, false otherwise.
         */
        public abstract boolean onFling(int velocityX, int velocityY);
    }

这个接口中就只有一个onFling()方法，该方法会在RecyclerView开始做fling操作时被调用。我们来看看SnapHelper怎么实现onFling()方法：

    @Override
    public boolean onFling(int velocityX, int velocityY) {
        LayoutManager layoutManager = mRecyclerView.getLayoutManager();
        if (layoutManager == null) {
            return false;
        }
        RecyclerView.Adapter adapter = mRecyclerView.getAdapter();
        if (adapter == null) {
            return false;
        }
      //获取RecyclerView要进行fling操作需要的最小速率，
      //只有超过该速率，ItemView才会有足够的动力在手指离开屏幕时继续滚动下去
        int minFlingVelocity = mRecyclerView.getMinFlingVelocity();
      //这里会调用snapFromFling()这个方法，就是通过该方法实现平滑滚动并使得在滚动停止时itemView对齐到目的坐标位置
        return (Math.abs(velocityY) > minFlingVelocity || Math.abs(velocityX) > minFlingVelocity)
                && snapFromFling(layoutManager, velocityX, velocityY);
    }

注释解释得很清楚。看下snapFromFling()怎么操作的：

    private boolean snapFromFling(@NonNull LayoutManager layoutManager, int velocityX,
            int velocityY) {
      //layoutManager必须实现ScrollVectorProvider接口才能继续往下操作
        if (!(layoutManager instanceof ScrollVectorProvider)) {
            return false;
        }
        
      //创建SmoothScroller对象，这个东西是一个平滑滚动器，用于对ItemView进行平滑滚动操作
        RecyclerView.SmoothScroller smoothScroller = createSnapScroller(layoutManager);
        if (smoothScroller == null) {
            return false;
        }
        
      //通过findTargetSnapPosition()方法，以layoutManager和速率作为参数，找到targetSnapPosition
        int targetPosition = findTargetSnapPosition(layoutManager, velocityX, velocityY);
        if (targetPosition == RecyclerView.NO_POSITION) {
            return false;
        }
        //通过setTargetPosition()方法设置滚动器的滚动目标位置
        smoothScroller.setTargetPosition(targetPosition);
        //利用layoutManager启动平滑滚动器，开始滚动到目标位置
        layoutManager.startSmoothScroll(smoothScroller);
        return true;
    }

可以看到，snapFromFling()方法会先判断layoutManager是否实现了ScrollVectorProvider接口，如果没有实现该接口就不允许通过该方法做滚动操作。那为啥一定要实现该接口呢？待会再来解释。接下来就去创建平滑滚动器SmoothScroller的一个实例，layoutManager可以通过该平滑滚动器来进行滚动操作。SmoothScroller需要设置一个滚动的目标位置，我们将通过findTargetSnapPosition()方法来计算得到的targetSnapPosition给它，告诉滚动器要滚到这个位置，然后就启动SmoothScroller进行滚动操作。

但是这里有一点需要注意一下，默认情况下通过setTargetPosition()方法设置的SmoothScroller只能将对应位置的ItemView滚动到与RecyclerView的边界对齐，那怎么实现将该ItemView滚动到我们需要对齐的目标位置呢？就得对SmoothScroller进行一下处理了。

看下平滑滚动器RecyclerView.SmoothScroller，这个东西是通过createSnapScroller()方法创建得到的：

    @Nullable
    protected LinearSmoothScroller createSnapScroller(LayoutManager layoutManager) {
      //同样，这里也是先判断layoutManager是否实现了ScrollVectorProvider这个接口，
      //没有实现该接口就不创建SmoothScroller
        if (!(layoutManager instanceof ScrollVectorProvider)) {
            return null;
        }
      //这里创建一个LinearSmoothScroller对象，然后返回给调用函数，
      //也就是说，最终创建出来的平滑滚动器就是这个LinearSmoothScroller
        return new LinearSmoothScroller(mRecyclerView.getContext()) {
          //该方法会在targetSnapView被layout出来的时候调用。
          //这个方法有三个参数：
          //第一个参数targetView，就是本文所讲的targetSnapView
          //第二个参数RecyclerView.State这里没用到，先不管它
          //第三个参数Action，这个是什么东西呢？它是SmoothScroller的一个静态内部类,
          //保存着SmoothScroller在平滑滚动过程中一些信息，比如滚动时间，滚动距离，差值器等
            @Override
            protected void onTargetFound(View targetView, RecyclerView.State state, Action action) {
             //calculateDistanceToFinalSnap（）方法上面解释过，
             //得到targetSnapView当前坐标到目的坐标之间的距离
                int[] snapDistances = calculateDistanceToFinalSnap(mRecyclerView.getLayoutManager(),
                        targetView);
                final int dx = snapDistances[0];
                final int dy = snapDistances[1];
              //通过calculateTimeForDeceleration（）方法得到做减速滚动所需的时间
                final int time = calculateTimeForDeceleration(Math.max(Math.abs(dx), Math.abs(dy)));
                if (time > 0) {
                  //调用Action的update()方法，更新SmoothScroller的滚动速率，使其减速滚动到停止
                  //这里的这样做的效果是，此SmoothScroller用time这么长的时间以mDecelerateInterpolator这个差值器的滚动变化率滚动dx或者dy这么长的距离
                    action.update(dx, dy, time, mDecelerateInterpolator);
                }
            }

          //该方法是计算滚动速率的，返回值代表滚动速率，该值会影响刚刚上面提到的
          //calculateTimeForDeceleration（）的方法的返回返回值，
          //MILLISECONDS_PER_INCH的值是100，也就是说该方法的返回值代表着每dpi的距离要滚动100毫秒
            @Override
            protected float calculateSpeedPerPixel(DisplayMetrics displayMetrics) {
                return MILLISECONDS_PER_INCH / displayMetrics.densityDpi;
            }
        };
    }

通过以上的分析可以看到，createSnapScroller()创建的是一个LinearSmoothScroller，并且在创建该LinearSmoothScroller的时候主要考虑两个方面：

    第一个是滚动速率，由calculateSpeedPerPixel()方法决定；
    第二个是在滚动过程中，targetView即将要进入到视野时，将匀速滚动变换为减速滚动，然后一直滚动目的坐标位置，使滚动效果更真实，这是由onTargetFound()方法决定。

刚刚不是留了一个疑问么？就是正常模式下SmoothScroller通过setTargetPosition()方法设置的ItemView只能滚动到与RecyclerView边缘对齐，而解决这个局限的处理方式就是在SmoothScroller的onTargetFound()方法中了。onTargetFound()方法会在SmoothScroller滚动过程中，targetSnapView被layout出来时调用。而这个时候利用calculateDistanceToFinalSnap()方法得到targetSnapView当前坐标与目的坐标之间的距离，然后通过Action.update()方法改变当前SmoothScroller的状态，让SmoothScroller根据新的滚动距离、新的滚动时间、新的滚动差值器来滚动，这样既能将targetSnapView滚动到目的坐标位置，又能实现减速滚动，使得滚动效果更真实。

从图中可以看到，很多时候targetSnapView被layout的时候（onTargetFound()方法被调用）并不是紧挨着界面上的Item，而是会有一定的提前，这是由于RecyclerView为了优化性能，提高流畅度，在滑动滚动的时候会有一个预加载的过程，提前将Item给layout出来了，这个知识点涉及到的内容很多，这里做个理解就可以了，不详细细展开了，以后有时间会专门讲下RecyclerView的相关原理机制。

到了这里，整理一下前面的思路：SnapHelper实现了OnFlingListener这个接口，该接口中的onFling()方法会在RecyclerView触发Fling操作时调用。在onFling()方法中判断当前方向上的速率是否足够做滚动操作，如果速率足够大就调用snapFromFling()方法实现滚动相关的逻辑。在snapFromFling()方法中会创建一个SmoothScroller，并且根据速率计算出滚动停止时的位置，将该位置设置给SmoothScroller并启动滚动。而滚动的操作都是由SmoothScroller全权负责，它可以控制Item的滚动速度（刚开始是匀速），并且在滚动到targetSnapView被layout时变换滚动速度（转换成减速），以让滚动效果更加真实。

所以，SnapHelper辅助RecyclerView实现滚动对齐就是通过给RecyclerView设置OnScrollerListenerh和OnFlingListener这两个监听器实现的。

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