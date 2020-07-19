前面一章我们讲解了解复用的实现流程，但并没有详细讲解解码器部分的处理，这一章我们将会介绍音频解码器以及视频解码器的实现。

### 准备解码器
准备解码器的流程一般如下：
1. 创建解码上下文
2. 从解复用上下文中复制参数到解码上下文
3. 根据解码上下文的id查找解码器，如果在播放器指定了实际的解码器名称，则需要根据指定的解码器名称查找解码器
4. 给解码上下文设置一些解码参数，比如lowres、refcounted_frames等解码参数
5. 打开解码器
6. 如果成功打开解码器，则根据类型创建解码器类，AudioDecoder或者是VideoDecoder。
7. 如果不成功，则需要释放解码上下文
准备解码器的完整代码如下：
```
int MediaPlayer::prepareDecoder(int streamIndex) {
    AVCodecContext *avctx;
    AVCodec *codec;
    AVDictionary *opts = NULL;
    AVDictionaryEntry *t = NULL;
    int ret = 0;
    const char *forcedCodecName = NULL;

    if (streamIndex < 0 || streamIndex >= pFormatCtx->nb_streams) {
        return -1;
    }

    // 创建解码上下文
    avctx = avcodec_alloc_context3(NULL);
    if (!avctx) {
        return AVERROR(ENOMEM);
    }

    do {
        // 复制解码上下文参数
        ret = avcodec_parameters_to_context(avctx, pFormatCtx->streams[streamIndex]->codecpar);
        if (ret < 0) {
            break;
        }

        // 设置时钟基准
        av_codec_set_pkt_timebase(avctx, pFormatCtx->streams[streamIndex]->time_base);

        // 查找解码器
        codec = avcodec_find_decoder(avctx->codec_id);
        // 指定解码器
        switch(avctx->codec_type) {
            case AVMEDIA_TYPE_AUDIO: {
                forcedCodecName = playerState->audioCodecName;
                break;
            }
            case AVMEDIA_TYPE_VIDEO: {
                forcedCodecName = playerState->videoCodecName;
                break;
            }
        }
        if (forcedCodecName) {
            codec = avcodec_find_decoder_by_name(forcedCodecName);
        }
        // 判断是否成功得到解码器
        if (!codec) {
            av_log(NULL, AV_LOG_WARNING,
                   "No codec could be found with id %d\n", avctx->codec_id);
            ret = AVERROR(EINVAL);
            break;
        }
        avctx->codec_id = codec->id;

        // 设置一些播放参数
        int stream_lowres = playerState->lowres;
        if (stream_lowres > av_codec_get_max_lowres(codec)) {
            av_log(avctx, AV_LOG_WARNING, "The maximum value for lowres supported by the decoder is %d\n",
                   av_codec_get_max_lowres(codec));
            stream_lowres = av_codec_get_max_lowres(codec);
        }
        av_codec_set_lowres(avctx, stream_lowres);
#if FF_API_EMU_EDGE
        if (stream_lowres) {
            avctx->flags |= CODEC_FLAG_EMU_EDGE;
        }
#endif
        if (playerState->fast) {
            avctx->flags2 |= AV_CODEC_FLAG2_FAST;
        }
#if FF_API_EMU_EDGE
        if (codec->capabilities & AV_CODEC_CAP_DR1) {
            avctx->flags |= CODEC_FLAG_EMU_EDGE;
        }
#endif
        opts = filterCodecOptions(playerState->codec_opts, avctx->codec_id, pFormatCtx, pFormatCtx->streams[streamIndex], codec);
        if (!av_dict_get(opts, "threads", NULL, 0)) {
            av_dict_set(&opts, "threads", "auto", 0);
        }

        if (stream_lowres) {
            av_dict_set_int(&opts, "lowres", stream_lowres, 0);
        }

        if (avctx->codec_type == AVMEDIA_TYPE_VIDEO || avctx->codec_type == AVMEDIA_TYPE_AUDIO) {
            av_dict_set(&opts, "refcounted_frames", "1", 0);
        }

        // 打开解码器
        if ((ret = avcodec_open2(avctx, codec, &opts)) < 0) {
            break;
        }
        if ((t = av_dict_get(opts, "", NULL, AV_DICT_IGNORE_SUFFIX))) {
            av_log(NULL, AV_LOG_ERROR, "Option %s not found.\n", t->key);
            ret =  AVERROR_OPTION_NOT_FOUND;
            break;
        }

        // 根据解码器类型创建解码器
        pFormatCtx->streams[streamIndex]->discard = AVDISCARD_DEFAULT;
        switch (avctx->codec_type) {
            case AVMEDIA_TYPE_AUDIO: {
                audioDecoder = new AudioDecoder(avctx, pFormatCtx->streams[streamIndex],
                                                streamIndex, playerState);
                break;
            }

            case AVMEDIA_TYPE_VIDEO: {
                videoDecoder = new VideoDecoder(pFormatCtx, avctx, pFormatCtx->streams[streamIndex],
                                                streamIndex, playerState);
                attachmentRequest = 1;
                break;
            }

            default:{
                break;
            }
        }
    } while (false);

    // 准备失败，则需要释放创建的解码上下文
    if (ret < 0) {
        if (playerCallback != NULL) {
            playerCallback->onError(0x01, "failed to open stream!");
        }
        avcodec_free_context(&avctx);
    }

    // 释放参数
    av_dict_free(&opts);

    return ret;
}
```

