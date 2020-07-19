这一章，我们来讲解播放器解复用(从文件中读取数据包)的流程。在讲解播放器的读数据包流程之前，我们先定义一个播放器状态结构体，用来记录播放器的各种状态。

### 播放器状态结构体
首先，我们定义一个结构体，用于记录播放器的开始、暂停、定位等各种状态标志，以及重置结构体的方法：
```
/**
 * 播放器状态结构体
 */
typedef struct PlayerState {

    AVDictionary *sws_dict;         // 视频转码option参数
    AVDictionary *swr_opts;         //
    AVDictionary *format_opts;      // 解复用option参数
    AVDictionary *codec_opts;       // 解码option参数
    AVDictionary *resample_opts;    // 重采样option参数

    const char *audioCodecName;     // 指定音频解码器名称
    const char *videoCodecName;     // 指定视频解码器名称

    int abortRequest;               // 退出标志
    int pauseRequest;               // 暂停标志
    SyncType syncType;              // 同步类型
    int64_t startTime;              // 播放起始位置
    int64_t duration;               // 播放时长
    int realTime;                   // 判断是否实时流
    int audioDisable;               // 是否禁止音频流
    int videoDisable;               // 是否禁止视频流
    int displayDisable;             // 是否禁止显示

    int fast;                       // 解码上下文的AV_CODEC_FLAG2_FAST标志
    int genpts;                     // 解码上下文的AVFMT_FLAG_GENPTS标志
    int lowres;                     // 解码上下文的lowres标志

    float playbackRate;             // 播放速度
    float playbackPitch;            // 播放音调

    int seekByBytes;                // 是否以字节定位
    int seekRequest;                // 定位请求
    int seekFlags;                  // 定位标志
    int64_t seekPos;                // 定位位置
    int64_t seekRel;                // 定位偏移

    int autoExit;                   // 是否自动退出
    int loop;                       // 循环播放
    int mute;                       // 静音播放
    int frameDrop;                  // 舍帧操作

} PlayerState;

/**
 * 重置播放器状态结构体
 * @param state
 */
inline void resetPlayerState(PlayerState *state) {

    av_opt_free(state);

    av_dict_free(&state->sws_dict);
    av_dict_free(&state->swr_opts);
    av_dict_free(&state->format_opts);
    av_dict_free(&state->codec_opts);
    av_dict_free(&state->resample_opts);
    av_dict_set(&state->sws_dict, "flags", "bicubic", 0);

    if (state->audioCodecName != NULL) {
        av_freep(&state->audioCodecName);
        state->audioCodecName = NULL;
    }
    if (state->videoCodecName != NULL) {
        av_freep(&state->videoCodecName);
        state->videoCodecName = NULL;
    }
    state->abortRequest = 1;
    state->pauseRequest = 0;
    state->seekByBytes = 0;
    state->syncType = AV_SYNC_AUDIO;
    state->startTime = AV_NOPTS_VALUE;
    state->duration = AV_NOPTS_VALUE;
    state->realTime = 0;
    state->audioDisable = 0;
    state->videoDisable = 0;
    state->displayDisable = 0;
    state->fast = 0;
    state->genpts = 0;
    state->lowres = 0;
    state->playbackRate = 1.0;
    state->playbackPitch = 1.0;
    state->seekRequest = 0;
    state->seekFlags = 0;
    state->seekPos = 0;
    state->seekRel = 0;
    state->seekRel = 0;
    state->autoExit = 0;
    state->loop = 0;
    state->frameDrop = 0;
}

```
播放器状态结构体的作用是用来给解复用、解码、同步等流程共享状态的，方便统一播放器的状态。

