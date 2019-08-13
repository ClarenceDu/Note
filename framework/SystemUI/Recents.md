# Recents #
该模块用于展示最近运行的应用列表。
## 参考 ##
https://www.jianshu.com/p/2b7bd6ec76db
## 运行流程 ##
### Recents模块的启动和主要逻辑的分析 ###
用户按KEYCODE_APP_SWITCH按键后系统会走PhoneWindowManager.java的interceptKeyBeforeDispatching方法。

    @Override
    public long interceptKeyBeforeDispatching(WindowState win, KeyEvent event, int policyFlags) {
    ...
       } else if (keyCode == KeyEvent.KEYCODE_APP_SWITCH) {
   	 		if (!keyguardOn) {
    			  if (down && repeatCount == 0) {
						//按下按键时
    					preloadRecentApps();
   				  } else if (!down) {
                        //松下时
    					toggleRecentApps();
   			      }
    	    }
       //返回-1表示该按键事件不会继续往下传输了
    	return -1;
   	 }
    ...
    }

分析preloadRecentApps()和toggleRecentApps()方法。

    private void preloadRecentApps() {
        mPreloadedRecentApps = true;
        StatusBarManagerInternal statusbar = getStatusBarManagerInternal();
        if (statusbar != null) {
            statusbar.preloadRecentApps();
        }
    }

    private void toggleRecentApps() {
        mPreloadedRecentApps = false; // preloading no longer needs to be canceled
        StatusBarManagerInternal statusbar = getStatusBarManagerInternal();
        if (statusbar != null) {
            statusbar.toggleRecentApps();
        }
    }

	
statusbar本质是SystemUI中的CommandQueue.java这个对象，跟踪源码最终先后调用的是Recents.java中preloadRecentApps()和
toggleRecentApps()，首先分析preloadRecentApps方法，

    /**
     * Preloads info for the Recents activity.
     *	加载最近任务元数据，同时加载应用图标，但是不加载缩略图
     */
    @Override
    public void preloadRecentApps() {
			...     
			//mImpl是RecentsImpl对象，使Recents.java类的具体功能实现
            mImpl.preloadRecents();
			...
    }

RecentsImpl：

    public void preloadRecents() {
        // Preload only the raw task list into a new load plan (which will be consumed by the
        // RecentsActivity) only if there is a task to animate to.  Post this to ensure that we
        // don't block the touch feedback on the nav bar button which triggers this.
        //此处采用java 8中的lambda表达式
        mHandler.post(() -> {
            SystemServicesProxy ssp = Recents.getSystemServices();
            MutableBoolean isHomeStackVisible = new MutableBoolean(true);
            if (!ssp.isRecentsActivityVisible(isHomeStackVisible)) {//当前Recents界面未显示才走此分支
            	//获取正在运行任务
                ActivityManager.RunningTaskInfo runningTask = ssp.getRunningTask();
                if (runningTask == null) {
                    return;
                }

                RecentsTaskLoader loader = Recents.getTaskLoader();
                sInstanceLoadPlan = loader.createLoadPlan(mContext);
				//完成加载任务，加载的结果存储在sInstanceLoadPlan中
                loader.preloadTasks(sInstanceLoadPlan, runningTask.id, !isHomeStackVisible.value);
				//TaskStack ：任务堆栈包含多个任务的列表。
                TaskStack stack = sInstanceLoadPlan.getTaskStack();
                if (stack.getTaskCount() > 0) {
                    // Only preload the icon (but not the thumbnail since it may not have been taken
                    // for the pausing activity)
                    //加载图标，不加载缩略图。
                    preloadIcon(runningTask.id);

                    // At this point, we don't know anything about the stack state.  So only
                    // calculate the dimensions of the thumbnail that we need for the transition
                    // into Recents, but do not draw it until we construct the activity options when
                    // we start Recents
                    updateHeaderBarLayout(stack, null /* window rect override*/);
                }
            }
        });
    }	

执行toggleRecentApps后开始执行toggleRecentApps方法。
Recents：

    /**
     * Toggles the Recents activity.
     */
    @Override
    public void toggleRecentApps() {
		...
            mImpl.toggleRecents(growTarget);
        ...
    }

