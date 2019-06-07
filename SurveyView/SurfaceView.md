# SurfaceView #
SurfaceView也是View的一个子类，不过绘制画面是在子线程中操作，不会影响ui线程，对于频繁更新界面的
操作可以使用SurfaceView来实现，例如VideoView用来播放视频会频繁刷新界面，就是使用SurfaceView实现的。

## 常用模板 ##
	//1.自定义一个类继承SurfaceView并且实现SurfaceHolder.Callback，Runnable
	public class MSurfaceView extends SurfaceView implements SurfaceHolder.Callback, Runnable {
    	private SurfaceHolder mHolder;
    	private Paint mPaint;

    	private boolean isRunning;

    	public MSurfaceView(Context context) {
        	super(context);
       	    init();
   		}

    	public MSurfaceView(Context context, AttributeSet attrs) {
        	super(context, attrs);
        	init();
   		 }

    	public MSurfaceView(Context context, AttributeSet attrs, int defStyleAttr) {
        	super(context, attrs, defStyleAttr);
	        init();
	    }

    	private void init() {
			//2.获取SurfaceView对应的SurfaceHolder
        	mHolder = getHolder();
			//3.将SurfaceHolder.Callback绑定到SurfaceHolder，即，绑定了Callback的surfaceCreated，
			//surfaceChanged，surfaceDestroyed实现的3个方法到SurfaceView
        	mHolder.addCallback(this);
        	mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        	mPaint.setColor(Color.RED);
        	mPaint.setTextSize(40);

        	//设置可获取焦点
       	 	setFocusable(true);
       		 setFocusableInTouchMode(true);

        	//设置常亮
        	setKeepScreenOn(true);
   	 	}

   
   	 	@Override
    	public void surfaceCreated(SurfaceHolder holder) {
			//4.定义一个isRunning标志位，判断是否停止绘制。
   	     	 isRunning = true;
			//5.启动绘制线程。
       		 new Thread(this).start();
   	 	}	

   
    	@Override
    	public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
       		
   		}

   
    	@Override
    	public void surfaceDestroyed(SurfaceHolder holder) {
        	//6.关闭绘制
        	isRunning = false;
    	}

    
    	@Override
    	public void run() {
        	//不断进行draw
        	while (isRunning) {
            	draw();
        	}
	
    	}
		//7.在此方法中执行绘制
    	private void draw() {

        	Canvas canvas = mHolder.lockCanvas();

        	try {

            	if (canvas != null) {
                	//自定义组件类似，通过canvas绘制内容
            	}
            	Thread.sleep(1000);
       	 	} catch (InterruptedException e) {
            	e.printStackTrace();
        	} finally {
            	if (canvas != null) {
               	 mHolder.unlockCanvasAndPost(canvas);
            }
        		}

    		}
		}
 