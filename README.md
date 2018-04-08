IJKMediaPlayback

IJKMediaPlayback是暴露给第三方使用的类，在此类中添加了：

/**
 *  录制视频（只能录制RTSP流）格式mov
 *
 *  @param streamURL        流地址
 *  @param filePath         录制的视频存储的路径
 *  @param bStop            是否开始录制（NO开始， YES结束）
 *  @return int             返回-1，录制失败， 0为正常
 */
- (int)rtsp2mov:(NSString *)streamURL storageFilePath:(NSString *)filePath isStop:(BOOL)bStop;

IJKMediaPlayer ios集成了3个视频播放框架，分别是IJKAVMoviePlayerController、IJKMPMoviePlayerController、IJKFFMoviePlayerController
电视宝（直播）使用了IJKFFMoviePlayerController，所以上面两个API（截图、录制视频）在IJKFFMoviePlayerController有做具体实现

#pragma --mark  录制视频（只能录制RTSP流）格式mov
- (int)rtsp2mov:(NSString *)streamURL storageFilePath:(NSString *)filePath isStop:(BOOL)bStop
{
    // 输入
    AVStream *i_video_stream = NULL;        //视频
    AVStream *i_audio_stream = NULL;        //音频
    AVFormatContext *i_fmt_ctx = NULL;
    
    // 输出
    AVStream *o_video_stream = NULL;        //视频
    AVStream *o_audio_stream = NULL;        //音频
    AVFormatContext *o_fmt_ctx = NULL;
    
//    ijkmp_global_init();
//    // FFMPEG注册
//    ijkav_register_all();
//    avcodec_register_all();
//    av_register_all();
//    if(avformat_network_init() != 0){
//        printf("avformat_network_init() 失败 \n");
//        return -1;
//    }
    
    // 视频存储的路径
    const char *filename = [filePath cStringUsingEncoding:NSUTF8StringEncoding];
    // 打开网络流或文件流
    if (avformat_open_input(&i_fmt_ctx, [streamURL cStringUsingEncoding:NSUTF8StringEncoding], NULL, NULL)!=0)
    {
        printf("无法打开输入流.\n");
        return -1;
    }
    
    if (avformat_find_stream_info(i_fmt_ctx, NULL)<0)
    {
        printf("找不到流信息.\n");
        return -1;
    }
    
    // 列出输入文件的相关流信息
    printf("---------------- 输入文件信息 ---------------\n");
    av_dump_format(i_fmt_ctx, 0, [streamURL cStringUsingEncoding:NSUTF8StringEncoding], 0);
    printf("-------------------------------------------------\n");
    
    
    /* find first video stream */
    for (unsigned i=0; i<i_fmt_ctx->nb_streams; i++)
    {
        if (i_fmt_ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {
            // 视频
            i_video_stream = i_fmt_ctx->streams[i];
        }
        else if(i_fmt_ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_AUDIO){
            // 音频
            i_audio_stream = i_fmt_ctx->streams[i];
        }
    }
    
    if (i_video_stream == NULL)
    {
        fprintf(stderr, "没有找到任何视频流媒体\n");
        return -1;
    }
    
    AVOutputFormat *oformat = av_guess_format(NULL, filename, NULL);
    /**
     *  avformat_alloc_output_context2(AVFormatContext **ctx, AVOutputFormat *oformat, const char *format_name, const char *filename)
     *
     *  ctx：            函数调用成功之后创建的AVFormatContext结构体
     *  oformat：        指定AVFormatContext中的AVOutputFormat，用于确定输出格式。如果指定为NULL，可以设定后两个参数（format_name或者filename）由FFmpeg猜测输出格式
     *  format_name：    指定输出格式的名称。根据格式名称，FFmpeg会推测输出格式。输出格式可以是“flv”，“mkv”等等
     *  filename：       指定输出文件的名称
     */
    if(avformat_alloc_output_context2(&o_fmt_ctx, oformat, NULL, filename) !=0){
        fprintf(stderr, "初始化o_fmt_ctx结构体失败\n");
        return -1;
    }
    
    /**************************************** 视频 **********************************/
    o_video_stream = avformat_new_stream(o_fmt_ctx, NULL);
    {
        if (o_video_stream == NULL){
            printf("没有获取到视频流信息.\n");
            return -1;
        }
        
        AVCodecParameters *oVcc;
        oVcc = o_video_stream->codecpar;
        
        //平均比特率
        oVcc->bit_rate = 400000;
        oVcc->codec_id = i_video_stream->codecpar->codec_id;
        oVcc->codec_type = i_video_stream->codecpar->codec_type;
        
        oVcc->sample_aspect_ratio.num = i_video_stream->sample_aspect_ratio.num;
        oVcc->sample_aspect_ratio.den = i_video_stream->sample_aspect_ratio.den;
        
        oVcc->extradata = i_video_stream->codecpar->extradata;
        oVcc->extradata_size = i_video_stream->codecpar->extradata_size;
        
        oVcc->width = i_video_stream->codecpar->width;
        oVcc->height = i_video_stream->codecpar->height;
        oVcc->format = i_video_stream->codecpar->format;
        
        //帧率
        o_video_stream->r_frame_rate = i_video_stream->r_frame_rate;
    }
    
    
    
    /**************************************** 音频 **********************************/
    o_audio_stream = avformat_new_stream(o_fmt_ctx, NULL);
    {
        if (o_audio_stream == NULL){
            printf("没有获取到音频流信息.\n");
            return -1;
        }
        
        AVCodecParameters *oAcc;
        oAcc = o_audio_stream->codecpar;
        
        oAcc->codec_id = AV_CODEC_ID_AAC;       //AAC格式
        oAcc->codec_type = AVMEDIA_TYPE_AUDIO;
        oAcc->bit_rate = 64000;
        //采样率（音频)
        oAcc->sample_rate = 44100;
        //声道数（音频）
        oAcc->channel_layout = AV_CH_LAYOUT_STEREO;
        oAcc->channels = av_get_channel_layout_nb_channels(oAcc->channel_layout);
    }
    
    /*******************************************************************************/
    
    // 列出输出文件的相关流信息
    printf("------------------- 输出文件信息 ------------------\n");
    av_dump_format(o_fmt_ctx, 0, filename, 1);
    printf("-------------------------------------------------\n");
    
    avio_open(&o_fmt_ctx->pb, filename, AVIO_FLAG_WRITE);
    

    if (!(o_fmt_ctx->flags & AVFMT_NOFILE)) {

    }
    
    if (!o_fmt_ctx->nb_streams) {
        fprintf(stderr, "output file dose not contain any stream\n");
        return -1;
    }
    
    // 根据文件名的后缀写相应格式的文件头
    if (avformat_write_header(o_fmt_ctx, NULL) < 0) {
        fprintf(stderr, "Could not write header for output file\n");
        return -1;
    }
    
    int last_pts = 0, last_dts = 0;
    int au_pts = 0, au_dts = 0;
    
    int64_t pts = 0, dts = 0;
    int64_t a_pts = 0, a_dts = 0;
    
    int64_t end_time = 0;
    
    while (!bStop)
    {
        end_time++;
        AVPacket i_pkt;
        av_init_packet(&i_pkt);
        i_pkt.size = 0;
        i_pkt.data = NULL;
        
        // 获取流中每一帧的数据流（包)
        if (av_read_frame(i_fmt_ctx, &i_pkt) < 0 )
            break;
        /*
         * pts and dts should increase monotonically
         * pts should be >= dts
         */
        
        if (i_pkt.stream_index == AVMEDIA_TYPE_VIDEO)
        {
            i_pkt.flags |= AV_PKT_FLAG_KEY;
            pts = i_pkt.pts;
            i_pkt.pts += last_pts;
            dts = i_pkt.dts;
            i_pkt.dts += last_dts;
            i_pkt.stream_index = 0;
            
            printf("%lld %lld\n", i_pkt.pts, i_pkt.dts);
            
            static int num = 1;
            printf("frame %d\n", num++);
            
            // 往输出流中写一个分包
            av_interleaved_write_frame(o_fmt_ctx, &i_pkt);
        }
        else if (i_pkt.stream_index == AVMEDIA_TYPE_AUDIO){
            i_pkt.flags |= AV_PKT_FLAG_KEY;
            a_pts = i_pkt.pts;
            i_pkt.pts += au_pts;
            a_dts = i_pkt.dts;
            i_pkt.dts += au_dts;
            i_pkt.stream_index = 1;
            
            printf("%lld %lld\n", i_pkt.pts, i_pkt.dts);
            
            static int num = 1;
            printf("frame %d\n", num++);
            
            // 往输出流中写一个分包
            av_interleaved_write_frame(o_fmt_ctx, &i_pkt);
        }
        
        if (end_time == 2000)
        {
            bStop = YES;
        }
    }
    
    last_dts += dts;
    last_pts += pts;
    
    au_dts += a_dts;
    au_pts += a_pts;
    
    avformat_close_input(&i_fmt_ctx);
    
    // 写输出流（文件）的文件尾
    av_write_trailer(o_fmt_ctx);
    
    // 视频信息
    av_freep(&o_fmt_ctx->streams[0]->codecpar);
    av_freep(&o_fmt_ctx->streams[0]);
    
    // 音频信息
    av_freep(&o_fmt_ctx->streams[1]->codecpar);
    av_freep(&o_fmt_ctx->streams[1]);
    
    avio_close(o_fmt_ctx->pb);
    av_free(o_fmt_ctx);
    
    return 0;
}


Android工程已上传，参见LVAS工程。有问题可以加QQ或者微信，QQ:942737690，微信：lm3515

工程直播流测试地址：rtmp://live.hkstv.hk.lxdns.com/live/hks