### 初始化以及解复用
我们在播放器调用prepare()时创建一个线程用来初始化解码器、打开音视频输出设备和音视频同步渲染线程等出来了，在准备完成后，等待播放器调用start()方法更新PlayerState的pauseRequest标志，进入读数据包流程。读数据包线程执行逻辑如下：
#### 初始化流程
1. 利用avformat_alloc_context()方法创建解复用上下文并设置解复用中断回调
2. 利用 avformat_open_input()方法打开url，url可以是本地文件，也可以使网络媒体流
3. 在成功打开文件之后，我们需要利用avformat_find_stream_info()方法查找媒体流信息
4. 如果开始播放的位置不是AV_NOPTS_VALUE，即从文件开头开始的话，需要先利用avformat_seek_file方法定位到播放的起始位置
5. 查找音频流、视频流索引，然后根据是否禁用音频流、视频流判断的设置，分别准备解码器对象
6. 当我们准备好解码器之后，通过媒体播放器回调播放器已经准备。
7. 判断音频解码器是否存在，通过openAudioDevice方法打开音频输出设备
8. 判断视频解码器是否存在，打开视频同步输出设备
其中，第7、第8步骤，我们需要根据实际情况，重新设定同步类型。同步有三种类型，同步到音频时钟、同步到视频时钟、同步到外部时钟。默认是同步到音频时钟，如果音频解码器不存在，则根据需求同步到视频时钟还是外部时钟。