### 解码器类
目前播放器支持音频解码和视频解码，暂不支持字幕解码，可以根据需要自行实现。为了方便使用，我封装了AudioDecoder和VideoDecoder类。下面介绍解码器类的实现。

#### MediaDecoder 解码器基类
MediaDecoder基类主要封装一些基本的数据，比如媒体流、媒体流索引、待解码的数据包队列以及获取数据包队列的内存大小、队列缓冲的时长等数据。整个解码器基类的代码如下：
```
class MediaDecoder : public Runnable {
public:
    MediaDecoder(AVCodecContext *avctx, AVStream *stream, int streamIndex, PlayerState *playerState);

    virtual ~MediaDecoder();

    virtual void start();

    virtual void stop();

    virtual void flush();

    int pushPacket(AVPacket *pkt);

    int pushNullPacket();

    int getPacketSize();

    int getStreamIndex();

    AVStream *getStream();

    AVCodecContext *getCodecContext();

    int getMemorySize();

    int hasEnoughPackets();

    virtual void run();

protected:
    Mutex mMutex;
    Condition mCondition;
    PlayerState *playerState;
    PacketQueue *packetQueue;       // 数据包队列
    AVCodecContext *pCodecCtx;
    AVStream *pStream;
    int streamIndex;
};

MediaDecoder::MediaDecoder(AVCodecContext *avctx, AVStream *stream, int streamIndex, PlayerState *playerState) {
    packetQueue = new PacketQueue();
    this->pCodecCtx = avctx;
    this->pStream = stream;
    this->streamIndex = streamIndex;
    this->playerState = playerState;
}

MediaDecoder::~MediaDecoder() {
    ALOGI("MediaDecoder destructor");
    stop();
    if (packetQueue) {
        packetQueue->flush();
        delete packetQueue;
        packetQueue = NULL;
    }
    if (pCodecCtx) {
        avcodec_close(pCodecCtx);
        avcodec_free_context(&pCodecCtx);
        pCodecCtx = NULL;
    }
    playerState = NULL;
}

void MediaDecoder::start() {
    if (packetQueue) {
        packetQueue->start();
    }
}

void MediaDecoder::stop() {
    if (packetQueue) {
        packetQueue->abort();
    }
}

void MediaDecoder::flush() {
    if (packetQueue) {
        packetQueue->flush();
    }
}

int MediaDecoder::pushPacket(AVPacket *pkt) {
    if (packetQueue) {
        return packetQueue->pushPacket(pkt);
    }
}

int MediaDecoder::pushNullPacket() {
    if (packetQueue != NULL) {
        return packetQueue->pushNullPacket(streamIndex);
    }
    return -1;
}

int MediaDecoder::getPacketSize() {
    return packetQueue ? packetQueue->getPacketSize() : 0;
}

int MediaDecoder::getStreamIndex() {
    return streamIndex;
}

AVStream *MediaDecoder::getStream() {
    return pStream;
}

AVCodecContext *MediaDecoder::getCodecContext() {
    return pCodecCtx;
}

int MediaDecoder::getMemorySize() {
    return packetQueue ? packetQueue->getSize() : 0;
}

int MediaDecoder::hasEnoughPackets() {
    Mutex::Autolock lock(mMutex);
    return (packetQueue == NULL) || (packetQueue->isAbort())
           || (pStream->disposition & AV_DISPOSITION_ATTACHED_PIC)
           || (packetQueue->getPacketSize() > MIN_FRAMES)
              && (!packetQueue->getDuration()
                  || av_q2d(pStream->time_base) * packetQueue->getDuration() > 1.0);
}

void MediaDecoder::run() {
    // do nothing
}

```

