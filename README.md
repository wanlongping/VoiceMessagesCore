语音留言
===
# 一、简介
该语音留言模块解决了通过语音留言的方式告知其他用户相关信息。通常这个功能的业务逻辑是这样的：
1、无记录界面
1）若无留言记录，则显示“请长按录音，开始录制”；
2）录音时间不足，取消保存。当不足2s不保存，toast提示：录音时间太短；
3）取消录制，上划手指即可；
4）保存录音留言。正常抬起松开手指，保存录音至留言列表；
5）超时录音保存。录音时长限时1min,录音时间>=60s时即使手指不松开，留言也保存到录音列表；
6）录音中，显示时长进度，显示最终留言时长；
7）录音留言中断（结束录音自动保存到留言列表）。

2、有记录界面
1）总留言时长显示，气泡显示实际留言时长；
2）留言日期、时间：今天+时间（今天 23：59），昨天+时间（昨天 23：59），日期+时间（2019-03-25 23：59）；
3）录音结束引导框，“长按气泡可以进行【标为未读】和【删除】留言”，同时可将引导框勾选为不再提示；
4）留言“标为未读”和“删除”。点击“标为未读”后将录音留言标为未读，留言右上角呈现未读小红点，同时将录音留言添加至主界面留言管理便签。点击“删除”后将弹框将录音留言删除；	
5）清空所有留言，弹框询问是否清空；
6）列表限量30条留言，超过部分依次覆盖旧留言。且留言按时间排序。

这里面涉及很多录音过程中的逻辑处理，细节不太好说，感兴趣可以直接看代码。这里直接介绍下如何使用。
# 二、简单使用
首先是最简单的使用（这里是录音按钮触摸事件的监听，当手指抬起时，保存录音文件）

```
/**
 * 手指滑动监听
 * @param event
 * @return
 */
@Override
public boolean onTouchEvent(MotionEvent event) {
    int action = event.getAction();
    int x = (int) event.getX();
    int y = (int) event.getY();

    switch (action) {
        case MotionEvent.ACTION_DOWN:
            break;
        case MotionEvent.ACTION_MOVE:
            if (isRecording) {
                // 根据x，y来判断用户是否想要取消
                if (wantToCancel(x, y)) {
                    changeState(STATE_WANT_TO_CANCEL);
                } else {
                    if (!isOverTime)
                        changeState(STATE_RECORDING);
                }
            }
            break;
        case MotionEvent.ACTION_UP:
            //发送停止录音播放动画显示的广播
            getContext().sendBroadcast(new Intent(RESET_ANIM_ACTION));
            //取消控制不进入屏保模式
            mKeyHandler.removeCallbacks(mKeyRunnable);
            // 首先判断是否有触发onlongclick事件，没有的话直接返回reset
            if (!mReady) {
                reset();
                return super.onTouchEvent(event);
            }
            // 如果按的时间太短，还没准备好或者时间录制太短不足3秒，就离开了，则显示这个dialog
            if (!isRecording || mTime < 2f) {
                mRecordingMessagesActivity.timeTooShort();
                mAudioManager.cancel();
                mStateHandler.sendEmptyMessageDelayed(MSG_DIALOG_DIMISS, 500);// 持续1.3s
            } else if (mCurrentState == STATE_RECORDING) {//正常录制结束
                if (isOverTime) return super.onTouchEvent(event);//超时
                //隐藏录音动画框
                mRecordingMessagesActivity.dimissRecordingDialog();
                //判断checkbox是否勾选，如果勾选，则不显示引导框，如果没被勾选，则每次录完依旧显示引导框，且延时十秒自动隐藏
                mPrefs = mContext.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE);
                boolean isCheckBoxCheckedState = mPrefs.getBoolean("isCheckBoxChecked",false);
                if(isCheckBoxCheckedState){
                    mRecordingMessagesActivity.mPromptBox.setVisibility(GONE);
                }else{
                    //显示引导框
                    mRecordingMessagesActivity.mPromptBox.setVisibility(VISIBLE);
                    //设置延时十秒让引导框自动消失
                    mRecordingMessagesActivity.mPromptBox.postDelayed(new Runnable() {
                        @Override
                        public void run() {
                            mRecordingMessagesActivity.mPromptBox.setVisibility(GONE);
                        }
                    },10000);
                }
                // release释放一个mediarecorder
                mAudioManager.release();
                // 并且callbackActivity，保存录音
                if (mListener != null) {
                    mListener.onFinished(mTime, mAudioManager.getCurrentFilePath());
                }
            } else if (mCurrentState == STATE_WANT_TO_CANCEL) {
                // cancel
                mAudioManager.cancel();
                mRecordingMessagesActivity.dimissWantToRecordingDialog();
            }
            // 恢复标志位
            reset();
            //进度条计时取消
            countDownTimer.cancel();
            curPercentate = 0;
            percentToAngle(curPercentate);
            break;
    }
    return super.onTouchEvent(event);
}


```
按钮抬起时，录音保存方法

