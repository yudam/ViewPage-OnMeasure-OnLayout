# ViewPage-OnMeasure-OnLayout
ViewPage源码阅读：绘制篇  
今天主要来讲ViePage的绘制篇章，因为onDraw没有什么操作，因此只讲解onMeasure和onLayout，因为是对源码的理解，就需要对View的绘制流程有一些了解，这里不再单独介绍，接下来就看一看ViewPage的onMeasure方法
    
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

        //设置ViewPage的尺寸大小
        setMeasuredDimension(getDefaultSize(0, widthMeasureSpec),
                getDefaultSize(0, heightMeasureSpec));



        final int measuredWidth = getMeasuredWidth();
        final int maxGutterSize = measuredWidth / 10;
        mGutterSize = Math.min(maxGutterSize, mDefaultGutterSize);

        // Children 的可用宽高大小
        int childWidthSize = measuredWidth - getPaddingLeft() - getPaddingRight();
        int childHeightSize = getMeasuredHeight() - getPaddingTop() - getPaddingBottom();

        int size = getChildCount();
        //处理非适配器View
        for (int i = 0; i < size; ++i) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();

                //isDecor=true表明该view属于ViewPage,不是适配器提供的视图
                if (lp != null && lp.isDecor) {

                    //获取DecordView在竖直，水平方向的Gravity
                    final int hgrav = lp.gravity & Gravity.HORIZONTAL_GRAVITY_MASK;
                    final int vgrav = lp.gravity & Gravity.VERTICAL_GRAVITY_MASK;

                    //默认AT_MOST(最大不能超过父View大小)
                    int widthMode = MeasureSpec.AT_MOST;
                    int heightMode = MeasureSpec.AT_MOST;

                    //记录DecordView是在竖直还是水平占用空间
                    boolean consumeVertical = vgrav == Gravity.TOP || vgrav == Gravity.BOTTOM;
                    boolean consumeHorizontal = hgrav == Gravity.LEFT || hgrav == Gravity.RIGHT;

                    if (consumeVertical) {

                        //消费竖直方向，则水平MODE为EXACTLY
                        widthMode = MeasureSpec.EXACTLY;
                    } else if (consumeHorizontal) {
                        //消费水平方向，则竖直MODE为EXACTLY
                        heightMode = MeasureSpec.EXACTLY;
                    }

                    int widthSize = childWidthSize;
                    int heightSize = childHeightSize;
                    //设置width的size，mode
                    if (lp.width != LayoutParams.WRAP_CONTENT) {
                        widthMode = MeasureSpec.EXACTLY;
                        if (lp.width != LayoutParams.MATCH_PARENT) {
                            widthSize = lp.width;
                        }
                    }
                    //设置width的size，mode
                    if (lp.height != LayoutParams.WRAP_CONTENT) {
                        heightMode = MeasureSpec.EXACTLY;
                        if (lp.height != LayoutParams.MATCH_PARENT) {
                            heightSize = lp.height;
                        }
                    }
                    final int widthSpec = MeasureSpec.makeMeasureSpec(widthSize, widthMode);
                    final int heightSpec = MeasureSpec.makeMeasureSpec(heightSize, heightMode);
                    child.measure(widthSpec, heightSpec);

                    //减去decordView占据的竖直或者水平方向空间
                    if (consumeVertical) {
                        childHeightSize -= child.getMeasuredHeight();
                    } else if (consumeHorizontal) {
                        childWidthSize -= child.getMeasuredWidth();
                    }
                }
            }
        }

        //adapter中View实际可用空间，size，mode
        mChildWidthMeasureSpec = MeasureSpec.makeMeasureSpec(childWidthSize, MeasureSpec.EXACTLY);
        mChildHeightMeasureSpec = MeasureSpec.makeMeasureSpec(childHeightSize, MeasureSpec.EXACTLY);

        // Make sure we have created all fragments that we need to have shown.
        mInLayout = true;
        //调用该方法是为了计算偏移量
        populate();
        mInLayout = false;

        // Page views next.
        size = getChildCount();
        for (int i = 0; i < size; ++i) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                if (DEBUG) {
                    Log.v(TAG, "Measuring #" + i + " " + child + ": " + mChildWidthMeasureSpec);
                }

                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                if (lp == null || !lp.isDecor) {
                    final int widthSpec = MeasureSpec.makeMeasureSpec(
                            (int) (childWidthSize * lp.widthFactor), MeasureSpec.EXACTLY);
                    child.measure(widthSpec, mChildHeightMeasureSpec);
                }
            }
        }
    }
    
代码中主要方法基本上都加入了注释，这里可以看到，首先就是测量ViewPage的可用宽高，然后逐一测量child，这里注意的是ViewPage市先测量decordView，然后测量非decordViewView，decordView这里可能有些同学不太理解，它属于非Adapter中添加的View，需要实现decordView特殊接口，在之前的一章中我们讲解了setAdapter()，populate()方法，其中除了缓存指定页面外，还对ViewPage的child进行了排序，详细可见
    
    private void sortChildDrawingOrder() {
        if (mDrawingOrder != DRAW_ORDER_DEFAULT) {
            if (mDrawingOrderedChildren == null) {
                mDrawingOrderedChildren = new ArrayList<View>();
            } else {
                mDrawingOrderedChildren.clear();
            }
            final int childCount = getChildCount();
            for (int i = 0; i < childCount; i++) {
                final View child = getChildAt(i);
                mDrawingOrderedChildren.add(child);
            }
            Collections.sort(mDrawingOrderedChildren, sPositionComparator);
        }
    }
    
