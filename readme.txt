三、主函数main()、解析函数parse()与编码函数encode()

main()是x264控制台程序的入口函数，可以看出main()的定义很简单，它主要调用了两个函数：parse()和encode()。main()首先调用parse()解析输入的命令行参数，然后调用encode()进行编码。

parse()用于解析命令行输入的参数（存储于argv[]中）。parse()的流程大致为：

（1）调用x264_param_default()为存储参数的结构体x264_param_t赋默认值；

（2）调用x264_param_default_preset()为x264_param_t赋值；

（3）在一个大循环中调用getopt_long()逐个解析输入的参数，并作相应的处理。举几个例子：

        a)“-h”：调用help()打开帮助菜单。
        b)“-V”调用print_version_info()打印版本信息。
        c)对于长选项，调用x264_param_parse()进行处理。

（4）调用select_input()解析输出文件格式（例如raw，flv，MP4…）

（5）调用select_output()解析输入文件格式（例如yuv，y4m…）


encode()编码YUV为H.264码流，主要流程为：

（1）调用x264_encoder_open()打开H.264编码器；

（2）调用x264_encoder_parameters()获得当前的参数集x264_param_t，用于后续步骤中的一些配置；

（3）调用输出格式（H.264裸流、FLV、mp4等）对应cli_output_t结构体的set_param()方法，为输出格式的封装器设定参数。其中参数源自于上一步骤得到的x264_param_t；

（4）如果不是在每个keyframe前面都增加SPS/PPS/SEI的话，就调用x264_encoder_headers()在整个码流前面加SPS/PPS/SEI；

（5）进入一个循环中进行一帧一帧的将YUV编码为H.264：

        a)调用输入格式（YUV、Y4M等）对应的cli_vid_filter_t结构体get_frame()方法，获取一帧YUV数据。
        b)调用encode_frame()编码该帧YUV数据为H.264数据，并且输出出来。该函数内部调用x264_encoder_encode()完成编码工作，调用输出格式对应cli_output_t结构体的write_frame()完成了输出工作。
       c)调用输入格式（YUV、Y4M等）对应的cli_vid_filter_t结构体release_frame()方法，释放刚才获取的YUV数据。
       d)调用print_status()输出一些统计信息。

（6）编码即将结束的时候，进入另一个循环，输出编码器中缓存的视频帧：

       a)不再传递新的YUV数据，直接调用encode_frame()，将编码器中缓存的剩余几帧数据编码输出出来。
       b)调用print_status()输出一些统计信息。

（7）调用x264_encoder_close()关闭H.264编码器。