MediaDecoder 解码基类用到了PacketQueue，PacketQueue是一个存放数据包的队列，其代码如下：
```
typedef struct PacketList {
    AVPacket pkt;
    struct PacketList *next;
} PacketList;

/**
 * 备注：这里不用std::queue是为了方便计算队列占用内存和队列的时长，在解码的时候要用到
 */
class PacketQueue {
public:
    PacketQueue();

    virtual ~PacketQueue();

    // 入队数据包
    int pushPacket(AVPacket *pkt);

    // 入队空数据包
    int pushNullPacket(int stream_index);

    // 刷新
    void flush();

    // 终止
    void abort();

    // 开始
    void start();

    // 获取数据包
    int getPacket(AVPacket *pkt);

    // 获取数据包
    int getPacket(AVPacket *pkt, int block);

    int getPacketSize();

    int getSize();

    int64_t getDuration();

    int isAbort();

private:
    int put(AVPacket *pkt);

private:
    Mutex mMutex;
    Condition mCondition;
    PacketList *first_pkt, *last_pkt;
    int nb_packets;
    int size;
    int64_t duration;
    int abort_request;
};

PacketQueue::PacketQueue() {
    abort_request = 0;
    first_pkt = NULL;
    last_pkt = NULL;
    nb_packets = 0;
    size = 0;
    duration = 0;
}

PacketQueue::~PacketQueue() {
    abort();
    flush();
}

/**
 * 入队数据包
 * @param pkt
 * @return
 */
int PacketQueue::put(AVPacket *pkt) {
    PacketList *pkt1;

    if (abort_request) {
        return -1;
    }

    pkt1 = (PacketList *) av_malloc(sizeof(PacketList));
    if (!pkt1) {
        return -1;
    }
    pkt1->pkt = *pkt;
    pkt1->next = NULL;

    if (!last_pkt) {
        first_pkt = pkt1;
    } else {
        last_pkt->next = pkt1;
    }
    last_pkt = pkt1;
    nb_packets++;
    size += pkt1->pkt.size + sizeof(*pkt1);
    duration += pkt1->pkt.duration;
    return 0;
}

/**
 * 入队数据包
 * @param pkt
 * @return
 */
int PacketQueue::pushPacket(AVPacket *pkt) {
    int ret;
    mMutex.lock();
    ret = put(pkt);
    mCondition.signal();
    mMutex.unlock();

    if (ret < 0) {
        av_packet_unref(pkt);
    }

    return ret;
}

int PacketQueue::pushNullPacket(int stream_index) {
    AVPacket pkt1, *pkt = &pkt1;
    av_init_packet(pkt);
    pkt->data = NULL;
    pkt->size = 0;
    pkt->stream_index = stream_index;
    return pushPacket(pkt);
}

/**
 * 刷新数据包
 */
void PacketQueue::flush() {
    PacketList *pkt, *pkt1;

    mMutex.lock();
    for (pkt = first_pkt; pkt; pkt = pkt1) {
        pkt1 = pkt->next;
        av_packet_unref(&pkt->pkt);
        av_freep(&pkt);
    }
    last_pkt = NULL;
    first_pkt = NULL;
    nb_packets = 0;
    size = 0;
    duration = 0;
    mCondition.signal();
    mMutex.unlock();
}

/**
 * 队列终止
 */
void PacketQueue::abort() {
    mMutex.lock();
    abort_request = 1;
    mCondition.signal();
    mMutex.unlock();
}

/**
 * 队列开始
 */
void PacketQueue::start() {
    mMutex.lock();
    abort_request = 0;
    mCondition.signal();
    mMutex.unlock();
}

/**
 * 取出数据包
 * @param pkt
 * @return
 */
int PacketQueue::getPacket(AVPacket *pkt) {
    return getPacket(pkt, 1);
}

/**
 * 取出数据包
 * @param pkt
 * @param block
 * @return
 */
int PacketQueue::getPacket(AVPacket *pkt, int block) {
    PacketList *pkt1;
    int ret;

    mMutex.lock();
    for (;;) {
        if (abort_request) {
            ret = -1;
            break;
        }

        pkt1 = first_pkt;
        if (pkt1) {
            first_pkt = pkt1->next;
            if (!first_pkt) {
                last_pkt = NULL;
            }
            nb_packets--;
            size -= pkt1->pkt.size + sizeof(*pkt1);
            duration -= pkt1->pkt.duration;
            *pkt = pkt1->pkt;
            av_free(pkt1);
            ret = 1;
            break;
        } else if (!block) {
            ret = 0;
            break;
        } else {
            mCondition.wait(mMutex);
        }
    }
    mMutex.unlock();
    return ret;
}

int PacketQueue::getPacketSize() {
    Mutex::Autolock lock(mMutex);
    return nb_packets;
}

int PacketQueue::getSize() {
    return size;
}

int64_t PacketQueue::getDuration() {
    return duration;
}

int PacketQueue::isAbort() {
    return abort_request;
}
```
数据包队列并没有使用std::queue，这是为了方便得到缓冲的内存大小以及缓冲的数据包时长。由于每个数据包(AVPacket)的数据大小不一致，时长也不一致，如果缓冲的数据包过小，则会导致播放卡顿等现象，比如后台播放视频时，如果缓冲的时长过短，则OpenSLES播放音频可能会出现卡顿、杂音等现象。

