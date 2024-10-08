
# android 版本
android-12.0.0_r3
## 应用调用
```
Toast toast = new Toast(this);
toast.setDuration(Toast.LENGTH_SHORT);
toast.setText("hello");
toast.show();
```
## Toast构造函数
```
 //创建Token
 mToken = new Binder();
 //创建TN 对象，用于与NotificationManagerService 通信
  mTN = new TN(context, context.getPackageName(), mToken,mCallbacks, looper);
```
### TN解析
TN  是Toast 内部类
```
class TN extends ITransientNotification.Stub
```
### TN核心参数
```
//负责在应用程序进程和System UI 中呈现 toast 的类。
ToastPresenter mPresenter 
```
### TN 构造函数
```
 //创建ToastPresenter 对象，其中getService是获取NotificationManagerService 的代理对象，用于与之通信
 mPresenter = new ToastPresenter(context, accessibilityManager, getService(),packageName);
 //获取布局参数
 mParams = mPresenter.getLayoutParams();
 //创建Handler 对象
 mHandler = new Handler(looper, null)
```
### ToastPresenter 构造函数
该构造方法核心都是些对象赋值，其中 createLayoutParams   方法调用用与设置默认的Toast布局参数，代码如下：
```
    private WindowManager.LayoutParams createLayoutParams() {        
        WindowManager.LayoutParams params = new WindowManager.LayoutParams();        
        params.height = WindowManager.LayoutParams.WRAP_CONTENT;        
        params.width = WindowManager.LayoutParams.WRAP_CONTENT;        
        params.format = PixelFormat.TRANSLUCENT;        
        params.windowAnimations = R.style.Animation_Toast;        
        params.type = WindowManager.LayoutParams.TYPE_TOAST;        
        params.setFitInsetsIgnoringVisibility(true);        
        params.setTitle(WINDOW_TITLE);        
        params.flags = WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON                
        | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE                
        | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;        
        setShowForAllUsersIfApplicable(params, mPackageName);        
        return params;    
     }
```
## setDuration 解析
```
 public void setDuration(@Duration int duration) {        
     mDuration = duration;  
     //将duration 给TN 对象
     mTN.mDuration = duration;    
 }   
```

## setText 解析
```
public void setText(@StringRes int resId) {
    setText(mContext.getText(resId));
}   
 /**     
 * Update the text in a Toast that was previously created using one of the makeText() methods.     
 * @param s The new text for the Toast.     
 */    
 public void setText(CharSequence s) {
     // Text toasts will be rendered by SystemUI instead of in-app, so apps can't circumvent
     if (Compatibility.isChangeEnabled(CHANGE_TEXT_TOASTS_IN_THE_SYSTEM)) {
         if (mNextView != null) {                
             throw new IllegalStateException(                        
                 "Text provided for custom toast, remove previous setView() calls if you "
                                + "want a text toast instead.");            
         }            
         mText = s;        
      } else {//表示在app 内渲染
          if (mNextView == null) {                
              throw new RuntimeException("This Toast was not created with Toast.makeText()");            
           }            
           TextView tv = mNextView.findViewById(com.android.internal.R.id.message);            
           if (tv == null) {                
                 throw new RuntimeException("This Toast was not created with Toast.makeText()");            
           }            
           tv.setText(s);        
      }    
  }
```

## show解析
```
/**
* Show the view for the specified duration.
*/    
public void show() {
        //在systemui 里面渲染
        if (Compatibility.isChangeEnabled(CHANGE_TEXT_TOASTS_IN_THE_SYSTEM)) {
                //mNextView 和 text 至少有一个不为空
               checkState(mNextView != null || mText != null, "You must either set a text or a view");
         } else {            
               if (mNextView == null) {
                      throw new RuntimeException("setView must have been called");
                }
         }        
         //NotificationManagerService 代理
         INotificationManager service = getService();
         String pkg = mContext.getOpPackageName();
         TN tn = mTN;
         tn.mNextView = mNextView;
         final int displayId = mContext.getDisplayId();
         try {            
             if (Compatibility.isChangeEnabled(CHANGE_TEXT_TOASTS_IN_THE_SYSTEM)) { 
                       if (mNextView != null) {                    
                           // It's a custom toast
                           service.enqueueToast(pkg, mToken, tn, mDuration, displayId);
                        } else {
                           // It's a text toast
                           ITransientNotificationCallback callback =
                               new CallbackBinder(mCallbacks, mHandler);
                           service.enqueueTextToast(pkg, mToken, mText, mDuration, displayId, callback);
                        }          
               } else {
                       service.enqueueToast(pkg, mToken, tn, mDuration, displayId);
               }        
          } catch (RemoteException e) { 
                     // Empty        
          }    
    }
```
```
   public static void checkState(final boolean expression, String errorMessage) {
            if (!expression) {
                  throw new IllegalStateException(errorMessage);
            }
    }
```
我们先a text toast 为例分析，view的后面再补充
```
//接收NotificationManagerService的回调
  ITransientNotificationCallback callback =
     new CallbackBinder(mCallbacks, mHandler);
//将消息入栈
  service.enqueueTextToast(pkg, mToken, mText, mDuration, displayId, callback);
```
### NotificationManagerService toast 分发流程