#### 解复用流程
经过前面的初始化之后，我们就可以进入读数据包流程。读数据包流程如下：
1. 判断是否退出播放器
2. 判断暂停状态是否发生改变，设置解复用是否暂停还是播放 —— av_read_pause 和 av_read_play 
3. 处理定位状态。利用avformat_seek_file()方法定位到实际的位置，如果定位成功，我们需要清空音频解码器、视频解码器中待解码队列的数据。处理完前面的逻辑，我们需要更新外部时钟，并且更新视频同步刷新的时钟
4. 根据是否设置attachmentRequest标志，从视频流取出attached_pic的数据包
5. 判断解码器待解码队列的数据包如果大于某个数量，则等待解码器消耗
6. 读数据包
7. 判断数据包是否读取成功。如果没成功，则判断读取的结果是否AVERROR_EOF，即结尾标志。如果到了结尾，则入队一个空的数据包。如果读取出错，则直接退出读数据包流程。如果都不是，则判断解码器中的待解码数据包、待输出帧是否存在数据，如果都不存在数据，则判断是否跳转至起始位置还是判断是否自动退出，或者是继续下一轮读数据包流程。
8. 根据取得的数据包判断是否在播放范围内的数据包。如果在播放范围内，则根据数据包的媒体流索引判断是否入队数据包舍弃。
至此，解复用流程就完成了。
整个线程执行体的代码如下：
```
int MediaPlayer::readPackets() {
    int ret = 0;
    AVDictionaryEntry *t;
    AVDictionary **opts;
    int scan_all_pmts_set = 0;

    // 准备解码器
    mMutex.lock();
    do {
        // 创建解复用上下文
        pFormatCtx = avformat_alloc_context();
        if (!pFormatCtx) {
            av_log(NULL, AV_LOG_FATAL, "Could not allocate context.\n");
            ret = AVERROR(ENOMEM);
            break;
        }

        // 设置解复用中断回调
        pFormatCtx->interrupt_callback.callback = avformat_interrupt_cb;
        pFormatCtx->interrupt_callback.opaque = playerState;
        if (!av_dict_get(playerState->format_opts, "scan_all_pmts", NULL, AV_DICT_MATCH_CASE)) {
            av_dict_set(&playerState->format_opts, "scan_all_pmts", "1", AV_DICT_DONT_OVERWRITE);
            scan_all_pmts_set = 1;
        }

        // 设置rtmp/rtsp的超时值
        if (av_stristart(url, "rtmp", NULL) || av_stristart(url, "rtsp", NULL)) {
            // There is total different meaning for 'timeout' option in rtmp
            av_log(NULL, AV_LOG_WARNING, "remove 'timeout' option for rtmp.\n");
            av_dict_set(&playerState->format_opts, "timeout", NULL, 0);
        }

        // 打开文件
        ret = avformat_open_input(&pFormatCtx, url, NULL, &playerState->format_opts);
        if (ret < 0) {
            printError(url, ret);
            ret = -1;
            break;
        }

        if (scan_all_pmts_set) {
            av_dict_set(&playerState->format_opts, "scan_all_pmts", NULL, AV_DICT_MATCH_CASE);
        }

        if ((t = av_dict_get(playerState->format_opts, "", NULL, AV_DICT_IGNORE_SUFFIX))) {
            av_log(NULL, AV_LOG_ERROR, "Option %s not found.\n", t->key);
            ret = AVERROR_OPTION_NOT_FOUND;
            break;
        }

        if (playerState->genpts) {
            pFormatCtx->flags |= AVFMT_FLAG_GENPTS;
        }
        av_format_inject_global_side_data(pFormatCtx);

        opts = setupStreamInfoOptions(pFormatCtx, playerState->codec_opts);

        // 查找媒体流信息
        ret = avformat_find_stream_info(pFormatCtx, opts);
        if (opts != NULL) {
            for (int i = 0; i < pFormatCtx->nb_streams; i++) {
                if (opts[i] != NULL) {
                    av_dict_free(&opts[i]);
                }
            }
            av_freep(&opts);
        }

        if (ret < 0) {
            av_log(NULL, AV_LOG_WARNING,
                   "%s: could not find codec parameters\n", url);
            ret = -1;
            break;
        }

        // 文件时长(秒)
        if (pFormatCtx->duration != AV_NOPTS_VALUE) {
            mDuration = (int)(pFormatCtx->duration / AV_TIME_BASE);
        }

        if (pFormatCtx->pb) {
            pFormatCtx->pb->eof_reached = 0;
        }
        // 判断是否以字节方式定位
        playerState->seekByBytes = !!(pFormatCtx->iformat->flags & AVFMT_TS_DISCONT)
                                     && strcmp("ogg", pFormatCtx->iformat->name);

        // 设置最大帧间隔
        mediaSync->setMaxDuration((pFormatCtx->iformat->flags & AVFMT_TS_DISCONT) ? 10.0 : 3600.0);

        // 如果不是从头开始播放，则跳转到播放位置
        if (playerState->startTime != AV_NOPTS_VALUE) {
            int64_t timestamp;

            timestamp = playerState->startTime;
            if (pFormatCtx->start_time != AV_NOPTS_VALUE) {
                timestamp += pFormatCtx->start_time;
            }
            ret = avformat_seek_file(pFormatCtx, -1, INT64_MIN, timestamp, INT64_MAX, 0);
            if (ret < 0) {
                av_log(NULL, AV_LOG_WARNING, "%s: could not seek to position %0.3f\n",
                       url, (double)timestamp / AV_TIME_BASE);
            }
        }
        // 判断是否实时流，判断是否需要设置无限缓冲区
        playerState->realTime = isRealTime(pFormatCtx);
        if (playerState->infiniteBuffer < 0 && playerState->realTime) {
            playerState->infiniteBuffer = 1;
        }

        // 查找媒体流信息
        int audioIndex = -1;
        int videoIndex = -1;
        for (int i = 0; i < pFormatCtx->nb_streams; ++i) {
            if (pFormatCtx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_AUDIO) {
                if (audioIndex == -1) {
                    audioIndex = i;
                }
            } else if (pFormatCtx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {
                if (videoIndex == -1) {
                    videoIndex = i;
                }
            }
        }
        // 如果不禁止视频流，则查找最合适的视频流索引
        if (!playerState->videoDisable) {
            videoIndex = av_find_best_stream(pFormatCtx, AVMEDIA_TYPE_VIDEO,
                                             videoIndex, -1, NULL, 0);
        } else {
            videoIndex = -1;
        }
        // 如果不禁止音频流，则查找最合适的音频流索引(与视频流关联的音频流)
        if (!playerState->audioDisable) {
            audioIndex = av_find_best_stream(pFormatCtx, AVMEDIA_TYPE_AUDIO,
                                             audioIndex, videoIndex, NULL, 0);
        } else {
            audioIndex = -1;
        }

        // 如果音频流和视频流都没有找到，则直接退出
        if (audioIndex == -1 && videoIndex == -1) {
            av_log(NULL, AV_LOG_WARNING,
                   "%s: could not find audio and video stream\n", url);
            ret = -1;
            break;
        }

        // 根据媒体流索引准备解码器
        if (audioIndex >= 0) {
            prepareDecoder(audioIndex);
        }
        if (videoIndex >= 0) {
            prepareDecoder(videoIndex);
        }

        if (!audioDecoder && !videoDecoder) {
            av_log(NULL, AV_LOG_WARNING,
                   "failed to create audio and video decoder\n");
            ret = -1;
            break;
        }
        ret = 0;
    } while (false);
    mMutex.unlock();

    // 出错返回
    if (ret < 0) {
        if (playerCallback) {
            playerCallback->onError(0x01, "prepare decoder failed!");
        }
        return -1;
    }

    // 准备完成回调
    if (playerCallback) {
        playerCallback->onPrepared();
    }

    // 视频解码器开始解码
    if (videoDecoder != NULL) {
        videoDecoder->start();
    } else {
        if (playerState->syncType == AV_SYNC_VIDEO) {
            playerState->syncType = AV_SYNC_AUDIO;
        }
    }

    // 音频解码器开始解码
    if (audioDecoder != NULL) {
        audioDecoder->start();
    } else {
        if (playerState->syncType == AV_SYNC_AUDIO) {
            playerState->syncType = AV_SYNC_EXTERNAL;
        }
    }

    // 打开音频输出设备
    if (audioDecoder != NULL) {
        AVCodecContext *avctx = audioDecoder->getCodecContext();
        ret = openAudioDevice(avctx->channel_layout, avctx->channels,
                        avctx->sample_rate);
        if (ret < 0) {
            av_log(NULL, AV_LOG_WARNING, "could not open audio device\n");
            // 如果音频设备打开失败，则调整时钟的同步类型
            if (playerState->syncType == AV_SYNC_AUDIO) {
                if (videoDecoder != NULL) {
                    playerState->syncType = AV_SYNC_VIDEO;
                } else {
                    playerState->syncType = AV_SYNC_EXTERNAL;
                }
            }
        } else {
            // 启动音频输出设备
            audioDevice->start();
        }
    }

    if (videoDecoder) {
        if (playerState->syncType == AV_SYNC_AUDIO) {
            videoDecoder->setMasterClock(mediaSync->getAudioClock());
        } else if (playerState->syncType == AV_SYNC_VIDEO) {
            videoDecoder->setMasterClock(mediaSync->getVideoClock());
        } else {
            videoDecoder->setMasterClock(mediaSync->getExternalClock());
        }
    }

    // 开始同步
    mediaSync->start(videoDecoder, audioDecoder);

    // 读数据包流程
    eof = 0;
    ret = 0;
    AVPacket pkt1, *pkt = &pkt1;
    int64_t stream_start_time;
    int playInRange = 0;
    int64_t pkt_ts;
    for (;;) {

        // 退出播放器
        if (playerState->abortRequest) {
            break;
        }

        // 是否暂停
        if (playerState->pauseRequest != lastPaused) {
            lastPaused = playerState->pauseRequest;
            if (playerState->pauseRequest) {
                av_read_pause(pFormatCtx);
            } else {
                av_read_play(pFormatCtx);
            }
        }

#if CONFIG_RTSP_DEMUXER || CONFIG_MMSH_PROTOCOL
        if (playerState->pauseRequest &&
            (!strcmp(pFormatCtx->iformat->name, "rtsp") ||
             (pFormatCtx->pb && !strncmp(url, "mmsh:", 5)))) {
            continue;
        }
#endif
        // 定位处理
        if (playerState->seekRequest) {
            int64_t seek_target = playerState->seekPos;
            int64_t seek_min = playerState->seekRel > 0 ? seek_target - playerState->seekRel + 2: INT64_MIN;
            int64_t seek_max = playerState->seekRel < 0 ? seek_target - playerState->seekRel - 2: INT64_MAX;
            // 定位
            ret = avformat_seek_file(pFormatCtx, -1, seek_min, seek_target, seek_max, playerState->seekFlags);
            if (ret < 0) {
                av_log(NULL, AV_LOG_ERROR, "%s: error while seeking\n", url);
            } else {
                if (audioDecoder) {
                    audioDecoder->flush();
                }
                if (videoDecoder) {
                    videoDecoder->flush();
                }

                if (audioDevice) {
                    audioDevice->flush();
                }

                // 更新外部时钟值
                if (playerState->seekFlags & AVSEEK_FLAG_BYTE) {
                    mediaSync->updateExternalClock(NAN);
                } else {
                    mediaSync->updateExternalClock(seek_target / (double)AV_TIME_BASE);
                }
                mediaSync->refreshVideoTimer();
            }
            attachmentRequest = 1;
            playerState->seekRequest = 0;
            eof = 0;
        }

        // 取得封面数据包
        if (attachmentRequest) {
            if (videoDecoder && (videoDecoder->getStream()->disposition
                                 & AV_DISPOSITION_ATTACHED_PIC)) {
                AVPacket copy;
                if ((ret = av_copy_packet(&copy, &videoDecoder->getStream()->attached_pic)) < 0) {
                    break;
                }
                videoDecoder->pushPacket(&copy);
                videoDecoder->pushNullPacket();
            }
            attachmentRequest = 0;
        }

        // 如果队列中存在足够的数据包，则等待消耗
        // 备注：这里要等待一定时长的缓冲队列，要不然会导致OpenSLES播放音频出现卡顿等现象
        if (playerState->infiniteBuffer < 1 &&
            ((audioDecoder ? audioDecoder->getMemorySize() : 0) + (videoDecoder ? videoDecoder->getMemorySize() : 0) > MAX_QUEUE_SIZE
             || (!audioDecoder || audioDecoder->hasEnoughPackets()) && (!videoDecoder || videoDecoder->hasEnoughPackets()))) {
            continue;
        }

        // 读出数据包
        ret = av_read_frame(pFormatCtx, pkt);
        if (ret < 0) {
            // 如果没能读出裸数据包，判断是否是结尾
            if ((ret == AVERROR_EOF || avio_feof(pFormatCtx->pb)) && !eof) {
                if (videoDecoder != NULL) {
                    videoDecoder->pushNullPacket();
                }
                if (audioDecoder != NULL) {
                    audioDecoder->pushNullPacket();
                }
                eof = 1;
            }
            // 读取出错，则直接退出
            if (pFormatCtx->pb && pFormatCtx->pb->error) {
                ret = -1;
                break;
            }

            // 如果不处于暂停状态，并且队列中所有数据都没有，则判断是否需要
            if (!playerState->pauseRequest && (!audioDecoder || audioDecoder->getPacketSize() == 0)
                && (!videoDecoder || (videoDecoder->getPacketSize() == 0
                                      && videoDecoder->getFrameSize() == 0))) {
                if (playerState->loop) {
                    seekTo(playerState->startTime != AV_NOPTS_VALUE ? playerState->startTime : 0);
                } else if (playerState->autoExit) {
                    ret = AVERROR_EOF;
                    break;
                }
            }
            continue;
        } else {
            eof = 0;
        }

        // 计算pkt的pts是否处于播放范围内
        stream_start_time = pFormatCtx->streams[pkt->stream_index]->start_time;
        pkt_ts = pkt->pts == AV_NOPTS_VALUE ? pkt->dts : pkt->pts;
        // 播放范围
        playInRange = playerState->duration == AV_NOPTS_VALUE
                      || (pkt_ts - (stream_start_time != AV_NOPTS_VALUE ? stream_start_time : 0)) *
                         av_q2d(pFormatCtx->streams[pkt->stream_index]->time_base)
                         - (double)(playerState->startTime != AV_NOPTS_VALUE ? playerState->startTime : 0) / 1000000
                         <= ((double)playerState->duration / 1000000);
        if (playInRange && audioDecoder && pkt->stream_index == audioDecoder->getStreamIndex()) {
            audioDecoder->pushPacket(pkt);
        } else if (playInRange && videoDecoder && pkt->stream_index == videoDecoder->getStreamIndex()) {
            videoDecoder->pushPacket(pkt);
        } else {
            av_packet_unref(pkt);
        }
    }

    if (ret < 0) {
        if (playerCallback) {
            playerCallback->onError(0x02, "error when reading packets!");
        }
    } else { // 播放完成
        if (playerCallback) {
            playerCallback->onComplete();
        }
    }

    ALOGD("read packets thread exit!");
    return ret;
}
```
完整代码请参考本人的播放器项目：[CainPlayer](https://github.com/CainKernel/CainPlayer)