RecentsImpl：

    public void toggleRecents(int growTarget) {
        // Skip this toggle if we are already waiting to trigger recents via alt-tab
        if (mFastAltTabTrigger.isDozing()) {
            return;
        }

        mDraggingInRecents = false;
        mLaunchedWhileDocking = false;
        mTriggeredFromAltTab = false;

        try {
            SystemServicesProxy ssp = Recents.getSystemServices();
			//此变量用于判断Recents是从app中启动还是从home界面启动的。若从app启动则按返回键或KEYCODE_APP_SWITCH
			//退出Recents界面时界面回到启动Recents时的app，否则就回到Home页
            MutableBoolean isHomeStackVisible = new MutableBoolean(true);
            long elapsedTime = SystemClock.elapsedRealtime() - mLastToggleTime;

            if (ssp.isRecentsActivityVisible(isHomeStackVisible)) {//最近任务界面已启动
                RecentsDebugFlags debugFlags = Recents.getDebugFlags();
                RecentsConfiguration config = Recents.getConfiguration();
                RecentsActivityLaunchState launchState = config.getLaunchState();
                if (!launchState.launchedWithAltTab) {
                    // Has the user tapped quickly?
                    boolean isQuickTap = elapsedTime < ViewConfiguration.getDoubleTapTimeout();
                    if (Recents.getConfiguration().isGridEnabled) {
                        if (isQuickTap) {
                            EventBus.getDefault().post(new LaunchNextTaskRequestEvent());
                        } else {
                            EventBus.getDefault().post(new LaunchMostRecentTaskRequestEvent());
                        }
                    } else {
                        if (!debugFlags.isPagingEnabled() || isQuickTap) {
                            // Launch the next focused task 启动下一个有焦点的task
                            EventBus.getDefault().post(new LaunchNextTaskRequestEvent());
                        } else {
                            // Notify recents to move onto the next task 焦点移动到下一个task
                            EventBus.getDefault().post(new IterateRecentsEvent());
                        }
                    }
                } else {
                    // If the user has toggled it too quickly, then just eat up the event here (it's
                    // better than showing a janky screenshot).
                    // NOTE: Ideally, the screenshot mechanism would take the window transform into
                    // account
                    if (elapsedTime < MIN_TOGGLE_DELAY_MS) {
                        return;
                    }
					//发送隐藏Recents界面的消息
                    EventBus.getDefault().post(new ToggleRecentsEvent());
                    mLastToggleTime = SystemClock.elapsedRealtime();
                }
                return;
            } else {
                // If the user has toggled it too quickly, then just eat up the event here (it's
                // better than showing a janky screenshot).
                // NOTE: Ideally, the screenshot mechanism would take the window transform into
                // account
                //最近任务界面未启动，速度过快
                if (elapsedTime < MIN_TOGGLE_DELAY_MS) {
                    return;
                }

                // Otherwise, start the recents activity
                ActivityManager.RunningTaskInfo runningTask = ssp.getRunningTask();
				//启动RecentsActivity
                startRecentsActivity(runningTask, isHomeStackVisible.value, true /* animate */,
                        growTarget);

                // Only close the other system windows if we are actually showing recents
                ssp.sendCloseSystemWindows(StatusBar.SYSTEM_DIALOG_REASON_RECENT_APPS);
                mLastToggleTime = SystemClock.elapsedRealtime();
            }
        } catch (ActivityNotFoundException e) {
            Log.e(TAG, "Failed to launch RecentsActivity", e);
        }
    }
	
toggleRecents方法主要逻辑，当最近任务界面没有启动时，启动最近运行界面，若启动就关闭它或切换task。

