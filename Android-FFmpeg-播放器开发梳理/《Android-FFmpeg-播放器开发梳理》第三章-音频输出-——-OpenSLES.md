前面一章，我们讲解了音频解码器和视频解码器的封装和实现。这一章我们将会讲解音频输出部分的处理。

#### 打开音频设备
前面在讲解播放器初始化以及解复用流程时，讲到在初始化准备好解码器之后，需要打开音频输出设备，但跳过了没有讲解。这里讲解一下过程。打开音频设备主要是通过AudioDeviceSpec结构体根据期望的音频channel layout、nb_samples、sample_rate等参数，传递给音频设备，通过音频设备对象的open方法打开音频输出设备，根据结果判断是否打开成功。如果成功，则获取音频输出设备的硬件参数、音频PCM缓冲区大小等数据。
打开音频设备代码如下：
```
int MediaPlayer::openAudioDevice(int64_t wanted_channel_layout, int wanted_nb_channels,
                                 int wanted_sample_rate) {
    AudioDeviceSpec wanted_spec, spec;
    const int next_nb_channels[] = {0, 0, 1, 6, 2, 6, 4, 6};
    const int next_sample_rates[] = {44100, 48000};
    int next_sample_rate_idx = FF_ARRAY_ELEMS(next_sample_rates) - 1;
    if (wanted_nb_channels != av_get_channel_layout_nb_channels(wanted_channel_layout)
        || !wanted_channel_layout) {
        wanted_channel_layout = av_get_default_channel_layout(wanted_nb_channels);
        wanted_channel_layout &= ~AV_CH_LAYOUT_STEREO_DOWNMIX;
    }
    wanted_nb_channels = av_get_channel_layout_nb_channels(wanted_channel_layout);
    wanted_spec.channels = wanted_nb_channels;
    wanted_spec.freq = wanted_sample_rate;
    if (wanted_spec.freq <= 0 || wanted_spec.channels <= 0) {
        av_log(NULL, AV_LOG_ERROR, "Invalid sample rate or channel count!\n");
        return -1;
    }
    while (next_sample_rate_idx && next_sample_rates[next_sample_rate_idx] >= wanted_spec.freq) {
        next_sample_rate_idx--;
    }

    wanted_spec.format = AV_SAMPLE_FMT_S16;
    wanted_spec.samples = FFMAX(AUDIO_MIN_BUFFER_SIZE,
                                2 << av_log2(wanted_spec.freq / AUDIO_MAX_CALLBACKS_PER_SEC));
    wanted_spec.callback = audioPCMQueueCallback;
    wanted_spec.userdata = this;

    // 打开音频设备
    while (audioDevice->open(&wanted_spec, &spec) < 0) {
        av_log(NULL, AV_LOG_WARNING, "Failed to open audio device: (%d channels, %d Hz)!\n",
               wanted_spec.channels, wanted_spec.freq);
        wanted_spec.channels = next_nb_channels[FFMIN(7, wanted_spec.channels)];
        if (!wanted_spec.channels) {
            wanted_spec.freq = next_sample_rates[next_sample_rate_idx--];
            wanted_spec.channels = wanted_nb_channels;
            if (!wanted_spec.freq) {
                av_log(NULL, AV_LOG_ERROR, "No more combinations to try, audio open failed\n");
                return -1;
            }
        }
        wanted_channel_layout = av_get_default_channel_layout(wanted_spec.channels);
    }

    if (spec.format != AV_SAMPLE_FMT_S16) {
        av_log(NULL, AV_LOG_ERROR, "audio format %d is not supported!\n", spec.format);
        return -1;
    }

    if (spec.channels != wanted_spec.channels) {
        wanted_channel_layout = av_get_default_channel_layout(spec.channels);
        if (!wanted_channel_layout) {
            av_log(NULL, AV_LOG_ERROR, "channel count %d is not supported!\n", spec.channels);
            return -1;
        }
    }

    // 初始化音频重采样器
    if (!audioResampler) {
        audioResampler = new AudioResampler(playerState, audioDecoder, mediaSync);
    }
    // 设置需要重采样的参数
    audioResampler->setResampleParams(&spec, wanted_channel_layout);

    return spec.size;
}
```