先绘制DecordView，通过以下方法获取decordView在竖直和水平方向的Gravity：
    
     //获取DecordView在竖直，水平方向的Gravity
                    final int hgrav = lp.gravity & Gravity.HORIZONTAL_GRAVITY_MASK;
                    final int vgrav = lp.gravity & Gravity.VERTICAL_GRAVITY_MASK;
                    
可以看到通过该方法获取得值是为了计算水平和竖直方向的MODE，然后测量decordView，接下来看看绘制onLayout：
    
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {

        final int count = getChildCount();
        int width = r - l;
        int height = b - t;
        int paddingLeft = getPaddingLeft();
        int paddingTop = getPaddingTop();
        int paddingRight = getPaddingRight();
        int paddingBottom = getPaddingBottom();
        final int scrollX = getScrollX();

        int decorCount = 0;

        //先绘制DecordView位置(已经排好序了，先绘制Decord)
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                //左边和顶部边距初始化
                int childLeft = 0;
                int childTop = 0;
                if (lp.isDecor) {
                    final int hgrav = lp.gravity & Gravity.HORIZONTAL_GRAVITY_MASK;
                    final int vgrav = lp.gravity & Gravity.VERTICAL_GRAVITY_MASK;
                    switch (hgrav) {
                        default://默认绘制在最左边会覆盖
                            childLeft = paddingLeft;
                            break;
                        case Gravity.LEFT://绘制在左边，每次要加上前一次的view宽度
                            childLeft = paddingLeft;
                            paddingLeft += child.getMeasuredWidth();
                            break;
                        case Gravity.CENTER_HORIZONTAL://水平居中
                            childLeft = Math.max((width - child.getMeasuredWidth()) / 2,
                                    paddingLeft);
                            break;
                        case Gravity.RIGHT:
                            //左边距=viewpage宽度-右边距-DecordView的宽度
                            childLeft = width - paddingRight - child.getMeasuredWidth();
                            paddingRight += child.getMeasuredWidth();
                            break;
                    }

                    //同水平绘制类似
                    switch (vgrav) {
                        default:
                            childTop = paddingTop;
                            break;
                        case Gravity.TOP:
                            childTop = paddingTop;
                            paddingTop += child.getMeasuredHeight();
                            break;
                        case Gravity.CENTER_VERTICAL:
                            childTop = Math.max((height - child.getMeasuredHeight()) / 2,
                                    paddingTop);
                            break;
                        case Gravity.BOTTOM:
                            childTop = height - paddingBottom - child.getMeasuredHeight();
                            paddingBottom += child.getMeasuredHeight();
                            break;
                    }
                    //还需要+ViewPage滑动的距离
                    childLeft += scrollX;
                    //布局DecordView
                    child.layout(childLeft, childTop,
                            childLeft + child.getMeasuredWidth(),
                            childTop + child.getMeasuredHeight());
                    decorCount++;
                }
            }
        }

        //适配器中View实际可用宽度
        final int childWidth = width - paddingLeft - paddingRight;
        //
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                ItemInfo ii;
                //当前页面属于适配器且不为空
                if (!lp.isDecor && (ii = infoForChild(child)) != null) {
                    //页卡的左偏移量
                    int loff = (int) (childWidth * ii.offset);
                    //计算世界页卡偏移量
                    int childLeft = paddingLeft + loff;
                    int childTop = paddingTop;
                    if (lp.needsMeasure) {
                          //标记已经测量过了
                        lp.needsMeasure = false;
                        //测量child实际大小
                        final int widthSpec = MeasureSpec.makeMeasureSpec(
                                (int) (childWidth * lp.widthFactor),
                                MeasureSpec.EXACTLY);
                        final int heightSpec = MeasureSpec.makeMeasureSpec(
                                (int) (height - paddingTop - paddingBottom),
                                MeasureSpec.EXACTLY);
                        child.measure(widthSpec, heightSpec);
                    }
                   //绘制child
                    child.layout(childLeft, childTop,
                            childLeft + child.getMeasuredWidth(),
                            childTop + child.getMeasuredHeight());
                }
            }
        }
        //保存部分局部变量
        mTopPageBounds = paddingTop;
        mBottomPageBounds = height - paddingBottom;
        mDecorChildCount = decorCount;

        //第一次布局，默认滑动到第一页
        if (mFirstLayout) {
            scrollToItem(mCurItem, false, 0, false);
        }
        mFirstLayout = false;
    }

可以看到依然根据测量，首先绘制DecordView，会致使需要注意，因为DecordView可以位于屏幕任意边界，所以需要根据
    
    final int hgrav = lp.gravity & Gravity.HORIZONTAL_GRAVITY_MASK;
    final int vgrav = lp.gravity & Gravity.VERTICAL_GRAVITY_MASK;
                    
分别进行计算出left，top，然后根据left+getMeasureWidth,top_getMeasureHeight计算出四个顶点，绘制View。接下来因为适配器中View存在偏移量offset和widthFactor等因素，需要再次测量实际大小
    
    if (lp.needsMeasure) {
                          //标记已经测量过了
                        lp.needsMeasure = false;
                        //测量child实际大小
                        final int widthSpec = MeasureSpec.makeMeasureSpec(
                                (int) (childWidth * lp.widthFactor),
                                MeasureSpec.EXACTLY);
                        final int heightSpec = MeasureSpec.makeMeasureSpec(
                                (int) (height - paddingTop - paddingBottom),
                                MeasureSpec.EXACTLY);
                        child.measure(widthSpec, heightSpec);
                    }
                   //绘制child
                    child.layout(childLeft, childTop,
                            childLeft + child.getMeasuredWidth(),
                            childTop + child.getMeasuredHeight());
                            
可以看到绘制的时候要计算 childLeft = paddingLeft + loff和int childTop = paddingTop，之测量一次。