当Recents没有被显示，则启动它，调用startRecentsActivity方法。
RecentsImpl：

  	/**
     * Shows the recents activity
     */
    protected void startRecentsActivity(ActivityManager.RunningTaskInfo runningTask,
            boolean isHomeStackVisible, boolean animate, int growTarget) {
    	...
        boolean isBlacklisted = (runningTask != null)
                ? ssp.isBlackListedActivity(runningTask.baseActivity.getClassName())
                : false;

		...
        boolean hasRecentTasks = stack.getTaskCount() > 0;
        boolean useThumbnailTransition = (runningTask != null) && !isHomeStackVisible &&
                hasRecentTasks;
		....
 		
        if (!animate) {//不要动画
            startRecentsActivity(ActivityOptions.makeCustomAnimation(mContext, -1, -1),
                    null /* future */);
            return;
        }

        Pair<ActivityOptions, AppTransitionAnimationSpecsFuture> pair;
        if (isBlacklisted) {
            pair = new Pair<>(getUnknownTransitionActivityOptions(), null);
        } else if (useThumbnailTransition) {//从app启动应用会有进场动画
            // Try starting with a thumbnail transition
            pair = getThumbnailTransitionActivityOptions(runningTask, windowOverrideRect);
        } else {//默认是没有动画，例如从Home进入
            // If there is no thumbnail transition, but is launching from home into recents, then
            // use a quick home transition
            pair = new Pair<>(hasRecentTasks
                    ? getHomeTransitionActivityOptions()
                    : getUnknownTransitionActivityOptions(), null);
        }
		//启动tRecentsActivity
        startRecentsActivity(pair.first, pair.second);
        mLastToggleTime = SystemClock.elapsedRealtime();
    }

	....

    /**
     * Starts the recents activity.
     */
    private void startRecentsActivity(ActivityOptions opts,
            final AppTransitionAnimationSpecsFuture future) {
        Intent intent = new Intent();
        intent.setClassName(RECENTS_PACKAGE, RECENTS_ACTIVITY);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS
                | Intent.FLAG_ACTIVITY_TASK_ON_HOME);
        HidePipMenuEvent hideMenuEvent = new HidePipMenuEvent();
        hideMenuEvent.addPostAnimationCallback(() -> {
			//启动真正启动RecentsActivity
            Recents.getSystemServices().startActivityAsUserAsync(intent, opts);
            EventBus.getDefault().send(new RecentsActivityStartingEvent());
            if (future != null) {
                future.precacheSpecs();
            }
        });
        EventBus.getDefault().send(hideMenuEvent);
    }

到此RecentsActivity已经被启动，该类是一个Activity的子类，launchMode="singleInstance"，接下来根据Activity的生命周期来分析这个类。

  	@Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
		...
        // Register this activity with the event bus 注册EventBus用于发送消息，Recents模块采用EventBus机制发送消息。
        EventBus.getDefault().register(this, EVENT_BUS_PRIORITY);
      	...
		//布局文件
        // Set the Recents layout
        setContentView(R.layout.recents);
        takeKeyEvents(true);
		//Recents所有UI的界面的绘制均在RecentsView中实现的
        mRecentsView = findViewById(R.id.recents_view);
       	...
		//设置背景
        // Set the window background
        getWindow().setBackgroundDrawable(mRecentsView.getBackgroundScrim());

      	...
		
        // Reload the stack view  
        reloadStackView();
    }

    @Override
    protected void onStart() {
        super.onStart();

        // Notify that recents is now visible
        EventBus.getDefault().send(new RecentsVisibilityChangedEvent(this, true));
        MetricsLogger.visible(this, MetricsEvent.OVERVIEW_ACTIVITY);
		
        // Notify of the next draw  当Ui开始绘制前会调用此监听器，加载task缩略图（即应用快照），前期preloadRecents里是不会加载缩略图。
        mRecentsView.getViewTreeObserver().addOnPreDrawListener(mRecentsDrawnEventListener);
    }

    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);

        // Reload the stack view
        reloadStackView();
    }

    /**
     * Reloads the stack views upon launching Recents.   onCreate和onNewIntent都会执行此方法，用于重新加载UI界面
     */
    private void reloadStackView() {
		//获取最近运行的任务。
        // If the Recents component has preloaded a load plan, then use that to prevent
        // reconstructing the task stack
        RecentsTaskLoader loader = Recents.getTaskLoader();
        RecentsTaskLoadPlan loadPlan = RecentsImpl.consumeInstanceLoadPlan();
        if (loadPlan == null) {
            loadPlan = loader.createLoadPlan(this);
        }

        // Start loading tasks according to the load plan
        RecentsConfiguration config = Recents.getConfiguration();
        RecentsActivityLaunchState launchState = config.getLaunchState();
        if (!loadPlan.hasTasks()) {
            loader.preloadTasks(loadPlan, launchState.launchedToTaskId,
                    !launchState.launchedFromHome && !launchState.launchedViaDockGesture);
        }

     	...
		//look  下面两个是钩子函数，用重新加载RecentsView界面UI
        mRecentsView.onReload(mIsVisible, stack.getTaskCount() == 0);
        mRecentsView.updateStack(stack, true /* setStackViewTasks */);

       ...
    }