#### 音频设备对象
前面调用的AudioDevice类是一个音频输出基类，其代码如下：
```
// 音频PCM填充回调
typedef void (*AudioPCMCallback) (void *userdata, uint8_t *stream, int len);

typedef struct AudioDeviceSpec {
    int freq;                   // 采样率
    AVSampleFormat format;      // 音频采样格式
    uint8_t channels;           // 声道
    uint16_t samples;           // 采样大小
    uint32_t size;              // 缓冲区大小
    AudioPCMCallback callback;  // 音频回调
    void *userdata;             // 音频上下文
} AudioDeviceSpec;

class AudioDevice {
public:
    AudioDevice();

    virtual ~AudioDevice();

    virtual int open(const AudioDeviceSpec *desired, AudioDeviceSpec *obtained);

    virtual void start();

    virtual void stop();

    virtual void pause();

    virtual void resume();

    virtual void flush();

    virtual void setStereoVolume(float left_volume, float right_volume);
};

AudioDevice::AudioDevice() {

}

AudioDevice::~AudioDevice() {

}

int AudioDevice::open(const AudioDeviceSpec *desired, AudioDeviceSpec *obtained) {
    return 0;
}

void AudioDevice::start() {

}

void AudioDevice::stop() {

}

void AudioDevice::pause() {

}

void AudioDevice::resume() {

}

void AudioDevice::flush() {

}

void AudioDevice::setStereoVolume(float left_volume, float right_volume) {

}
```
其中AudioDeviceSpec 结构体用于存放音频设备的帧率、音频采样格式、声道数、采样大小、缓冲区大小、PCM填充回调方法等等参数，用于给播放器核心与音频输出设备进行交互。基类不做任何逻辑处理，我们接下来利用OpenSLES封装音频输出设备。

