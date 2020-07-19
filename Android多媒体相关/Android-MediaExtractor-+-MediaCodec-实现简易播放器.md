今天我们来讲讲如何使用MediaExtractor + MediaCodec实现一个简易的播放器。
我们都知道MediaCodec是Android 环境下的硬编解码器，而MediaExtractor 则给我们提供了读取视频等媒体文件信息的功能。
如何使用MediaExtractor 和 MediaCodec 实现一个简易的播放器呢？其实并不难，整体的流程如下：
1、创建视频解码线程
2、创建音频解码线程
3、开始视频解码
4、开始音频解码
5、解码播放延时同步

首先，我们来看看视频解码线程如何实现：
视频解码的大体流程
1、获取视频的轨道信息
2、创建MediaCodec
3、将解复用得到的数据传递给解码器
4、获取解码后的数据
5、显示输出
实现代码如下：
```
/**
     * 视频解码线程
     */
    private class VideoDecodeThread extends Thread {
        @Override
        public void run() {
            MediaExtractor videoExtractor = new MediaExtractor();
            MediaCodec videoCodec = null;
            try {
                videoExtractor.setDataSource(filePath);
            } catch (IOException e) {
                e.printStackTrace();
            }
            int videoTrackIndex;
            // 获取视频所在轨道
            videoTrackIndex = getTrackIndex(videoExtractor, "video/");
            if (videoTrackIndex >= 0) {
                MediaFormat mediaFormat = videoExtractor.getTrackFormat(videoTrackIndex);
                int width = mediaFormat.getInteger(MediaFormat.KEY_WIDTH);
                int height = mediaFormat.getInteger(MediaFormat.KEY_HEIGHT);
                float time = mediaFormat.getLong(MediaFormat.KEY_DURATION) / 1000000;
                if (mListener != null) {
                    mListener.videoAspect(width, height, time);
                }
                videoExtractor.selectTrack(videoTrackIndex);
                try {
                    videoCodec = MediaCodec.createDecoderByType(mediaFormat.getString(MediaFormat.KEY_MIME));
                    videoCodec.configure(mediaFormat, surface, null, 0);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

            if (videoCodec == null) {
                if (VERBOSE) {
                    Log.d(TAG, "video decoder is unexpectedly null");
                }
                return;
            }

            videoCodec.start();
            MediaCodec.BufferInfo videoBufferInfo = new MediaCodec.BufferInfo();
            ByteBuffer[] inputBuffers = videoCodec.getInputBuffers();
            boolean isVideoEOS = false;

            long startMs = System.currentTimeMillis();

            while (!Thread.interrupted() && !cancel) {
                if (isPlaying) {
                    // 暂停
                    if (isPause) {
                        continue;
                    }
                    // 将资源传递到解码器
                    if (!isVideoEOS) {
                        isVideoEOS = decodeMediaData(videoExtractor, videoCodec, inputBuffers);
                    }
                    // 获取解码后的数据
                    int outputBufferIndex = videoCodec.dequeueOutputBuffer(videoBufferInfo, TIMEOUT_US);
                    switch (outputBufferIndex) {
                        case MediaCodec.INFO_OUTPUT_FORMAT_CHANGED:
                            if (VERBOSE) {
                                Log.d(TAG, "INFO_OUTPUT_FORMAT_CHANGED");
                            }
                            break;
                        case MediaCodec.INFO_TRY_AGAIN_LATER:
                            if (VERBOSE) {
                                Log.d(TAG, "INFO_TRY_AGAIN_LATER");
                            }
                            break;
                        case MediaCodec.INFO_OUTPUT_BUFFERS_CHANGED:
                            if (VERBOSE) {
                                Log.d(TAG, "INFO_OUTPUT_BUFFERS_CHANGED");
                            }
                            break;
                        default:
                            // 延迟解码
                            decodeDelay(videoBufferInfo, startMs);
                            // 释放资源
                            videoCodec.releaseOutputBuffer(outputBufferIndex, true);
                            break;
                    }
                    // 结尾
                    if ((videoBufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
                        Log.v(TAG, "buffer stream end");
                        break;
                    }
                }
            }
            // 释放解码器
            videoCodec.stop();
            videoCodec.release();
            videoExtractor.release();
        }
    }
```
其中，解复用的方法如下：
```
/**
     * 解复用，得到需要解码的数据
     * @param extractor
     * @param decoder
     * @param inputBuffers
     * @return
     */
    private static boolean decodeMediaData(MediaExtractor extractor, MediaCodec decoder, ByteBuffer[] inputBuffers) {
        boolean isMediaEOS = false;
        int inputBufferIndex = decoder.dequeueInputBuffer(TIMEOUT_US);
        if (inputBufferIndex >= 0) {
            ByteBuffer inputBuffer = inputBuffers[inputBufferIndex];
            int sampleSize = extractor.readSampleData(inputBuffer, 0);
            if (sampleSize < 0) {
                decoder.queueInputBuffer(inputBufferIndex, 0, 0, 0, MediaCodec.BUFFER_FLAG_END_OF_STREAM);
                isMediaEOS = true;
                if (VERBOSE) {
                    Log.d(TAG, "end of stream");
                }
            } else {
                decoder.queueInputBuffer(inputBufferIndex, 0, sampleSize, extractor.getSampleTime(), 0);
                extractor.advance();
            }
        }
        return isMediaEOS;
    }
```
解码延时实现如下：
```
/**
     * 解码延时
     * @param bufferInfo
     * @param startMillis
     */
    private void decodeDelay(MediaCodec.BufferInfo bufferInfo, long startMillis) {
        while (bufferInfo.presentationTimeUs / 1000 > System.currentTimeMillis() - startMillis) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
                break;
            }
        }
    }
```

