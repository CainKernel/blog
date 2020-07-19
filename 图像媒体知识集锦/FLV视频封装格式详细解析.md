### FLV的定义：
Flash Video(简称FLV)，是一种流行的网络格式，是Adobe推出的。目前大部分视频网站都支持这种格式。

### FLV的文件结构
FLV文件由FLV Header 和 FLV Body构成。

### FLV Header 头部信息
Header 部分记录了FLV的类型、版本、流信息、Header 长度等。一般整个Header占用9个字节，大于9个字节则表示头部信息在这基础之上还存在扩展数据。Header 的头部信息排布如下所示：
![FLV头部信息排布](http://upload-images.jianshu.io/upload_images/2103804-9ee339c2098b378b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### FLV Body 文件内容部分
Body 是由一个个Tag组成的，每个Tag下面有一块4个字节的空间，用来记录这个Tag 的长度。这个后置的PreviousTagSize用于逆向读取处理，表示的是前面的Tag的大小，对于FLV版本0x01来说，数值等于 11 + Tag的DataSize。其结构排布如下：
![FLV Body结构排布](http://upload-images.jianshu.io/upload_images/2103804-9c99eb6859ee6da6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### FLV Tag
每个Tag 也是由两部分组成的：Tag Header 和 Tag Data。Tag Header 存放了当前Tag的类型，数据长度、时间戳、时间戳扩展、StreamsID等信息，然后再接着数据区Tag Data。Tag的排布如下：
![Tag构成排布](http://upload-images.jianshu.io/upload_images/2103804-45946bb014093dce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Tag Data
Tag Data分成 Audio，Video，Script 三种。

### Audio Tag Data
音频的Tag Data又分为 AudioTagHeader 和 Data 数据区，其排布结构如下图所示：
![音频Tag Data排布](http://upload-images.jianshu.io/upload_images/2103804-9d71ded1a69a1193.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
AudioTagHeader通常占用1个字节，AAC编码则会多出一个AACPacketType的字节，用于表示AAC的序列头还是裸数据。
其中，前4bits表示SoundFormat，其数值对应声音格式，如下：
0 - Linear PCM, platform endian
1 - ADPCM
2 - MP3
3 - LinearPCM，little endian
4 - Nellymoser 16-kHz mono
5 - Nellymoser 8-kHZ mono
6 - Nellymoser
7 - G.711 A-law logarithmic PCM
8 - G.711 U-law logarithmic PCM
9 - reserved
10 - AAC
11 - Speex
14 - MP3 8-kHz
15 - Device-specific sound

第5、6bit 表示SoundRate，数值对应采样率，对于AAC来说，总是3：
0 - 5.5kHz
1 - 11kHz
2 - 22kHz
3 - 44kHz

第7bit 表示采样大小：
0 - snd 8 bit
1 - snd 16 bit

第8bit 表示音频声道数，对于AAC来说，总是1：
0 - sndMono
1 - sndStereo

audio Data数据区，根据SoundFormat的数值来确定，如果SoundFormat = 10，则Data数据区是AAC编码部分，其他声音类型，则根据具体格式进行解析。

### AAC编码
针对AAC编码，音频Data数据区的定义如下：
AACPacketType = 0 时，表示AAC序列头，也就是AudioSpecificConfig， AACPacketType = 1 时，表示AAC的裸流，也就是AAC Raw frame data。

### AudioSpecificConfig 
AudioSpecificConfig 只出现在第一个Audio Tag中，结构如下，
```
AudioSpecificConfig() {     
    audioObjectType;                              5bits
    samplingFrequencyIndex;                 4bits
    if  ( samplingFrequencyIndex == 0xf ) {
        samplingFrequency;                         24bits
    }
    channelConfiguration;                           4bits
    sbrPresentFlag = -1;
    if  ( audioObjectType == 5 ) {
        extensionAudioObjectType = audioObjectType;
        sbrPresentFlag = 1;
        extensionSamplingFrequencyIndex;            4bits
        if ( extensionSamplingFrequencyIndex == 0xf )  {
            extensionSamplingFrequency;             24bits
        }
        audioObjectType;                            5bits
    } else {
         extensionAudioObjectType = 0;
    }
    
    if ( audioObjectType == 1 || audioObjectType == 2 ||
         audioObjectType == 3 || audioObjectType == 4 ||
         audioObjectType == 6 || audioObjectType == 7 ) {
        GASpecificConfig();
    }
    if ( audioObjectType == 8 )  {
        CelpSpecificConfig();
    }
    if ( audioObjectType == 9 ) {
        HvxcSpecificConfig();
    }
    if ( audioObjectType == 12 ) {
        TTSSpecificConfig();
    }
    if ( audioObjectType == 13 || audioObjectType == 14 ||
        audioObjectType == 15 || audioObjectType == 16 ) {
        StructureAudioSpecificConfig();
    }
    if ( audioObjectType == 17 || audioObjectType == 19 ||
        audioObjectType == 20 || audioObjectType == 21 ||
        audioObjectType == 22 || audioObjectType == 23 ) {
        GASpecificConfig();
    }
    if ( audioObjectType == 24 ) {
        ErrorResilientCelpSpecificConfig();
    }
    if ( audioObjectType == 25 ) {
        ErrorResilientHvxcSpecificConfig();
    }
    if ( audioObjectType == 26 || audioObjectType == 27 ) {
        ParametricSpecificConfig();
    }
    if ( audioObjectType == 17 || audioObjectType == 19 ||
        audioObjectType == 20 || audioObjectType == 21 ||
        audioObjectType == 22 || audioObjectType == 23 ||
        audioObjectType == 24 || audioObjectType == 25 ||
        audioObjectType == 26 || audioObjectType == 27 )  {
        epConfig;                                      2bits
        if ( epConfig == 2 || epConfig == 3 ) {
            ErrorProtectionSpecificConfig();
        }
        if ( epConfig == 3 ) {
            directMapping;                               1bit
            if ( ! directMapping ) {
              /* tbd */
            }
        }
    }
    if ( audioObjectType == 28 ) {
        SSCSpecificConfig();
    }
    if ( extensionAudioObjectType != 5 && bits_to_decode() >= 16 ) { 
        syncExtensionType;                                          11bits
        if ( syncExtensionType == 0x2b7 ) {
            extensionAudioObjectType;                                 5bits
            if ( extensionAudioObjectType == 5 ) {
                sbrPresentFlag;                                        1bit
                if ( sbrPresentFlag == 1 ) {
                    extensionSamplingFrequencyIndex;                     4bits
                    if ( extensionSamplingFrequencyIndex == 0xf ) {
                        extensionSamplingFrequency;                      24bits
                    }
                }
            }
        }
    }
}

```
AudioSpecificConfig 简化格式如下：
```
audioObjectType               5bits    编码结构类型，AAC-LC为2
samplingFreguencyIndex        4bits    音频采样率索引值
channelConfiguration          4bits    音频声道数
GASpecificConfig
    frameLengthFlag           1bits    标志位，用于表明IMDCT窗口长度，为0
    dependsOnCoreCoder        1bits    标志位，用于表明是否依赖corecoder，为0
    extensionFlag             1bits    扩展标志位，选择了AAC-LC，这里必须为0
```
其中，samplingFreguencyIndex 对应关系如下：
```
0 - 96000
1 - 88200
2 - 64000
3 - 48000
4 - 44100
5 - 32000
6 - 24000
7 - 22050
8 - 16000
9 - 12000
10 - 11025
11 - 8000
12 - 7350
13 - Reserved
14 - Reserved
15 - frequency is written explictly
```

### AAC Raw frame data
AAC裸流AAC Raw frame data，即AAC音频原始数据，不包括AAC头数据ADTS。

### AAC头部数据 ADTS
ADTS包括采样频率、帧长度等信息，共7个字节，分为两部分：
adts_fixed_header 、adts_variable_header

### adts_fixed_header 排布如下： 
```
adts_fixed_header() {
    syncword                  12bits    always 0xFFF
    ID                        1bit      0--MPEG-4   1--MPEG-2
    layer                     2bits     always '00'
    protection_absent         1bit
    profile                   2bits     0--Main profile     1--Low Complexity profile(LC) 
                                        2--Scalable Sampling Rate profile(SSR) 3--reserved
    sampling_frequency_index  4bits     音频采样率索引值
    private_bit               1bit
    channel_configuration     3bits     音频声道数
    original_copy             1bit
    home                      1bit
}
```
channel_configuration 对应关系如下：
```
0 bit : Defined in AOT Specific Config
1 bit : 1 channel：front-center
2 bit : 2 channels：front-left，front-right  
3 bit : 3 channels：front-center，front-left，front-right  
4 bit : 4 channels：front-center，front-left，front-right，back-center
5 bit : 5 channels：front-center，front-left，front-right，back-left，back-right  
6 bit : 6 channels：front-center，front-left，front-right，back-left，back-right，LFE-channel 
7 bit : 8 channels：front-center，front-left，front-right，side-left，side-right，back-left，back-right，LFE-channel 
8-15 bit：Reserved
```
### adts_variable_header排布如下：
```
adts_variable_header() {
    copyright_identification_bit        1bit
    copyright_identification_start      1bit
    aac_frame_length                    13bits  ADTS头 + AAC原始流     
    adts_buffer_fullness                11bits  0x7FF表示码率可变的码流
    number_of_raw_data_blocks_in_frame  2bits 
}
```

### MP3编码
如果是MP3 编码部分，audio 数据区直接为 MP3 头 + MP3 原始数据。

### MP3 头部格式如下：
```
MP3FrameHeader
{
    sync                11bits  同步信息always 0xFFF     
    version             2bits   版本 00--MPEG 2.5  01--未定义 10--MPEG 2  11--MPEG 1    
    layer               2bits   层 00--未定义 01--Layer 3  10--Layer 2  11--Layer 1    
    error_protection    1bit    CRC校验0--校验  1--不校验    
    bitrate_index       4bits   位率 
    sampling_frequency  2bits   采样率索引值   
    padding             1bit    帧长调节    
    private             1bit    保留字    
    mode                2bits   声道模式    
    mode_extension      2bits   扩充模式    
    copyright           1bit    版权     
    original            1bit    原版标志    
    emphasis            2bits   强调模式 
}
```
sampling_frequency对应关系如下：
```
version   00--MPEG 2.5    01--未定义  10--MPEG 2    11--MPEG 1 
MPEG 1:   00--44.1kHz     01--48kHz   10--32kHz     11--未定义 
MPEG 2:   00--22.05kHz    01--24kHz   10--16kHz     11--未定义 
MPEG 2.5: 00--11.025kHz   01--12kHz   10--8kHz      11--未定义
```
mode对应关系如下： 
```
00--立体声Stereo   01--Joint Stereo   10--双声道   11--单声道
```

### Video Tag Data
如果Tag 包中的TagType = 9时，这个Tag就表示video数据。则StreamID之后紧跟着 VideoTagHeader 和 Video Data 数据区。
Video Tag 有一个字节的VideoTagHeader 和 Video数据区部分组成

### VideoTagHeader
VideoTagHeader 通常由一个字节构成，前4bit表示类型，后4bit表示编码器Id。
![VideoTagHeader](http://upload-images.jianshu.io/upload_images/2103804-09071a0223b93da0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Video数据区
Video数据区部分格式不确定。对于H264/AVC编码部分，Video数据区排布如下:
![Video数据区排布](http://upload-images.jianshu.io/upload_images/2103804-c9f7e22e62ec5f40.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中，AVCsequenceheader只有出现在第一个Video Tag，只有一个。为了能够从FLV中获取NALU，必须知道前面的NALU长度所占的字节数，通常是1、2、4个字节，这个内容则必须从AVCDecoderConfigurationRecord中获取，这个遵从标准ISO/IEC 14496-15的5.2.4小节。AVCDecoderConfigurationRecord 的结构如下：
```
aligned(8) class AVCDecoderConfigurationRecord {
    unsigned int(8) configurationVersion = 1;  //版本号
    unsigned int(8) AVCProfileIndication;     //sps[1]，即0x67后面那个字节
    unsigned int(8) profile_compatibility;     //sps[2]
    unsigned int(8) AVCLevelIndication;      //sps[3]
    bit(6) reserved = '111111'b;
    unsigned int(2) lengthSizeMinusOne;     //NALUnitLength的长度减1，一般为3  
    bit(3) reserved = '111'b;
    unsigned int(5) numOfSequenceParameterSets;    //sps个数，一般为1
    for ( i=0; i<numOfSequenceParameterSets; i++ ) {
        unsigned int(16) sequenceParameterSetLength;  //sps的长度      
        bit(8*sequenceParameterSetLength) sequenceParameterSetNALUnit;
    }
    unsigned int(8) numOfPictureParameterSets;      //pps个数，一般为1
    for ( i=0; i<numOfPictureParameterSets; i++) {
        unsigned int(16) pictureParameterSetLength;   //pps长度   
        bit(8*pictureParameterSetLength) pictureParameterSetNALUnit;
   }
}
```
其中，lengthSizeMinusOne + 1 = NALU长度字段所占字节数

### Script Tag
当TagType = 0x12时， 这个Tag就是Script tag。Script Tag一般只有一个，是FLV文件的第一个Tag，用于存放FLV文件信息，比如时长、分辨率、音频采样率等。所有的数据都是以数据类型 + (数据长度) + 数据格式出现，数据类型占1个字节，数据长度看数据类型是否存在，后面才是数据。数据排布如下：
![ScriptTag排布](http://upload-images.jianshu.io/upload_images/2103804-3c8f4be78f6b6d13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中SCRIPTDATAOBJECTEND和SCRIPTDATAVARIABLEEND为0x000009，用于标记结尾

SCRIPTDATASTRING结构为：
```
StringLength    2bytes  
StringData      String 
```

SCRIPTDATALONGSTRING结构为：
```
StringLength  4bytes  
StringData    String  
```

ECMA array type结构为：
```
ECMAArrayLength   4bytes 
StringLength      2bytes 
StringData        String 
DataType          1byte 
Data              不定长 
SCRIPTDATAVARIABLEEND  
```

Object type结构为：
```
StringLength  2bytes 
StringData    String 
DataType      1byte  
DataVale      不定长 
SCRIPTDATAOBJECTEND  
```

Strict array type结构为：
```
ArrayNum    4bytes 
DataType    byte 
DataValue   不定长
```
类型在FLV的官方文档中有详细的介绍。

### onMeteData
onMeteData 是FLV文件中的第一个Tag，是ScriptData中对我们来说十分重要的信息，结构如下：
![onMeteData结构](http://upload-images.jianshu.io/upload_images/2103804-7c178a44d65c262b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### audiocodecid
音频编码类型id，跟AudioTagHeader中的SoundFormat是对应的。

#### videocodecid
视频编码类型id，跟VideoTagHeader 的 CodecID是对应的