#### OpenSLES 音频输出
Android 环境下音频播放通常有两种方式—— AudioTrack 和 OpenSLES。AudioTrack 本身是Java实现，如果要使用的话，需要通过JNI Call的方式调用，实现起来也比较简单，这里就不做介绍了，有兴趣的可以自行实现。本项目采用OpenSLES 播放音频。
我们继承前面的AudioDevice基类，封装OpenSLES 音频输出设备。
OpenSLES 在open方法时初始化，并根据传进来的数据计算出需要的SL采样率、channel layout 等数据。并且在调用start方法时，启用一个音频输出处理线程。由于slBufferQueueItf设定了缓冲区大小，这里是设置了4个缓冲区，因此，线程不断从回调中取得PCM数据时，需要首先取得缓冲区填充的数量，如果队列中的4个缓冲区都填满了数据，则等待消耗，再从回调中取得PCM数据入队SL中进行播放。实现代码如下：
```
class SLESDevice : public AudioDevice {
public:
    SLESDevice();

    virtual ~SLESDevice();

    int open(const AudioDeviceSpec *desired, AudioDeviceSpec *obtained) override;

    void start() override;

    void stop() override;

    void pause() override;

    void resume() override;

    void flush() override;

    void setStereoVolume(float left_volume, float right_volume) override;

    virtual void run();

private:
    // 转换成SL采样率
    SLuint32 getSLSampleRate(int sampleRate);

    // 获取SLES音量
    SLmillibel getAmplificationLevel(float volumeLevel);

private:
    // 引擎接口
    SLObjectItf slObject;
    SLEngineItf slEngine;

    // 混音器
    SLObjectItf slOutputMixObject;

    // 播放器对象
    SLObjectItf slPlayerObject;
    SLPlayItf slPlayItf;
    SLVolumeItf slVolumeItf;

    // 缓冲器队列接口
    SLAndroidSimpleBufferQueueItf slBufferQueueItf;

    AudioDeviceSpec audioDeviceSpec;    // 音频设备参数
    int bytes_per_frame;                // 一帧占多少字节
    int milli_per_buffer;               // 一个缓冲区时长占多少
    int frames_per_buffer;              // 一个缓冲区有多少帧
    int bytes_per_buffer;               // 一个缓冲区的大小
    uint8_t *buffer;                    // 缓冲区
    size_t buffer_capacity;             // 缓冲区总大小

    Mutex mMutex;
    Condition mCondition;
    Thread *audioThread;                // 音频播放线程
    int abortRequest;                   // 终止标志
    int pauseRequest;                   // 暂停标志
    int flushRequest;                   // 刷新标志

};

#define OPENSLES_BUFFERS 4 // 最大缓冲区数量
#define OPENSLES_BUFLEN  10 // 缓冲区长度(毫秒)

SLESDevice::SLESDevice() {
    slObject = NULL;
    slEngine = NULL;
    slOutputMixObject = NULL;
    slPlayerObject = NULL;
    slPlayItf = NULL;
    slVolumeItf = NULL;
    slBufferQueueItf = NULL;
    memset(&audioDeviceSpec, 0, sizeof(AudioDeviceSpec));
    abortRequest = 1;
    pauseRequest = 0;
    flushRequest = 0;
    audioThread = NULL;
}

SLESDevice::~SLESDevice() {
    memset(&audioDeviceSpec, 0, sizeof(AudioDeviceSpec));
    if (slPlayerObject != NULL) {
        (*slPlayerObject)->Destroy(slPlayerObject);
        slPlayerObject = NULL;
        slPlayItf = NULL;
        slVolumeItf = NULL;
        slBufferQueueItf = NULL;
    }

    if (slOutputMixObject != NULL) {
        (*slOutputMixObject)->Destroy(slOutputMixObject);
        slOutputMixObject = NULL;
    }

    if (slObject != NULL) {
        (*slObject)->Destroy(slObject);
        slObject = NULL;
        slEngine = NULL;
    }
}

void SLESDevice::start() {
    // 回调存在时，表示成功打开SLES音频设备，另外开一个线程播放音频
    if (audioDeviceSpec.callback != NULL) {
        abortRequest = 0;
        pauseRequest = 0;
        if (!audioThread) {
            audioThread = new Thread(this, Priority_High);
            audioThread->start();
        }
    } else {
        ALOGE("audio device callback is NULL!");
    }
}

void SLESDevice::stop() {
    mMutex.lock();
    abortRequest = 1;
    if (slPlayItf) {
        (*slPlayItf)->SetPlayState(slPlayItf, SL_PLAYSTATE_STOPPED);
    }
    mCondition.signal();
    mMutex.unlock();
    if (audioThread) {
        audioThread->join();
        delete audioThread;
        audioThread = NULL;
    }
}

void SLESDevice::pause() {
    mMutex.lock();
    pauseRequest = 1;
    mCondition.signal();
    mMutex.unlock();
}

void SLESDevice::resume() {
    mMutex.lock();
    pauseRequest = 0;
    mCondition.signal();
    mMutex.unlock();
}

/**
 * 清空SL缓冲队列
 */
void SLESDevice::flush() {
    mMutex.lock();
    flushRequest = 1;
    mCondition.signal();
    mMutex.unlock();
}

/**
 * 设置音量
 * @param left_volume
 * @param right_volume
 */
void SLESDevice::setStereoVolume(float left_volume, float right_volume) {
    Mutex::Autolock lock(mMutex);
    if (slVolumeItf != NULL) {
        SLmillibel level = getAmplificationLevel((left_volume + right_volume) / 2);
        SLresult result = (*slVolumeItf)->SetVolumeLevel(slVolumeItf, level);
        if (result != SL_RESULT_SUCCESS) {
            ALOGE("slVolumeItf->SetVolumeLevel failed %d\n", (int)result);
        }
    }
}

void SLESDevice::run() {
    uint8_t *next_buffer = NULL;
    int next_buffer_index = 0;

    if (!abortRequest && !pauseRequest) {
        (*slPlayItf)->SetPlayState(slPlayItf, SL_PLAYSTATE_PLAYING);
    }

    while (true) {

        // 退出播放线程
        if (abortRequest) {
            break;
        }

        // 暂停
        if (pauseRequest) {
            continue;
        }

        // 获取缓冲队列状态
        SLAndroidSimpleBufferQueueState slState = {0};
        SLresult slRet = (*slBufferQueueItf)->GetState(slBufferQueueItf, &slState);
        if (slRet != SL_RESULT_SUCCESS) {
            ALOGE("%s: slBufferQueueItf->GetState() failed\n", __func__);
            mMutex.unlock();
        }
        // 判断暂停或者队列中缓冲区填满了
        mMutex.lock();
        if (!abortRequest && (pauseRequest || slState.count >= OPENSLES_BUFFERS)) {
            while (!abortRequest && (pauseRequest || slState.count >= OPENSLES_BUFFERS)) {

                if (!pauseRequest) {
                    (*slPlayItf)->SetPlayState(slPlayItf, SL_PLAYSTATE_PLAYING);
                }
                mCondition.waitRelative(mMutex, 10 * 1000000);
                slRet = (*slBufferQueueItf)->GetState(slBufferQueueItf, &slState);
                if (slRet != SL_RESULT_SUCCESS) {
                    ALOGE("%s: slBufferQueueItf->GetState() failed\n", __func__);
                    mMutex.unlock();
                }

                if (pauseRequest) {
                    (*slPlayItf)->SetPlayState(slPlayItf, SL_PLAYSTATE_PAUSED);
                }
            }

            if (!abortRequest && !pauseRequest) {
                (*slPlayItf)->SetPlayState(slPlayItf, SL_PLAYSTATE_PLAYING);
            }

        }
        if (flushRequest) {
            (*slBufferQueueItf)->Clear(slBufferQueueItf);
            flushRequest = 0;
        }
        mMutex.unlock();

        mMutex.lock();
        // 通过回调填充PCM数据
        if (audioDeviceSpec.callback != NULL) {
            next_buffer = buffer + next_buffer_index * bytes_per_buffer;
            next_buffer_index = (next_buffer_index + 1) % OPENSLES_BUFFERS;
            audioDeviceSpec.callback(audioDeviceSpec.userdata, next_buffer, bytes_per_buffer);
        }
        mMutex.unlock();

        // 刷新缓冲区还是将数据入队缓冲区
        if (flushRequest) {
            (*slBufferQueueItf)->Clear(slBufferQueueItf);
            flushRequest = 0;
        } else {
            if (slPlayItf != NULL) {
                (*slPlayItf)->SetPlayState(slPlayItf, SL_PLAYSTATE_PLAYING);
            }
            slRet = (*slBufferQueueItf)->Enqueue(slBufferQueueItf, next_buffer, bytes_per_buffer);
            if (slRet == SL_RESULT_SUCCESS) {
                // do nothing
            } else if (slRet == SL_RESULT_BUFFER_INSUFFICIENT) {
                // don't retry, just pass through
                ALOGE("SL_RESULT_BUFFER_INSUFFICIENT\n");
            } else {
                ALOGE("slBufferQueueItf->Enqueue() = %d\n", (int)slRet);
                break;
            }
        }
    }
    ALOGD("audio play thread exit!");
}


/**
 * SLES缓冲回调
 * @param bf
 * @param context
 */
void slBufferPCMCallBack(SLAndroidSimpleBufferQueueItf bf, void *context) {

}

/**
 * 打开音频设备，并返回缓冲区大小
 * @param desired
 * @param obtained
 * @return
 */
int SLESDevice::open(const AudioDeviceSpec *desired, AudioDeviceSpec *obtained) {
    SLresult result;
    result = slCreateEngine(&slObject, 0, NULL, 0, NULL, NULL);
    if ((result) != SL_RESULT_SUCCESS) {
        ALOGE("%s: slCreateEngine() failed", __func__);
        return -1;
    }
    result = (*slObject)->Realize(slObject, SL_BOOLEAN_FALSE);
    if (result != SL_RESULT_SUCCESS) {
        ALOGE("%s: slObject->Realize() failed", __func__);
        return -1;
    }
    result = (*slObject)->GetInterface(slObject, SL_IID_ENGINE, &slEngine);
    if (result != SL_RESULT_SUCCESS) {
        ALOGE("%s: slObject->GetInterface() failed", __func__);
        return -1;
    }

    const SLInterfaceID mids[1] = {SL_IID_ENVIRONMENTALREVERB};
    const SLboolean mreq[1] = {SL_BOOLEAN_FALSE};
    result = (*slEngine)->CreateOutputMix(slEngine, &slOutputMixObject, 1, mids, mreq);
    if (result != SL_RESULT_SUCCESS) {
        ALOGE("%s: slEngine->CreateOutputMix() failed", __func__);
        return -1;
    }
    result = (*slOutputMixObject)->Realize(slOutputMixObject, SL_BOOLEAN_FALSE);
    if (result != SL_RESULT_SUCCESS) {
        ALOGE("%s: slOutputMixObject->Realize() failed", __func__);
        return -1;
    }
    SLDataLocator_OutputMix outputMix = {SL_DATALOCATOR_OUTPUTMIX, slOutputMixObject};
    SLDataSink audioSink = {&outputMix, NULL};

    SLDataLocator_AndroidSimpleBufferQueue android_queue = {
            SL_DATALOCATOR_ANDROIDSIMPLEBUFFERQUEUE,
            OPENSLES_BUFFERS
    };

    // 根据通道数设置通道mask
    SLuint32 channelMask;
    switch (desired->channels) {
        case 2: {
            channelMask = SL_SPEAKER_FRONT_LEFT | SL_SPEAKER_FRONT_RIGHT;
            break;
        }
        case 1: {
            channelMask = SL_SPEAKER_FRONT_CENTER;
            break;
        }
        default: {
            ALOGE("%s, invalid channel %d", __func__, desired->channels);
            return -1;
        }
    }
    SLDataFormat_PCM format_pcm = {
            SL_DATAFORMAT_PCM,              // 播放器PCM格式
            desired->channels,              // 声道数
            getSLSampleRate(desired->freq), // SL采样率
            SL_PCMSAMPLEFORMAT_FIXED_16,    // 位数 16位
            SL_PCMSAMPLEFORMAT_FIXED_16,    // 和位数一致
            channelMask,                    // 格式
            SL_BYTEORDER_LITTLEENDIAN       // 小端存储
    };

    SLDataSource slDataSource = {&android_queue, &format_pcm};

    const SLInterfaceID ids[3] = {SL_IID_ANDROIDSIMPLEBUFFERQUEUE, SL_IID_VOLUME, SL_IID_PLAY};
    const SLboolean req[3] = {SL_BOOLEAN_TRUE, SL_BOOLEAN_TRUE, SL_BOOLEAN_TRUE};

    result = (*slEngine)->CreateAudioPlayer(slEngine, &slPlayerObject, &slDataSource,
                                            &audioSink, 3, ids, req);
    if (result != SL_RESULT_SUCCESS) {
        ALOGE("%s: slEngine->CreateAudioPlayer() failed", __func__);
        return -1;
    }

    result = (*slPlayerObject)->Realize(slPlayerObject, SL_BOOLEAN_FALSE);
    if (result != SL_RESULT_SUCCESS) {
        ALOGE("%s: slPlayerObject->Realize() failed", __func__);
        return -1;
    }

    result = (*slPlayerObject)->GetInterface(slPlayerObject, SL_IID_PLAY, &slPlayItf);
    if (result != SL_RESULT_SUCCESS) {
        ALOGE("%s: slPlayerObject->GetInterface(SL_IID_PLAY) failed", __func__);
        return -1;
    }

    result = (*slPlayerObject)->GetInterface(slPlayerObject, SL_IID_VOLUME, &slVolumeItf);
    if (result != SL_RESULT_SUCCESS) {
        ALOGE("%s: slPlayerObject->GetInterface(SL_IID_VOLUME) failed", __func__);
        return -1;
    }

    result = (*slPlayerObject)->GetInterface(slPlayerObject, SL_IID_ANDROIDSIMPLEBUFFERQUEUE,
                                             &slBufferQueueItf);
    if (result != SL_RESULT_SUCCESS) {
        ALOGE("%s: slPlayerObject->GetInterface(SL_IID_ANDROIDSIMPLEBUFFERQUEUE) failed", __func__);
        return -1;
    }

    result = (*slBufferQueueItf)->RegisterCallback(slBufferQueueItf, slBufferPCMCallBack, this);
    if (result != SL_RESULT_SUCCESS) {
        ALOGE("%s: slBufferQueueItf->RegisterCallback() failed", __func__);
        return -1;
    }

    // 这里计算缓冲区大小等参数
    bytes_per_frame   = format_pcm.numChannels * format_pcm.bitsPerSample / 8;  // 一帧占多少字节
    milli_per_buffer  = OPENSLES_BUFLEN;                                        // 每个缓冲区占多少毫秒
    frames_per_buffer = milli_per_buffer * format_pcm.samplesPerSec / 1000000;  // 一个缓冲区有多少帧数据
    bytes_per_buffer  = bytes_per_frame * frames_per_buffer;                    // 一个缓冲区大小
    buffer_capacity   = OPENSLES_BUFFERS * bytes_per_buffer;

    ALOGI("OpenSL-ES: bytes_per_frame  = %d bytes\n",  bytes_per_frame);
    ALOGI("OpenSL-ES: milli_per_buffer = %d ms\n",     milli_per_buffer);
    ALOGI("OpenSL-ES: frame_per_buffer = %d frames\n", frames_per_buffer);
    ALOGI("OpenSL-ES: buffer_capacity  = %d bytes\n",  buffer_capacity);
    ALOGI("OpenSL-ES: buffer_capacity  = %d bytes\n",  (int)buffer_capacity);

    if (obtained != NULL) {
        *obtained = *desired;
        obtained->size = (uint32_t)buffer_capacity;
        obtained->freq = format_pcm.samplesPerSec / 1000;
    }
    audioDeviceSpec = *desired;

    // 创建缓冲区
    buffer = (uint8_t *)malloc(buffer_capacity);
    if (!buffer) {
        ALOGE("%s: failed to alloc buffer %d\n", __func__, (int)buffer_capacity);
        return -1;
    }

    // 填充缓冲区数据
    memset(buffer, 0, buffer_capacity);
    for(int i = 0; i < OPENSLES_BUFFERS; ++i) {
        result = (*slBufferQueueItf)->Enqueue(slBufferQueueItf,
                                              buffer + i * bytes_per_buffer,
                                              bytes_per_buffer);
        if (result != SL_RESULT_SUCCESS)  {
            ALOGE("%s: slBufferQueueItf->Enqueue(000...) failed", __func__);
        }
    }

    ALOGD("open SLES Device success");
    // 返回缓冲大小
    return buffer_capacity;
}

/**
 * 转换成SL的采样率
 * @param sampleRate
 * @return
 */
SLuint32 SLESDevice::getSLSampleRate(int sampleRate) {
    switch (sampleRate) {
        case 8000: {
            return SL_SAMPLINGRATE_8;
        }
        case 11025: {
            return SL_SAMPLINGRATE_11_025;
        }
        case 12000: {
            return SL_SAMPLINGRATE_12;
        }
        case 16000: {
            return SL_SAMPLINGRATE_16;
        }
        case 22050: {
            return SL_SAMPLINGRATE_22_05;
        }
        case 24000: {
            return SL_SAMPLINGRATE_24;
        }
        case 32000: {
            return SL_SAMPLINGRATE_32;
        }
        case 44100: {
            return SL_SAMPLINGRATE_44_1;
        }
        case 48000: {
            return SL_SAMPLINGRATE_48;
        }
        case 64000: {
            return SL_SAMPLINGRATE_64;
        }
        case 88200: {
            return SL_SAMPLINGRATE_88_2;
        }
        case 96000: {
            return SL_SAMPLINGRATE_96;
        }
        case 192000: {
            return SL_SAMPLINGRATE_192;
        }
        default: {
            return SL_SAMPLINGRATE_44_1;
        }
    }
}

/**
 * 计算SL的音量
 * @param volumeLevel
 * @return
 */
SLmillibel SLESDevice::getAmplificationLevel(float volumeLevel) {
    if (volumeLevel < 0.00000001) {
        return SL_MILLIBEL_MIN;
    }
    SLmillibel mb = lroundf(2000.f * log10f(volumeLevel));
    if (mb < SL_MILLIBEL_MIN) {
        mb = SL_MILLIBEL_MIN;
    } else if (mb > 0) {
        mb = 0;
    }
    return mb;
}
```
至此，我们就把音频输出给弄完了。
完整代码请参考本人的播放器项目：[CainPlayer](https://github.com/CainKernel/CainPlayer)