几个重要生命周期中的主要逻辑已经在代码中分析了，接下来深入分析分下RecentsActivity。
布局文件：recents.xml

    <FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    
    <!-- Recents View -->
    <com.android.systemui.recents.views.RecentsView
    android:id="@+id/recents_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    </com.android.systemui.recents.views.RecentsView>
    
    <!-- Incompatible task overlay -->
    <ViewStub android:id="@+id/incompatible_app_overlay_stub"
    android:inflatedId="@+id/incompatible_app_overlay"
    android:layout="@layout/recents_incompatible_app_overlay"
    android:layout_width="match_parent"
    android:layout_height="128dp"
    android:layout_gravity="center_horizontal|top" />
    
    <!-- Nav Bar Scrim View -->
    <ImageView
    android:id="@+id/nav_bar_scrim"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_gravity="center_horizontal|bottom"
    android:scaleType="fitXY"
    android:src="@drawable/recents_lower_gradient" />
    </FrameLayout>


可以看出RecentsView是负责主要的UI显示，Recents的主要UI功能都是在RecentsView中完成的。
简单分析下RecentsView类
RecentsView：

	public class RecentsView extends FrameLayout 
可以看出RecentsView是FrameLayout的子类，其实就是一个ViewGroup。

RecentsView：
		
	//这是RecentsView的构造方法
    public RecentsView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
        setWillNotDraw(false);

        SystemServicesProxy ssp = Recents.getSystemServices();
			//A helper class to create transitions to/from Recents
        mTransitionHelper = new RecentsTransitionHelper(getContext());
        mDividerSize = ssp.getDockedDividerSize(context);
			//处理RecentsView的触摸事件
        mTouchHandler = new RecentsViewTouchHandler(this);
        mFlingAnimationUtils = new FlingAnimationUtils(context, 0.3f);
        mScrimAlpha = Recents.getConfiguration().isGridEnabled
                ? GRID_LAYOUT_SCRIM_ALPHA : DEFAULT_SCRIM_ALPHA;
		//背景图
        mBackgroundScrim = new ColorDrawable(
                Color.argb((int) (mScrimAlpha * 255), 0, 0, 0)).mutate();

        LayoutInflater inflater = LayoutInflater.from(context);
        if (RecentsDebugFlags.Static.EnableStackActionButton) {
			//清除全部按钮
            mStackActionButton = (TextView) inflater.inflate(R.layout.recents_stack_action_button,
                    this, false);
            mStackActionButton.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    EventBus.getDefault().send(new DismissAllTaskViewsEvent());
                }
            });
			//将此按钮添加到UI上
            addView(mStackActionButton);
        }
		//没有task时显示的界面
        mEmptyView = (TextView) inflater.inflate(R.layout.recents_empty, this, false);
        addView(mEmptyView);
    }


    /**
     * Called from RecentsActivity when it is relaunched.
     *  RecentsActivity会调用此方法
     */
    public void onReload(boolean isResumingFromVisible, boolean isTaskStackEmpty) {
        RecentsConfiguration config = Recents.getConfiguration();
        RecentsActivityLaunchState launchState = config.getLaunchState();
		//创建TaskStackView对象，是一个FrameLayout子类，用于显示最近任务列表UI。
        if (mTaskStackView == null) {
            isResumingFromVisible = false;
            mTaskStackView = new TaskStackView(getContext());
            mTaskStackView.setSystemInsets(mSystemInsets);
            addView(mTaskStackView);
        }

        // Reset the state
        mAwaitingFirstLayout = !isResumingFromVisible;
        mLastTaskLaunchedWasFreeform = false;

        // Update the stack
        mTaskStackView.onReload(isResumingFromVisible);

        if (isResumingFromVisible) {
            // If we are already visible, then restore the background scrim
            animateBackgroundScrim(1f, DEFAULT_UPDATE_SCRIM_DURATION);
        } else {
            // If we are already occluded by the app, then set the final background scrim alpha now.
            // Otherwise, defer until the enter animation completes to animate the scrim alpha with
            // the tasks for the home animation.
            if (launchState.launchedViaDockGesture || launchState.launchedFromApp
                    || isTaskStackEmpty) {
                mBackgroundScrim.setAlpha(255);
            } else {
                mBackgroundScrim.setAlpha(0);
            }
        }
    }	

    /** 
     * Called from RecentsActivity when the task stack is updated. 
     * RecentsActivity会调用此方法 
     */
    public void updateStack(TaskStack stack, boolean setStackViewTasks) {
        if (setStackViewTasks) {
            mTaskStackView.setTasks(stack, true /* allowNotifyStackChanges */);
        }

        // Update the top level view's visibilities
        if (stack.getTaskCount() > 0) {
            hideEmptyView();
        } else {
            showEmptyView(R.string.recents_empty_message);
        }
    }	