#### AudioDecoder 音频解码器类
音频解码器类AudioDecoder 继承于MediaDecoder，并且封装了解码方法decodeAudio(Frame *af)。该方法主要的作用在于将数据包解码得到AVFrame之后，将数据复制到Frame结构体中，并计算帧的pts等数据。整个AudioDecoder的代码如下：
```
class AudioDecoder : public MediaDecoder {
public:
    AudioDecoder(AVCodecContext *avctx, AVStream *stream, int streamIndex, PlayerState *playerState);

    virtual ~AudioDecoder();

    int getAudioFrame(AVFrame *frame);

private:
    AVPacket *packet;
    int64_t next_pts;
    AVRational next_pts_tb;
};

AudioDecoder::AudioDecoder(AVCodecContext *avctx, AVStream *stream, int streamIndex, PlayerState *playerState)
        : MediaDecoder(avctx, stream, streamIndex, playerState) {
    packet = av_packet_alloc();
}

AudioDecoder::~AudioDecoder() {
    stop();
    if (packet) {
        av_packet_free(&packet);
        av_freep(&packet);
        packet = NULL;
    }
    ALOGI("AudioDecoder destructor");
}

int AudioDecoder::getAudioFrame(AVFrame *frame) {
    int got_frame = 0;
    int ret = 0;

    if (!frame) {
        return AVERROR(ENOMEM);
    }
    av_frame_unref(frame);

    do {

        if (packetQueue->getPacket(packet) < 0) {
            return -1;
        }

        ret = avcodec_send_packet(pCodecCtx, packet);
        if (ret < 0 && ret != AVERROR(EAGAIN) && ret != AVERROR_EOF) {
            av_packet_unref(packet);
            continue;
        }
        ret = avcodec_receive_frame(pCodecCtx, frame);
        // 释放数据包的引用，防止内存泄漏
        av_packet_unref(packet);
        if (ret < 0) {
            av_frame_unref(frame);
            got_frame = 0;
            continue;
        } else {
            got_frame = 1;
            // 这里要重新计算frame的pts 否则会导致网络视频出现pts 对不上的情况
            AVRational tb = (AVRational){1, frame->sample_rate};
            if (frame->pts != AV_NOPTS_VALUE) {
                frame->pts = av_rescale_q(frame->pts, av_codec_get_pkt_timebase(pCodecCtx), tb);
            } else if (next_pts != AV_NOPTS_VALUE) {
                frame->pts = av_rescale_q(next_pts, next_pts_tb, tb);
            }
            if (frame->pts != AV_NOPTS_VALUE) {
                next_pts = frame->pts + frame->nb_samples;
                next_pts_tb = tb;
            }
        }
    } while (!got_frame);

    return got_frame;
}

```
音频解码器这里我们并不需要用一个帧队列缓存数据，因为我们默认是同步到音频时钟的，而音频流的解码重采样等处理比视频处理要快很多，因此，音频解码并不一定要一个音频帧队列缓存数据。尤其是对于本地音视频文件，就更不需要了，音频队列的数据都是连贯的。这里跟ffplay的流程就不同了，ijkplayer 和 ffplay在音频解码器都是另开一个解码线程并且将解码得到的数据存放到解码队列中，等待音频输出设备从解码队列取出音频帧做重采样输出处理。这里有个注意的地方，那就是解码得到的音频帧需要重新计算pts，对于网络视频流来说，解码得到的pts 有可能不正确，这样会导致音视频不同步的情况发生。

