在上一篇文章[OpenGLES + MediaCodec 短视频分段录制实现与无丢帧录制优化](http://www.jianshu.com/p/9dc03b01bae3)中说到了MediaCodec录制视频的一些优化思路。经过上一步，我们就实现了短视频的分段录制功能以及录制视频帧率的优化。在得到多段视频之后，我们如何将这些细小的视频文件合成一个视频文件呢？ 这是有好多种方法，比如通过MP4Parser、FFmpeg等进行合成。这里介绍如何使用Android原生的MediaExtractor + MediaMuxer进行对多段MP4视频文件进行合成。

1、[MediaExtractor](https://developer.android.google.cn/reference/android/media/MediaExtractor.html) 提取媒体信息
MediaExtractor 主要的作用是从数据源中提取媒体数据，并将媒体数据传递给解复用器(通常是解码器) 
使用MediaExtractor 一般是这样的，不断循环读取VideoPath/AudioPath，将媒体信息从文件中读取出来， 筛选出不同的轨道trackIndex并通过getTrackFormat(trackIndex)方法获取对应的轨道信息：
```
        // MediaExtractor拿到多媒体信息，用于MediaMuxer创建文件
        while (videoIterator.hasNext()) {
            String videoPath = (String) videoIterator.next();
            MediaExtractor extractor = new MediaExtractor();
            try {
                extractor.setDataSource(videoPath);
            } catch (Exception ex) {
                ex.printStackTrace();
            }

            int trackIndex;
            if (!hasVideoFormat) {
                trackIndex = selectTrack(extractor, "video/");
                if(trackIndex < 0) {
                    Log.e(TAG, "No video track found in " + videoPath);
                } else {
                    extractor.selectTrack(trackIndex);
                    mVideoFormat = extractor.getTrackFormat(trackIndex);
                    hasVideoFormat = true;
                }
            }

            if (!hasAudioFormat) {
                trackIndex = selectTrack(extractor, "audio/");
                if(trackIndex < 0) {
                    Log.e(TAG, "No audio track found in " + videoPath);
                } else {
                    extractor.selectTrack(trackIndex);
                    mAudioFormat = extractor.getTrackFormat(trackIndex);
                    hasAudioFormat = true;
                }
            }

            extractor.release();

            if (hasVideoFormat && hasAudioFormat) {
                break;
            }
        }

/**
     *  选择轨道
     * @param extractor     MediaExtractor
     * @param mimePrefix    音轨还是视轨
     * @return
     */
    private int selectTrack(MediaExtractor extractor, String mimePrefix) {
        // 获取轨道总数
        int numTracks = extractor.getTrackCount();
        // 遍历查找包含mimePrefix的轨道
        for(int i = 0; i < numTracks; ++i) {
            MediaFormat format = extractor.getTrackFormat(i);
            String mime = format.getString("mime");
            if (mime.startsWith(mimePrefix)) {
                return i;
            }
        }

        return -1;
    }


```
2、创建[MediaMuxer](https://developer.android.google.cn/reference/android/media/MediaMuxer.html)并添加相应的轨道
通过MediaExtractor 拿到媒体的轨道trackIndex 和 轨道信息trackFormat之后，我们就可以创建一个MediaMuxer复用器并添加相应的轨道(音频轨道、视频轨道等)：
```
        try {
            mMuxer = new MediaMuxer(mDestPath, MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4);
        } catch (IOException e) {
            e.printStackTrace();
        }

        if (hasVideoFormat) {
            mOutVideoTrackIndex = mMuxer.addTrack(mVideoFormat);
        }

        if (hasAudioFormat) {
            mOutAudioTrackIndex = mMuxer.addTrack(mAudioFormat);
        }
        mMuxer.start();
```

3、合并视频
根据前面得到的媒体信息、媒体轨道等信息就可以进行合并了。合并的流程大体如下所示：
① 选择视频的轨道(音频轨道、视频轨道等)，判断是否存在音视频轨道，并获取对应轨道的index。
② 判断视频的视频轨道和视频轨道是否存在，如果都不存在，则该视频文件很可能损坏了，合并失败，并退出。
③ 如果存在音频轨道、视频轨道，进入合并阶段
④ 通过MediaExtractor的readSampleData()方法从视频中读取音频轨道、视频轨道的数据，并判断读取出来的数据帧是否大于0，如果大于0，则用getSampleTime()方法读取数据帧的pts，并通过getSampleFlags()读取flag标志，然后创建一个BufferInfo类用来保存这些帧数据。
⑤ 将MediaExtractor提取的数据，通过MediaMuxer 的writeSampleData()方法将媒体数据写入到新的文件
⑥ 遍历下一个视频文件，并计算从下一个视频文件提取出来的媒体数据的PTS，用于下一次的视频合成
⑦ 释放MediaMuxer、MediaExtractor

至此，我们就把多段MP4视频合成功能的流程介绍完了，完整的合并代码如下所示：
```
@TargetApi(18)
public class VideoCombiner {

    private static final String TAG = "VideoCombiner";
    private static final boolean VERBOSE = true;

    // 最大缓冲区(1024 x 1024 = 1048576、1920 x 1080 = 2073600)
    // 由于没有录制1080P的视频，因此用1024的Buffer来缓存
    private static final int MAX_BUFF_SIZE = 1048576;

    private List<String> mVideoList;
    private String mDestPath;

    private MediaMuxer mMuxer;
    private ByteBuffer mReadBuf;
    private int mOutAudioTrackIndex;
    private int mOutVideoTrackIndex;
    private MediaFormat mAudioFormat;
    private MediaFormat mVideoFormat;

    private VideoCombineListener mCombineListener;

    public interface VideoCombineListener {

        /**
         * 合并开始
         */
        void onCombineStart();

        /**
         * 合并过程
         * @param current 当前合并的视频
         * @param sum   合并视频总数
         */
        void onCombineProcessing(int current, int sum);

        /**
         * 合并结束
         * @param success   是否合并成功
         */
        void onCombineFinished(boolean success);
    }



    public VideoCombiner(List<String> videoList, String destPath, VideoCombineListener listener) {
        mVideoList = videoList;
        mDestPath = destPath;
        mCombineListener = listener;
        mReadBuf = ByteBuffer.allocate(MAX_BUFF_SIZE);
    }

    /**
     * 合并视频
     * @return
     */
    @SuppressLint("WrongConstant")
    public void combineVideo() {
        boolean hasAudioFormat = false;
        boolean hasVideoFormat = false;
        Iterator videoIterator = mVideoList.iterator();

        // 开始合并
        if (mCombineListener != null) {
            mCombineListener.onCombineStart();
        }

        // MediaExtractor拿到多媒体信息，用于MediaMuxer创建文件
        while (videoIterator.hasNext()) {
            String videoPath = (String) videoIterator.next();
            MediaExtractor extractor = new MediaExtractor();
            try {
                extractor.setDataSource(videoPath);
            } catch (Exception ex) {
                ex.printStackTrace();
            }

            int trackIndex;
            if (!hasVideoFormat) {
                trackIndex = selectTrack(extractor, "video/");
                if(trackIndex < 0) {
                    Log.e(TAG, "No video track found in " + videoPath);
                } else {
                    extractor.selectTrack(trackIndex);
                    mVideoFormat = extractor.getTrackFormat(trackIndex);
                    hasVideoFormat = true;
                }
            }

            if (!hasAudioFormat) {
                trackIndex = selectTrack(extractor, "audio/");
                if(trackIndex < 0) {
                    Log.e(TAG, "No audio track found in " + videoPath);
                } else {
                    extractor.selectTrack(trackIndex);
                    mAudioFormat = extractor.getTrackFormat(trackIndex);
                    hasAudioFormat = true;
                }
            }

            extractor.release();

            if (hasVideoFormat && hasAudioFormat) {
                break;
            }
        }

        try {
            mMuxer = new MediaMuxer(mDestPath, MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4);
        } catch (IOException e) {
            e.printStackTrace();
        }

        if (hasVideoFormat) {
            mOutVideoTrackIndex = mMuxer.addTrack(mVideoFormat);
        }

        if (hasAudioFormat) {
            mOutAudioTrackIndex = mMuxer.addTrack(mAudioFormat);
        }
        mMuxer.start();

        // MediaExtractor遍历读取帧，MediaMuxer写入帧，并记录帧信息
        long ptsOffset = 0L;
        Iterator trackIndex = mVideoList.iterator();
        int currentVideo = 0;
        boolean combineResult = true;
        while (trackIndex.hasNext()) {
            // 监听当前合并第几个视频
            currentVideo++;
            if (mCombineListener != null) {
                mCombineListener.onCombineProcessing(currentVideo, mVideoList.size());
            }

            String videoPath = (String) trackIndex.next();
            boolean hasVideo = true;
            boolean hasAudio = true;

            // 选择视频轨道
            MediaExtractor videoExtractor = new MediaExtractor();
            try {
                videoExtractor.setDataSource(videoPath);
            } catch (Exception e) {
                e.printStackTrace();
            }
            int inVideoTrackIndex = selectTrack(videoExtractor, "video/");
            if(inVideoTrackIndex < 0) {
                hasVideo = false;
            }
            videoExtractor.selectTrack(inVideoTrackIndex);

            // 选择音频轨道
            MediaExtractor audioExtractor = new MediaExtractor();
            try {
                audioExtractor.setDataSource(videoPath);
            } catch (Exception e) {
                e.printStackTrace();
            }
            int inAudioTrackIndex = selectTrack(audioExtractor, "audio/");
            if (inAudioTrackIndex < 0) {
                hasAudio = false;
            }
            audioExtractor.selectTrack(inAudioTrackIndex);

            // 如果存在视频轨道和音频轨道都不存在，则合并失败，文件出错
            if (!hasVideo && !hasAudio) {

                combineResult = false;

                videoExtractor.release();
                audioExtractor.release();

                break;
            }

            boolean bMediaDone = false;
            long presentationTimeUs = 0L;
            long audioPts = 0L;
            long videoPts = 0L;

            while (!bMediaDone) {
                // 判断是否存在音视频
                if(!hasVideo && !hasAudio) {
                    break;
                }

                int outTrackIndex;
                MediaExtractor extractor;
                int currentTrackIndex;
                if ((!hasVideo || audioPts - videoPts <= 50000L) && hasAudio) {
                    currentTrackIndex = inAudioTrackIndex;
                    outTrackIndex = mOutAudioTrackIndex;
                    extractor = audioExtractor;
                } else {
                    currentTrackIndex = inVideoTrackIndex;
                    outTrackIndex = mOutVideoTrackIndex;
                    extractor = videoExtractor;
                }

                if (VERBOSE) {
                    Log.d(TAG, "currentTrackIndex： " + currentTrackIndex
                            + ", outTrackIndex: " + outTrackIndex);
                }

                mReadBuf.rewind();
                // 读取数据帧
                int frameSize = extractor.readSampleData(mReadBuf, 0);
                if (frameSize < 0) {
                    if (currentTrackIndex == inVideoTrackIndex) {
                        hasVideo = false;
                    } else if (currentTrackIndex == inAudioTrackIndex) {
                        hasAudio = false;
                    }
                } else {
                    if (extractor.getSampleTrackIndex() != currentTrackIndex) {
                        Log.e(TAG, "got sample from track "
                                + extractor.getSampleTrackIndex()
                                + ", expected " + currentTrackIndex);
                    }

                    // 读取帧的pts
                    presentationTimeUs = extractor.getSampleTime();
                    if (currentTrackIndex == inVideoTrackIndex) {
                        videoPts = presentationTimeUs;
                    } else {
                        audioPts = presentationTimeUs;
                    }

                    // 帧信息
                    BufferInfo info = new BufferInfo();
                    info.offset = 0;
                    info.size = frameSize;
                    info.presentationTimeUs = ptsOffset + presentationTimeUs;

                    if ((extractor.getSampleFlags() & MediaCodec.BUFFER_FLAG_KEY_FRAME) != 0) {
                        info.flags = MediaCodec.BUFFER_FLAG_KEY_FRAME;
                    }
                    mReadBuf.rewind();
                    if (VERBOSE) {
                        Log.d(TAG, String.format("write sample track %d, size %d, pts %d flag %d",
                                Integer.valueOf(outTrackIndex),
                                Integer.valueOf(info.size),
                                Long.valueOf(info.presentationTimeUs),
                                Integer.valueOf(info.flags))
                        );
                    }
                    // 将读取到的数据写入文件
                    mMuxer.writeSampleData(outTrackIndex, mReadBuf, info);
                    extractor.advance();
                }
            }

            // 当前文件最后一帧的PTS，用作下一个视频的pts
            ptsOffset += videoPts > audioPts ? videoPts : audioPts;
            // 当前文件最后一帧和下一帧的间隔差40ms，默认录制25fps的视频，帧间隔时间就是40ms
            // 但由于使用MediaCodec录制完之后，后面又写入了一个OES的帧，导致前面解析的时候会有时间差
            // 这里设置10ms效果比40ms的要好些。
            ptsOffset += 10000L;

            if (VERBOSE) {
                Log.d(TAG, "finish one file, ptsOffset " + ptsOffset);
            }

            // 释放资源
            videoExtractor.release();
            audioExtractor.release();
        }

        // 释放复用器
        if (mMuxer != null) {
            try {
                mMuxer.stop();
                mMuxer.release();
            } catch (Exception e) {
                Log.e(TAG, "Muxer close error. No data was written");
            } finally {
                mMuxer = null;
            }
        }

        if (VERBOSE) {
            Log.d(TAG, "video combine finished");
        }

        // 合并结束
        if (mCombineListener != null) {
            mCombineListener.onCombineFinished(combineResult);
        }
    }

    /**
     *  选择轨道
     * @param extractor     MediaExtractor
     * @param mimePrefix    音轨还是视轨
     * @return
     */
    private int selectTrack(MediaExtractor extractor, String mimePrefix) {
        // 获取轨道总数
        int numTracks = extractor.getTrackCount();
        // 遍历查找包含mimePrefix的轨道
        for(int i = 0; i < numTracks; ++i) {
            MediaFormat format = extractor.getTrackFormat(i);
            String mime = format.getString("mime");
            if (mime.startsWith(mimePrefix)) {
                return i;
            }
        }

        return -1;
    }
}
```
4、如何使用
为了方便使用，我们可以创建一个Manager来管理合并视频功能，这样我们就可以在Activity、Fragment里面直接通过调用VideoCombineManager.getInstance().startVideoCombiner()方法，传递需要合并的路径列表、输出路径、合并状态监听接口参数，使用起来非常方便：
```
public final class VideoCombineManager {

    private static final String TAG = "VideoCombineManage";

    private static VideoCombineManager mInstance;


    public static VideoCombineManager getInstance() {
        if (mInstance == null) {
            mInstance = new VideoCombineManager();
        }
        return mInstance;
    }

    /**
     * 初始化媒体合并器
     * @param videoPath
     * @param destPath
     */
    public void startVideoCombiner(final List<String> videoPath, final String destPath,
                                   final VideoCombiner.VideoCombineListener listener) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                VideoCombiner videoCombiner = new VideoCombiner(videoPath, destPath, listener);
                videoCombiner.combineVideo();
            }
        }).start();
    }
}
```

5、MediaExtractor + MediaMuxer 合并视频的优缺点
在录制视频合成方面，MediaExtractor + MediaMuxer 比较适合应用内简单的视频合成场景，比如在相机里面录制了多段视频，然后跳转至预览页面直接合成视频这样的场景。如果要做短视频SDK，估计有点力不从心，比如短视频SDK里面，需要添加字幕、添加mkv等功能，MediaExtractor + MediaMuxer组合是做不到的。需要做短视频SDK的童鞋，还是老老实实用FFmpeg这样成熟的框架吧。