同样，音频解码线程跟视频解码线程大体类似。这里不做详细介绍，直接上代码：
```
/**
     * 音频解码线程
     */
    private class AudioDecodeThread extends Thread {
        private int mInputBufferSize;
        private AudioTrack audioTrack;

        @Override
        public void run() {
            MediaExtractor audioExtractor = new MediaExtractor();
            MediaCodec audioCodec = null;
            try {
                audioExtractor.setDataSource(filePath);
            } catch (IOException e) {
                e.printStackTrace();
            }
            for (int i = 0; i < audioExtractor.getTrackCount(); i++) {
                MediaFormat mediaFormat = audioExtractor.getTrackFormat(i);
                String mime = mediaFormat.getString(MediaFormat.KEY_MIME);
                if (mime.startsWith("audio/")) {
                    audioExtractor.selectTrack(i);
                    int audioChannels = mediaFormat.getInteger(MediaFormat.KEY_CHANNEL_COUNT);
                    int audioSampleRate = mediaFormat.getInteger(MediaFormat.KEY_SAMPLE_RATE);
                    int minBufferSize = AudioTrack.getMinBufferSize(audioSampleRate,
                            (audioChannels == 1 ? AudioFormat.CHANNEL_OUT_MONO : AudioFormat.CHANNEL_OUT_STEREO),
                            AudioFormat.ENCODING_PCM_16BIT);
                    int maxInputSize = mediaFormat.getInteger(MediaFormat.KEY_MAX_INPUT_SIZE);
                    mInputBufferSize = minBufferSize > 0 ? minBufferSize * 4 : maxInputSize;
                    int frameSizeInBytes = audioChannels * 2;
                    mInputBufferSize = (mInputBufferSize / frameSizeInBytes) * frameSizeInBytes;
                    audioTrack = new AudioTrack(AudioManager.STREAM_MUSIC,
                            audioSampleRate,
                            (audioChannels == 1 ? AudioFormat.CHANNEL_OUT_MONO : AudioFormat.CHANNEL_OUT_STEREO),
                            AudioFormat.ENCODING_PCM_16BIT,
                            mInputBufferSize,
                            AudioTrack.MODE_STREAM);
                    audioTrack.play();
                    try {
                        audioCodec = MediaCodec.createDecoderByType(mime);
                        audioCodec.configure(mediaFormat, null, null, 0);
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                    break;
                }
            }

            if (audioCodec == null) {
                if (VERBOSE) {
                    Log.d(TAG, "audio decoder is unexpectedly null");
                }
                return;
            }
            audioCodec.start();
            final ByteBuffer[] buffers = audioCodec.getOutputBuffers();
            int sz = buffers[0].capacity();
            if (sz <= 0) {
                sz = mInputBufferSize;
            }
            byte[] mAudioOutTempBuf = new byte[sz];

            MediaCodec.BufferInfo audioBufferInfo = new MediaCodec.BufferInfo();
            ByteBuffer[] inputBuffers = audioCodec.getInputBuffers();
            ByteBuffer[] outputBuffers = audioCodec.getOutputBuffers();
            boolean isAudioEOS = false;
            long startMs = System.currentTimeMillis();
            while (!Thread.interrupted() && !cancel) {
                if (isPlaying) {
                    // 暂停
                    if (isPause) {
                        continue;
                    }
                    // 解码
                    if (!isAudioEOS) {
                        isAudioEOS = decodeMediaData(audioExtractor, audioCodec, inputBuffers);
                    }
                    // 获取解码后的数据
                    int outputBufferIndex = audioCodec.dequeueOutputBuffer(audioBufferInfo, TIMEOUT_US);
                    switch (outputBufferIndex) {
                        case MediaCodec.INFO_OUTPUT_FORMAT_CHANGED:
                            if (VERBOSE) {
                                Log.d(TAG, "INFO_OUTPUT_FORMAT_CHANGED");
                            }
                            break;
                        case MediaCodec.INFO_TRY_AGAIN_LATER:
                            if (VERBOSE) {
                                Log.d(TAG, "INFO_TRY_AGAIN_LATER");
                            }
                            break;
                        case MediaCodec.INFO_OUTPUT_BUFFERS_CHANGED:
                            outputBuffers = audioCodec.getOutputBuffers();
                            if (VERBOSE) {
                                Log.d(TAG, "INFO_OUTPUT_BUFFERS_CHANGED");
                            }
                            break;
                        default:
                            ByteBuffer outputBuffer = outputBuffers[outputBufferIndex];
                            // 延时解码，跟视频时间同步
                            decodeDelay(audioBufferInfo, startMs);
                            // 如果解码成功，则将解码后的音频PCM数据用AudioTrack播放出来
                            if (audioBufferInfo.size > 0) {
                                if (mAudioOutTempBuf.length < audioBufferInfo.size) {
                                    mAudioOutTempBuf = new byte[audioBufferInfo.size];
                                }
                                outputBuffer.position(0);
                                outputBuffer.get(mAudioOutTempBuf, 0, audioBufferInfo.size);
                                outputBuffer.clear();
                                if (audioTrack != null)
                                    audioTrack.write(mAudioOutTempBuf, 0, audioBufferInfo.size);
                            }
                            // 释放资源
                            audioCodec.releaseOutputBuffer(outputBufferIndex, false);
                            break;
                    }

                    // 结尾了
                    if ((audioBufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
                        if (VERBOSE) {
                            Log.d(TAG, "BUFFER_FLAG_END_OF_STREAM");
                        }
                        break;
                    }
                }
            }

            // 释放MediaCode 和AudioTrack
            audioCodec.stop();
            audioCodec.release();
            audioExtractor.release();
            audioTrack.stop();
            audioTrack.release();
        }

    }
```
至此，我们就将简易播放器的核心功能实现了，完整的实现代码如下：
```
public class SimplePlayer {

    private static final String TAG = "Player";
    private static final boolean VERBOSE = false;
    private static final long TIMEOUT_US = 10000;

    private IPlayStateListener mListener;
    private VideoDecodeThread mVideoDecodeThread;
    private AudioDecodeThread mAudioDecodeThread;
    private boolean isPlaying;
    private boolean isPause;
    private String filePath;
    private Surface surface;

    // 是否取消播放线程
    private boolean cancel = false;

    public SimplePlayer(Surface surface, String filePath) {
        this.surface = surface;
        this.filePath = filePath;
        isPlaying = false;
        isPause = false;
    }

    /**
     * 设置回调
     * @param mListener
     */
    public void setPlayStateListener(IPlayStateListener mListener) {
        this.mListener = mListener;
    }

    /**
     * 是否处于播放状态
     * @return
     */
    public boolean isPlaying() {
        return isPlaying && !isPause;
    }

    /**
     * 开始播放
     */
    public void play() {
        isPlaying = true;
        if (mVideoDecodeThread == null) {
            mVideoDecodeThread = new VideoDecodeThread();
            mVideoDecodeThread.start();
        }
        if (mAudioDecodeThread == null) {
            mAudioDecodeThread = new AudioDecodeThread();
            mAudioDecodeThread.start();
        }
    }

    /**
     * 暂停
     */
    public void pause() {
        isPause = true;
    }

    /**
     * 继续播放
     */
    public void continuePlay() {
        isPause = false;
    }

    /**
     * 停止播放
     */
    public void stop() {
        isPlaying = false;
    }

    /**
     * 销毁
     */
    public void destroy() {
        stop();
        if (mAudioDecodeThread != null) {
            mAudioDecodeThread.interrupt();
        }
        if (mVideoDecodeThread != null) {
            mVideoDecodeThread.interrupt();
        }
    }

    /**
     * 解复用，得到需要解码的数据
     * @param extractor
     * @param decoder
     * @param inputBuffers
     * @return
     */
    private static boolean decodeMediaData(MediaExtractor extractor, MediaCodec decoder, ByteBuffer[] inputBuffers) {
        boolean isMediaEOS = false;
        int inputBufferIndex = decoder.dequeueInputBuffer(TIMEOUT_US);
        if (inputBufferIndex >= 0) {
            ByteBuffer inputBuffer = inputBuffers[inputBufferIndex];
            int sampleSize = extractor.readSampleData(inputBuffer, 0);
            if (sampleSize < 0) {
                decoder.queueInputBuffer(inputBufferIndex, 0, 0, 0, MediaCodec.BUFFER_FLAG_END_OF_STREAM);
                isMediaEOS = true;
                if (VERBOSE) {
                    Log.d(TAG, "end of stream");
                }
            } else {
                decoder.queueInputBuffer(inputBufferIndex, 0, sampleSize, extractor.getSampleTime(), 0);
                extractor.advance();
            }
        }
        return isMediaEOS;
    }

    /**
     * 解码延时
     * @param bufferInfo
     * @param startMillis
     */
    private void decodeDelay(MediaCodec.BufferInfo bufferInfo, long startMillis) {
        while (bufferInfo.presentationTimeUs / 1000 > System.currentTimeMillis() - startMillis) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
                break;
            }
        }
    }

    /**
     * 获取媒体类型的轨道
     * @param extractor
     * @param mediaType
     * @return
     */
    private static int getTrackIndex(MediaExtractor extractor, String mediaType) {
        int trackIndex = -1;
        for (int i = 0; i < extractor.getTrackCount(); i++) {
            MediaFormat mediaFormat = extractor.getTrackFormat(i);
            String mime = mediaFormat.getString(MediaFormat.KEY_MIME);
            if (mime.startsWith(mediaType)) {
                trackIndex = i;
                break;
            }
        }
        return trackIndex;
    }

    /**
     * 视频解码线程
     */
    private class VideoDecodeThread extends Thread {
        @Override
        public void run() {
            MediaExtractor videoExtractor = new MediaExtractor();
            MediaCodec videoCodec = null;
            try {
                videoExtractor.setDataSource(filePath);
            } catch (IOException e) {
                e.printStackTrace();
            }
            int videoTrackIndex;
            // 获取视频所在轨道
            videoTrackIndex = getTrackIndex(videoExtractor, "video/");
            if (videoTrackIndex >= 0) {
                MediaFormat mediaFormat = videoExtractor.getTrackFormat(videoTrackIndex);
                int width = mediaFormat.getInteger(MediaFormat.KEY_WIDTH);
                int height = mediaFormat.getInteger(MediaFormat.KEY_HEIGHT);
                float time = mediaFormat.getLong(MediaFormat.KEY_DURATION) / 1000000;
                if (mListener != null) {
                    mListener.videoAspect(width, height, time);
                }
                videoExtractor.selectTrack(videoTrackIndex);
                try {
                    videoCodec = MediaCodec.createDecoderByType(mediaFormat.getString(MediaFormat.KEY_MIME));
                    videoCodec.configure(mediaFormat, surface, null, 0);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

            if (videoCodec == null) {
                if (VERBOSE) {
                    Log.d(TAG, "video decoder is unexpectedly null");
                }
                return;
            }

            videoCodec.start();
            MediaCodec.BufferInfo videoBufferInfo = new MediaCodec.BufferInfo();
            ByteBuffer[] inputBuffers = videoCodec.getInputBuffers();
            boolean isVideoEOS = false;

            long startMs = System.currentTimeMillis();

            while (!Thread.interrupted() && !cancel) {
                if (isPlaying) {
                    // 暂停
                    if (isPause) {
                        continue;
                    }
                    // 将资源传递到解码器
                    if (!isVideoEOS) {
                        isVideoEOS = decodeMediaData(videoExtractor, videoCodec, inputBuffers);
                    }
                    // 获取解码后的数据
                    int outputBufferIndex = videoCodec.dequeueOutputBuffer(videoBufferInfo, TIMEOUT_US);
                    switch (outputBufferIndex) {
                        case MediaCodec.INFO_OUTPUT_FORMAT_CHANGED:
                            if (VERBOSE) {
                                Log.d(TAG, "INFO_OUTPUT_FORMAT_CHANGED");
                            }
                            break;
                        case MediaCodec.INFO_TRY_AGAIN_LATER:
                            if (VERBOSE) {
                                Log.d(TAG, "INFO_TRY_AGAIN_LATER");
                            }
                            break;
                        case MediaCodec.INFO_OUTPUT_BUFFERS_CHANGED:
                            if (VERBOSE) {
                                Log.d(TAG, "INFO_OUTPUT_BUFFERS_CHANGED");
                            }
                            break;
                        default:
                            // 延迟解码
                            decodeDelay(videoBufferInfo, startMs);
                            // 释放资源
                            videoCodec.releaseOutputBuffer(outputBufferIndex, true);
                            break;
                    }
                    // 结尾
                    if ((videoBufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
                        Log.v(TAG, "buffer stream end");
                        break;
                    }
                }
            }
            // 释放解码器
            videoCodec.stop();
            videoCodec.release();
            videoExtractor.release();
        }
    }

    /**
     * 音频解码线程
     */
    private class AudioDecodeThread extends Thread {
        private int mInputBufferSize;
        private AudioTrack audioTrack;

        @Override
        public void run() {
            MediaExtractor audioExtractor = new MediaExtractor();
            MediaCodec audioCodec = null;
            try {
                audioExtractor.setDataSource(filePath);
            } catch (IOException e) {
                e.printStackTrace();
            }
            for (int i = 0; i < audioExtractor.getTrackCount(); i++) {
                MediaFormat mediaFormat = audioExtractor.getTrackFormat(i);
                String mime = mediaFormat.getString(MediaFormat.KEY_MIME);
                if (mime.startsWith("audio/")) {
                    audioExtractor.selectTrack(i);
                    int audioChannels = mediaFormat.getInteger(MediaFormat.KEY_CHANNEL_COUNT);
                    int audioSampleRate = mediaFormat.getInteger(MediaFormat.KEY_SAMPLE_RATE);
                    int minBufferSize = AudioTrack.getMinBufferSize(audioSampleRate,
                            (audioChannels == 1 ? AudioFormat.CHANNEL_OUT_MONO : AudioFormat.CHANNEL_OUT_STEREO),
                            AudioFormat.ENCODING_PCM_16BIT);
                    int maxInputSize = mediaFormat.getInteger(MediaFormat.KEY_MAX_INPUT_SIZE);
                    mInputBufferSize = minBufferSize > 0 ? minBufferSize * 4 : maxInputSize;
                    int frameSizeInBytes = audioChannels * 2;
                    mInputBufferSize = (mInputBufferSize / frameSizeInBytes) * frameSizeInBytes;
                    audioTrack = new AudioTrack(AudioManager.STREAM_MUSIC,
                            audioSampleRate,
                            (audioChannels == 1 ? AudioFormat.CHANNEL_OUT_MONO : AudioFormat.CHANNEL_OUT_STEREO),
                            AudioFormat.ENCODING_PCM_16BIT,
                            mInputBufferSize,
                            AudioTrack.MODE_STREAM);
                    audioTrack.play();
                    try {
                        audioCodec = MediaCodec.createDecoderByType(mime);
                        audioCodec.configure(mediaFormat, null, null, 0);
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                    break;
                }
            }

            if (audioCodec == null) {
                if (VERBOSE) {
                    Log.d(TAG, "audio decoder is unexpectedly null");
                }
                return;
            }
            audioCodec.start();
            final ByteBuffer[] buffers = audioCodec.getOutputBuffers();
            int sz = buffers[0].capacity();
            if (sz <= 0) {
                sz = mInputBufferSize;
            }
            byte[] mAudioOutTempBuf = new byte[sz];

            MediaCodec.BufferInfo audioBufferInfo = new MediaCodec.BufferInfo();
            ByteBuffer[] inputBuffers = audioCodec.getInputBuffers();
            ByteBuffer[] outputBuffers = audioCodec.getOutputBuffers();
            boolean isAudioEOS = false;
            long startMs = System.currentTimeMillis();
            while (!Thread.interrupted() && !cancel) {
                if (isPlaying) {
                    // 暂停
                    if (isPause) {
                        continue;
                    }
                    // 解码
                    if (!isAudioEOS) {
                        isAudioEOS = decodeMediaData(audioExtractor, audioCodec, inputBuffers);
                    }
                    // 获取解码后的数据
                    int outputBufferIndex = audioCodec.dequeueOutputBuffer(audioBufferInfo, TIMEOUT_US);
                    switch (outputBufferIndex) {
                        case MediaCodec.INFO_OUTPUT_FORMAT_CHANGED:
                            if (VERBOSE) {
                                Log.d(TAG, "INFO_OUTPUT_FORMAT_CHANGED");
                            }
                            break;
                        case MediaCodec.INFO_TRY_AGAIN_LATER:
                            if (VERBOSE) {
                                Log.d(TAG, "INFO_TRY_AGAIN_LATER");
                            }
                            break;
                        case MediaCodec.INFO_OUTPUT_BUFFERS_CHANGED:
                            outputBuffers = audioCodec.getOutputBuffers();
                            if (VERBOSE) {
                                Log.d(TAG, "INFO_OUTPUT_BUFFERS_CHANGED");
                            }
                            break;
                        default:
                            ByteBuffer outputBuffer = outputBuffers[outputBufferIndex];
                            // 延时解码，跟视频时间同步
                            decodeDelay(audioBufferInfo, startMs);
                            // 如果解码成功，则将解码后的音频PCM数据用AudioTrack播放出来
                            if (audioBufferInfo.size > 0) {
                                if (mAudioOutTempBuf.length < audioBufferInfo.size) {
                                    mAudioOutTempBuf = new byte[audioBufferInfo.size];
                                }
                                outputBuffer.position(0);
                                outputBuffer.get(mAudioOutTempBuf, 0, audioBufferInfo.size);
                                outputBuffer.clear();
                                if (audioTrack != null)
                                    audioTrack.write(mAudioOutTempBuf, 0, audioBufferInfo.size);
                            }
                            // 释放资源
                            audioCodec.releaseOutputBuffer(outputBufferIndex, false);
                            break;
                    }

                    // 结尾了
                    if ((audioBufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
                        if (VERBOSE) {
                            Log.d(TAG, "BUFFER_FLAG_END_OF_STREAM");
                        }
                        break;
                    }
                }
            }

            // 释放MediaCode 和AudioTrack
            audioCodec.stop();
            audioCodec.release();
            audioExtractor.release();
            audioTrack.stop();
            audioTrack.release();
        }

    }
}
```
那么这样还存在什么问题呢？ 那就是同步和是否支持媒体流的问题了。上面的代码只是简单地获取本地视频文件，分别将视频帧解码显示和音频帧解码播放出来，还存在同步问题。同步无非就是追及过程，当视频帧播放快了，则等待音频帧播放完或者加快、舍弃音频帧，当音频播放快了，则判断是否需要加快视频帧的播放甚至舍弃视频帧。这里不同的同步方式，产生了几种不同的同步策略，分别是视频同步到音频、音频同步到视频、以外部时钟作为同步基准。详细的策略可以参考ffplay的源码。还有就是媒体流的支持问题，目前市面上的播放都是支持流媒体播放的，媒体流肯定要支持的，使用MediaCodec做播放器，在应对网络抖动方面还是比不上基于FFmpeg的软解码的播放器的。如果想要做商用的播放器，个人建议还是使用FFmpeg实现会好很多，可参考的资料也更多，遇到什么问题可以请教有经验的前辈，帮助我们少踩坑。