#### VideoDecoder 视频解码器类
视频解码器类继承与MediaDecoder，在调用开始解码方法后，打开一个解码线程，解码线程不断从待解码数据包队列中取出数据包解码，并且将解码得到的帧放入帧队列中。
```

class VideoDecoder : public MediaDecoder {
public:
    VideoDecoder(AVFormatContext *pFormatCtx, AVCodecContext *avctx,
                 AVStream *stream, int streamIndex, PlayerState *playerState);

    virtual ~VideoDecoder();

    void setMasterClock(MediaClock *masterClock);

    void start() override;

    void stop() override;

    void flush() override;

    int getFrameSize();

    FrameQueue *getFrameQueue();

    void run() override;

private:
    // 解码视频帧
    int decodeVideo();

private:
    AVFormatContext *pFormatCtx;    // 解复用上下文
    FrameQueue *frameQueue;         // 帧队列

    Thread *decodeThread;           // 解码线程
    MediaClock *masterClock;        // 主时钟
};

VideoDecoder::VideoDecoder(AVFormatContext *pFormatCtx, AVCodecContext *avctx,
                           AVStream *stream, int streamIndex, PlayerState *playerState)
        : MediaDecoder(avctx, stream, streamIndex, playerState) {
    this->pFormatCtx = pFormatCtx;
    frameQueue = new FrameQueue(VIDEO_QUEUE_SIZE, 1);
    decodeThread = NULL;
    masterClock = NULL;
}

VideoDecoder::~VideoDecoder() {
    ALOGI("VideoDecoder destructor");
    stop();
    mMutex.lock();
    pFormatCtx = NULL;
    if (frameQueue) {
        frameQueue->flush();
        delete frameQueue;
        frameQueue = NULL;
    }
    masterClock = NULL;
    mMutex.unlock();
}

void VideoDecoder::setMasterClock(MediaClock *masterClock) {
    Mutex::Autolock lock(mMutex);
    this->masterClock = masterClock;
}

void VideoDecoder::start() {
    MediaDecoder::start();
    if (frameQueue) {
        frameQueue->start();
    }
    if (!decodeThread) {
        decodeThread = new Thread(this);
        decodeThread->start();
    }
}

void VideoDecoder::stop() {
    MediaDecoder::stop();
    if (frameQueue) {
        frameQueue->abort();
    }
    if (decodeThread) {
        decodeThread->join();
        delete decodeThread;
        decodeThread = NULL;
    }
}

void VideoDecoder::flush() {
    MediaDecoder::flush();
    if (frameQueue) {
        frameQueue->flush();
    }
}

int VideoDecoder::getFrameSize() {
    Mutex::Autolock lock(mMutex);
    return frameQueue ? frameQueue->getFrameSize() : 0;
}

FrameQueue *VideoDecoder::getFrameQueue() {
    Mutex::Autolock lock(mMutex);
    return frameQueue;
}

void VideoDecoder::run() {
    decodeVideo();
}

/**
 * 解码视频数据包并放入帧队列
 * @return
 */
int VideoDecoder::decodeVideo() {
    AVFrame *frame = av_frame_alloc();
    Frame *vp;
    int got_picture;
    int ret = 0;

    AVRational tb = pStream->time_base;
    AVRational frame_rate = av_guess_frame_rate(pFormatCtx, pStream, NULL);

    if (!frame) {
        return AVERROR(ENOMEM);
    }

    AVPacket *packet = av_packet_alloc();
    if (!packet) {
        return AVERROR(ENOMEM);
    }

    for (;;) {

        if (playerState->abortRequest) {
            ret = -1;
            break;
        }

        if (packetQueue->getPacket(packet) < 0) {
            return -1;
        }

        // 送去解码
        ret = avcodec_send_packet(pCodecCtx, packet);
        if (ret < 0 && ret != AVERROR(EAGAIN) && ret != AVERROR_EOF) {
            av_packet_unref(packet);
            continue;
        }

        // 得到解码帧
        ret = avcodec_receive_frame(pCodecCtx, frame);
        if (ret < 0 && ret != AVERROR_EOF) {
            av_frame_unref(frame);
            av_packet_unref(packet);
            continue;
        } else {
            got_picture = 1;
            // 丢帧处理
            if (masterClock != NULL) {
                double dpts = NAN;

                if (frame->pts != AV_NOPTS_VALUE) {
                    dpts = av_q2d(pStream->time_base) * frame->pts;
                }
                // 计算视频帧的长宽比
                frame->sample_aspect_ratio = av_guess_sample_aspect_ratio(pFormatCtx, pStream,
                                                                          frame);
                // 是否需要做舍帧操作
                if (playerState->frameDrop > 0 ||
                    (playerState->frameDrop > 0 && playerState->syncType != AV_SYNC_VIDEO)) {
                    if (frame->pts != AV_NOPTS_VALUE) {
                        double diff = dpts - masterClock->getClock();
                        if (!isnan(diff) && fabs(diff) < AV_NOSYNC_THRESHOLD &&
                            diff < 0 && packetQueue->getPacketSize() > 0) {
                            av_frame_unref(frame);
                            got_picture = 0;
                        }
                    }
                }
            }
        }

        if (got_picture) {

            // 取出帧
            if (!(vp = frameQueue->peekWritable())) {
                ret = -1;
                break;
            }

            // 复制参数
            vp->uploaded = 0;
            vp->width = frame->width;
            vp->height = frame->height;
            vp->format = frame->format;
            vp->pts = (frame->pts == AV_NOPTS_VALUE) ? NAN : frame->pts * av_q2d(tb);
            vp->duration = frame_rate.num && frame_rate.den
                           ? av_q2d((AVRational){frame_rate.den, frame_rate.num}) : 0;
            av_frame_move_ref(vp->frame, frame);

            // 入队帧
            frameQueue->pushFrame();
        }

        // 释放数据包和缓冲帧的引用，防止内存泄漏
        av_frame_unref(frame);
        av_packet_unref(packet);
    }

    av_frame_free(&frame);
    av_free(frame);
    frame = NULL;

    av_packet_free(&packet);
    av_free(packet);
    packet = NULL;

    ALOGD("video decode thread exit!");
    return ret;
}
```
视频解码器用到了视频帧队列FrameQueue，FrameQueue是一个用数组实现的环形队列，并在创建是指定最大的存放数据(小于某个指定值)。其实现代码如下：
```
#define FRAME_QUEUE_SIZE 10

typedef struct Frame {
    AVFrame *frame;
    AVSubtitle sub;
    double pts;           /* presentation timestamp for the frame */
    double duration;      /* estimated duration of the frame */
    int width;
    int height;
    int format;
    int uploaded;
} Frame;

class FrameQueue {

public:
    FrameQueue(int max_size, int keep_last);

    virtual ~FrameQueue();

    void start();

    void abort();

    Frame *currentFrame();

    Frame *nextFrame();

    Frame *lastFrame();

    Frame *peekWritable();

    void pushFrame();

    void popFrame();

    void flush();

    int getFrameSize();

    int getShowIndex() const;

private:
    void unrefFrame(Frame *vp);

private:
    Mutex mMutex;
    Condition mCondition;
    int abort_request;
    Frame queue[FRAME_QUEUE_SIZE];
    int rindex;
    int windex;
    int size;
    int max_size;
    int keep_last;
    int show_index;
};

FrameQueue::FrameQueue(int max_size, int keep_last) {
    memset(queue, 0, sizeof(Frame) * FRAME_QUEUE_SIZE);
    this->max_size = FFMIN(max_size, FRAME_QUEUE_SIZE);
    this->keep_last = (keep_last != 0);
    for (int i = 0; i < this->max_size; ++i) {
        queue[i].frame = av_frame_alloc();
    }
    abort_request = 1;
    rindex = 0;
    windex = 0;
    size = 0;
    show_index = 0;
}

FrameQueue::~FrameQueue() {
    for (int i = 0; i < max_size; ++i) {
        Frame *vp = &queue[i];
        unrefFrame(vp);
        av_frame_free(&vp->frame);
    }
}

void FrameQueue::start() {
    mMutex.lock();
    abort_request = 0;
    mCondition.signal();
    mMutex.unlock();
}

void FrameQueue::abort() {
    mMutex.lock();
    abort_request = 1;
    mCondition.signal();
    mMutex.unlock();
}

Frame *FrameQueue::currentFrame() {
    return &queue[(rindex + show_index) % max_size];
}

Frame *FrameQueue::nextFrame() {
    return &queue[(rindex + show_index + 1) % max_size];
}

Frame *FrameQueue::lastFrame() {
    return &queue[rindex];
}

Frame *FrameQueue::peekWritable() {
    mMutex.lock();
    while (size >= max_size && !abort_request) {
        mCondition.wait(mMutex);
    }
    mMutex.unlock();

    if (abort_request) {
        return NULL;
    }

    return &queue[windex];
}

void FrameQueue::pushFrame() {
    if (++windex == max_size) {
        windex = 0;
    }
    mMutex.lock();
    size++;
    mCondition.signal();
    mMutex.unlock();
}

void FrameQueue::popFrame() {
    if (keep_last && !show_index) {
        show_index = 1;
        return;
    }
    unrefFrame(&queue[rindex]);
    if (++rindex == max_size) {
        rindex = 0;
    }
    mMutex.lock();
    size--;
    mCondition.signal();
    mMutex.unlock();
}

void FrameQueue::flush() {
    while (getFrameSize() > 0) {
        popFrame();
    }
}

int FrameQueue::getFrameSize() {
    return size - show_index;
}

void FrameQueue::unrefFrame(Frame *vp) {
    av_frame_unref(vp->frame);
    avsubtitle_free(&vp->sub);
}

int FrameQueue::getShowIndex() const {
    return show_index;
}

```
至此，我们就创建好了音频解码器和视频解码器，以及封装了解码队列。
完整代码请参考本人的播放器项目：[CainPlayer](https://github.com/CainKernel/CainPlayer)
