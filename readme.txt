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

七、从源代码可以看出，x264_encoder_encode()的流程大致如下：
（1）调用x264_frame_pop_unused获取一个空的fenc（x264_frame_t类型）用于存储一帧编码像素数据。

（2）调用x264_frame_copy_picture()将外部结构体的pic_in（x264_picture_t类型）的数据拷贝给内部结构体的fenc（x264_frame_t类型）。

（3）调用x264_lookahead_put_frame()将fenc放入Lookahead模块的队列中，等待确定帧类型。

（4）调用x264_lookahead_get_frames()分析Lookahead模块中一个帧的帧类型。分析后的帧保存在frames.current[]中。

（5）调用x264_frame_shift()从frames.current[]中取出分析帧类型之后的fenc。

（6）调用x264_reference_update()更新参考帧队列frames.reference[]。

（7）如果编码帧fenc是IDR帧，调用x264_reference_reset()清空参考帧队列frames.reference[]。

（8）调用x264_reference_build_list()创建参考帧列表List0和List1。

（9）根据选项做一些配置：

          a)、如果b_aud不为0，输出AUD类型NALU

          b)、在当前帧是关键帧的情况下，如果b_repeat_headers不为0，调用x264_sps_write()和x264_pps_write()输出SPS和PPS。

          c)、输出一些特殊的SEI信息，用于适配各种解码器。

（10）调用x264_slice_init()初始化Slice Header信息。

（11）调用x264_slices_write()进行编码。该部分是libx264的核心，在后续文章中会详细分析。

（12）调用x264_encoder_frame_end()做一些编码后的后续处理。


九、根据源代码简单梳理了x264_slice_write()的流程，如下所示：

（1）、调用x264_nal_start()开始输出一个NALU。

（2）、x264_macroblock_thread_init()：初始化宏块重建像素缓存fdec_buf[]和编码像素缓存fenc_buf[]。

（3）、调用x264_slice_header_write()输出 Slice Header。

（4）、进入一个循环，该循环每执行一遍编码一个宏块：

               a)、 每处理一行宏块，调用一次x264_fdec_filter_row()执行滤波模块。

               b)、 调用x264_macroblock_cache_load_progressive()将要编码的宏块的周围的宏块的信息读进来。

               c) 、调用x264_macroblock_analyse()执行分析模块。

               d) 、调用x264_macroblock_encode()执行宏块编码模块。

               e) 、调用x264_macroblock_write_cabac()/x264_macroblock_write_cavlc()执行熵编码模块。

               f) 、调用x264_macroblock_cache_save()保存当前宏块的信息。

              g) 、调用x264_ratecontrol_mb()执行码率控制。

              h) 、准备处理下一个宏块。

（5）、调用x264_nal_end()结束输出一个NALU。


十一、尽管x264_macroblock_analyse()的源代码比较长，但是它的逻辑比较清晰，如下所示：

（1）、如果当前是I Slice，调用x264_mb_analyse_intra()进行Intra宏块的帧内预测模式分析。


（2）、如果当前是P Slice，则进行下面流程的分析：

    a)、调用x264_macroblock_probe_pskip()分析是否为Skip宏块，如果是的话则不再进行下面分析。

    b)、调用x264_mb_analyse_inter_p16x16()分析P16x16帧间预测的代价。

    c)、调用x264_mb_analyse_inter_p8x8()分析P8x8帧间预测的代价。

    d)、如果P8x8代价值小于P16x16，则依次对4个8x8的子宏块分割进行判断：

        i、调用x264_mb_analyse_inter_p4x4()分析P4x4帧间预测的代价。

        ii、如果P4x4代价值小于P8x8，则调用 x264_mb_analyse_inter_p8x4()和x264_mb_analyse_inter_p4x8()分析P8x4和P4x8帧间预测的代价。

    e)、如果P8x8代价值小于P16x16，调用x264_mb_analyse_inter_p16x8()和x264_mb_analyse_inter_p8x16()分析P16x8和P8x16帧间预测的代价。

    f)、此外还要调用x264_mb_analyse_intra()，检查当前宏块作为Intra宏块编码的代价是否小于作为P宏块编码的代价（P Slice中也允许有Intra宏块）。



（3）、如果当前是B Slice，则进行和P Slice类似的处理。

十二、总体说来x264_mb_analyse_intra()通过计算Intra16x16，Intra8x8，Intra4x4这3中帧内预测模式的代价，比较后得到最佳的帧内预测模式。该函数的大致流程如下：
   （1）、进行Intra16X16模式的预测

       a)、调用predict_16x16_mode_available()根据周围宏块的情况判断其可用的预测模式（主要检查左边和上边的块是否可用）。

       b)、循环计算4种Intra16x16帧内预测模式：

           i.调用predict_16x16[]()汇编函数进行Intra16x16帧内预测

           ii.调用x264_pixel_function_t中的mbcmp[]()计算编码代价（mbcmp[]()指向SAD或者SATD汇编函数）。

       c)、获取最小代价的Intra16x16模式。



   （2）、进行Intra8x8模式的预测



   （3）、进行Intra4X4块模式的预测

       a)、循环处理16个4x4的块：

           i.调用x264_mb_predict_intra4x4_mode()根据周围宏块情况判断该块可用的预测模式。

           ii.循环计算9种Intra4x4的帧内预测模式：

               1)、调用predict_4x4 []()汇编函数进行Intra4x4帧内预测

               2)、调用x264_pixel_function_t中的mbcmp[]()计算编码代价（mbcmp[]()指向SAD或者SATD汇编函数）。

           iii.获取最小代价的Intra4x4模式。

       b)、将16个4X4块的最小代价相加，得到总代价。


   （4）、将上述3中模式的代价进行对比，取最小者为当前宏块的帧内预测模式。