目前初始化工作已经完成，接下来开始绘制界面。
RecentsView：

  	/**
     * This is called with the full size of the window since we are handling our own insets.
     */
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int width = MeasureSpec.getSize(widthMeasureSpec);
        int height = MeasureSpec.getSize(heightMeasureSpec);
		//测量TaskStackView
        if (mTaskStackView.getVisibility() != GONE) {
            mTaskStackView.measure(widthMeasureSpec, heightMeasureSpec);
        }

		//mEmptyView
        // Measure the empty view to the full size of the screen
        if (mEmptyView.getVisibility() != GONE) {
            measureChild(mEmptyView, MeasureSpec.makeMeasureSpec(width, MeasureSpec.AT_MOST),
                    MeasureSpec.makeMeasureSpec(height, MeasureSpec.AT_MOST));
        }
		
		//全部清除按钮
        if (RecentsDebugFlags.Static.EnableStackActionButton) {
            // Measure the stack action button within the constraints of the space above the stack
            Rect buttonBounds = mTaskStackView.mLayoutAlgorithm.getStackActionButtonRect();
            measureChild(mStackActionButton,
                    MeasureSpec.makeMeasureSpec(buttonBounds.width(), MeasureSpec.AT_MOST),
                    MeasureSpec.makeMeasureSpec(buttonBounds.height(), MeasureSpec.AT_MOST));
        }

        setMeasuredDimension(width, height);
    }

    /**
     * This is called with the full size of the window since we are handling our own insets.
     */
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
		//布局TaskStackView界面
        if (mTaskStackView.getVisibility() != GONE) {
            mTaskStackView.layout(left, top, left + getMeasuredWidth(), top + getMeasuredHeight());
        }
		//布局mEmptyView界面
        // Layout the empty view
        if (mEmptyView.getVisibility() != GONE) {
            int leftRightInsets = mSystemInsets.left + mSystemInsets.right;
            int topBottomInsets = mSystemInsets.top + mSystemInsets.bottom;
            int childWidth = mEmptyView.getMeasuredWidth();
            int childHeight = mEmptyView.getMeasuredHeight();
            int childLeft = left + mSystemInsets.left +
                    Math.max(0, (right - left - leftRightInsets - childWidth)) / 2;
            int childTop = top + mSystemInsets.top +
                    Math.max(0, (bottom - top - topBottomInsets - childHeight)) / 2;
            mEmptyView.layout(childLeft, childTop, childLeft + childWidth, childTop + childHeight);
        }
	
		//布局全部清除按钮
        if (RecentsDebugFlags.Static.EnableStackActionButton) {
            // Layout the stack action button such that its drawable is start-aligned with the
            // stack, vertically centered in the available space above the stack
            Rect buttonBounds = getStackActionButtonBoundsFromStackLayout();
            mStackActionButton.layout(buttonBounds.left, buttonBounds.top, buttonBounds.right,
                    buttonBounds.bottom);
        }

       	...
    }

onMeasure和onLayout主要
