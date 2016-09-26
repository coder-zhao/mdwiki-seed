## 监听系统截屏事件

#### 1.实现原理

​      在Android系统中SDK并没有提供对于截屏进行监听的API所以只有根据截屏特性进行处理基本上有以下几种方案

###### 1.监听广播

很遗憾的是系统并没有提供类似广播来进行监听

###### 2.监听按钮  

android手机按下“电源键+音量减”会进行截屏，此外大部分手机状态栏下拉的页面中也会有截屏按钮。遗憾的是，监听这两处的操作并不是一件让人开心的事儿~~。

###### 3.监听产生的文件

截屏呢他会产生两个地方的变化一个当然是会产生一个新的文件，另一个就是他会在系统媒体库中去产生一条记录用于相册应用来进行快速的图片展示所有就有了以下思路使用ContentObserver，监听      1.MediaStore.Images.Media.EXTERNAL_CONTENT_URI资源的变化

2.使用FileObserver，监听Screenshots目录下的文件变化；    

#### 2.实现代码

```java
package com.youloft.util;

import android.app.Activity;
import android.content.ContentResolver;
import android.database.ContentObserver;
import android.database.Cursor;
import android.net.Uri;
import android.provider.MediaStore;
import android.util.Log;

/**
 * 截屏监听
 * <p>
 * <p>
 * 实现原理:
 * 监听媒体库的新入文件判断 Path中有[screenshot]等截屏关键字 且 创建时间与当前时间在一定范围内的认定为截屏
 * <p>
 * <p>
 * <p>
 * Created by coder on 16/9/21.
 */
public class ScreenShotDetector {

    public interface ScreenShotListener {
        void onScreenShot(String path);
    }


    private static final String TAG = "RxScreenshotDetector";
    private static final String EXTERNAL_CONTENT_URI_MATCHER =
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI.toString();
    private static final String[] PROJECTION = new String[]{
            MediaStore.Images.Media.DISPLAY_NAME, MediaStore.Images.Media.DATA,
            MediaStore.Images.Media.DATE_ADDED
    };
    private static final String SORT_ORDER = MediaStore.Images.Media.DATE_ADDED + " DESC";
    private static final long DEFAULT_DETECT_WINDOW_SECONDS = 10;

    private ContentResolver contentResolver;

    private Activity context;

    private ScreenShotListener mShotListener;

    public ScreenShotDetector(Activity context, ScreenShotListener listener) {
        contentResolver = context.getContentResolver();
        this.context = context;
        this.mShotListener = listener;
    }

    final ContentObserver contentObserver = new ContentObserver(null) {
        @Override
        public void onChange(boolean selfChange, Uri uri) {
            Log.d(TAG, "onChange: " + selfChange + ", " + uri.toString());
            if (uri.toString().matches(EXTERNAL_CONTENT_URI_MATCHER)) {
                Cursor cursor = null;
                try {
                    cursor = contentResolver.query(uri, PROJECTION, null, null,
                            SORT_ORDER);
                    if (cursor != null && cursor.moveToFirst()) {
                        String path = cursor.getString(
                                cursor.getColumnIndex(MediaStore.Images.Media.DATA));
                        long dateAdded = cursor.getLong(cursor.getColumnIndex(
                                MediaStore.Images.Media.DATE_ADDED));
                        long currentTime = System.currentTimeMillis() / 1000;
                        Log.d(TAG, "path: " + path + ", dateAdded: " + dateAdded +
                                ", currentTime: " + currentTime);
                        if (matchPath(path) && matchTime(currentTime, dateAdded)) {
                            onScreenShotEvent(path);
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                    Log.d(TAG, "open cursor fail");
                } finally {
                    if (cursor != null) {
                        cursor.close();
                    }
                }
            }
            super.onChange(selfChange, uri);
        }
    };

    /**
     * 开始
     */
    public void start() {
        contentResolver.registerContentObserver(
                MediaStore.Images.Media.EXTERNAL_CONTENT_URI, true, contentObserver);
    }

    /**
     * 结束
     */
    public void stop() {
        contentResolver.unregisterContentObserver(contentObserver);
    }

    /**
     * 截屏事件发生
     *
     * @param path
     */
    private void onScreenShotEvent(final String path) {
        this.context.runOnUiThread(new Runnable() {
            @Override
            public void run() {
                if (mShotListener != null) {
                    mShotListener.onScreenShot(path);
                }
            }
        });
    }


    private static boolean matchPath(String path) {
        return path.toLowerCase().contains("screenshot")
                || path.contains("截屏")
                || path.contains("截图")
                || path.toLowerCase().contains("screenshots");
    }

    private static boolean matchTime(long currentTime, long dateAdded) {
        return Math.abs(currentTime - dateAdded) <= DEFAULT_DETECT_WINDOW_SECONDS;
    }
}

```