```
/**
 *按钮抬起时保存录音留言的方法
 */
public void saveRecordFile() {
    // 首先判断是否有触发onlongclick事件，没有的话直接返回reset
    if (!mReady) {
        reset();
    }
    //正常录制结束
    if (mCurrentState == STATE_RECORDING ) {
        //隐藏录音动画框
        mRecordingMessagesActivity.dimissRecordingDialog();
        // release释放一个mediarecorder
        mAudioManager.release();
        // 并且callbackActivity，保存录音
        if (mListener != null) {
            mListener.onFinished(mTime, mAudioManager.getCurrentFilePath());
        }
    }
    // 恢复标志位
    reset();
    //进度条计时取消
    countDownTimer.cancel();
    curPercentate = 0;
    percentToAngle(curPercentate);
}
```
自定义进度条
```
//进度条
TypedArray array = context.obtainStyledAttributes(attrs, R.styleable.CiclePercentView);
radius = array.getInt(R.styleable.CiclePercentView_radius,67);
array.recycle();
//进度条初始化
initProgressBar();

/**
 * 进度条的初始化
 */
private void initProgressBar() {
    paint = new Paint();
    paint.setColor(Color.BLUE);
    paint.setStyle(Paint.Style.STROKE);
    paint.setAntiAlias(true);
    paint.setStrokeWidth(8);
    //起始角度
    startAngle = -90;
}

/**
 * 进度条绘制
 * @param canvas
 */
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    //画圆弧
    RectF rectf = new RectF(1,1,dp2px(radius),dp2px(radius));
    canvas.drawArc(rectf,startAngle,curAngle,false,paint);
}

/**
 * 进度条加载百分比控制
 * @param percentage
 */
private void percentToAngle(int percentage){
    curAngle =  (int) (percentage/100f*360);
    invalidate();
}

/**
 * 进度条按下时间计算
 * @param countdownTime
 */
public void setCountdownTime(int countdownTime){
    this.countdownTime = countdownTime;
}

/**
 * 进度条计算按下时间
 * @param totalTime
 */
public void countDown(final int totalTime){
    countDownTimer = new CountDownTimer(totalTime, (long)(totalTime/100f)) {
        @Override
        public void onTick(long millisUntilFinished) {
            curPercentate = (int) ((totalTime-millisUntilFinished)/(float)totalTime*100);
            percentToAngle(curPercentate);
        }
        @Override
        public void onFinish() {
            curPercentate = 0;
            percentToAngle(curPercentate);
        }
    }.start();
}

/**
 * 进度条尺寸控制
 * @param dp
 * @return
 */
private int dp2px(int dp){
    return (int) (getContext().getResources().getDisplayMetrics().density*dp + 0.5);
}

public interface AudioFinishRecorderListener {
    void onFinished(float seconds, String filePath);
}

public void setAudioFinishRecorderListener(AudioFinishRecorderListener listener) {
    mListener = listener;
}

```
录音留言的增删改查操作
```
/**
 * 数据库操作类，增、删、改、查
 *
 * @author: wlp 2018年11月3 创建<br>
 */
public class RecordingDao {
    //db
    private DBManager mgr;
    public RecordingDao(Context context) {
        //初始化DBManager
        mgr = new DBManager(context);
    }

    public void add(Record record) {
        if (record == null) {
            return;
        }
        mgr.add(record);
    }

    public void updateRecord(Record record) {
        mgr.updateRecord(record);
    }

    public void deleteRecord(Record record) {
        mgr.deleteRecord(record);
    }

    /**
     * query all Records, return list
     * @return List<Record>
     */
    public List<Record> query() {
        return mgr.query();
    }

    /**
     * 清空一个数据库表内容
     */
    public void clearTable() {
        mgr.clearTable("Record");

    }
}
```

# 三、后记
目前录音留言功能支持手机和平板终端集成去进行应用，用户也可根据自身需求对其进行适配和裁剪等。

