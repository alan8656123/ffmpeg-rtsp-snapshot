#include<stdio.h>
#include "libavdevice/avdevice.h"
#include "libavformat/avformat.h"
#include "libavfilter/avfilter.h"
#include "libavutil/avutil.h"
#include "libavutil/time.h"

#include "libswscale/swscale.h"
#include "libavutil/pixdesc.h"

int save_frame_as_jpeg(AVCodecContext *pCodecCtx, AVFrame *pFrame, const char* filename);
int capture_rtsp_image(const char *rtsp, const char* filename);
int save_frame_as_jpeg(AVCodecContext *pCodecCtx, AVFrame *pFrame, const char* filename) {
    FILE *JPEGFile;
    AVPacket packet ;
    int gotFrame;
    AVCodec *jpegCodec ;
    AVCodecContext *jpegContext;

    jpegCodec = avcodec_find_encoder(AV_CODEC_ID_MJPEG);
    if (!jpegCodec) {
        return -1;
    }
    jpegContext = avcodec_alloc_context3(jpegCodec);
    if (!jpegContext) {
        return -1;
    }

    jpegContext->pix_fmt =AV_PIX_FMT_YUVJ420P; //pCodecCtx->pix_fmt;
    jpegContext->height = pFrame->height;
    jpegContext->width = pFrame->width;
    jpegContext->time_base.den = 1;
    jpegContext->time_base.num = 25;

    if (avcodec_open2(jpegContext, jpegCodec, NULL) < 0) {
        return -1;
    }


    av_init_packet(&packet);
    packet.data = NULL;
    packet.size = 0;
    if (avcodec_encode_video2(jpegContext, &packet, pFrame, &gotFrame) < 0) {
        return -1;
    }

    JPEGFile = fopen(filename, "wb");
    fwrite(packet.data, 1, packet.size, JPEGFile);
    fclose(JPEGFile);

    av_free_packet(&packet);
    avcodec_close(jpegContext);
    return 0;
}


int capture_rtsp_image(const char *rtsp, const char* filename)
{
	unsigned int    i;
	int             ret;
	int 		av_result;
	int             video_st_index = -1;
	int             audio_st_index = -1;
	AVFormatContext *ifmt_ctx = NULL;
	AVPacket        pkt;
	AVStream        *st = NULL;
	char            errbuf[64];
	AVDictionary *optionsDict = NULL;


	int nRestart = 0;
	int videoindex = -1;
	int audioindex = -1;
	AVStream *pVst;
	AVCodecContext *pVideoCodecCtx = NULL;
	AVFrame         *pFrame = av_frame_alloc();
	int got_picture;
	AVCodec *pVideoCodec = NULL;

	av_register_all();                                                          // Register all codecs and formats so that they can be used.
	avformat_network_init();                                                    // Initialization of network components
	av_dict_set(&optionsDict, "rtsp_transport", "tcp", 0);                
	av_dict_set(&optionsDict, "stimeout", "2000000", 0);             
  
	if ((ret = avformat_open_input(&ifmt_ctx, rtsp, 0, &optionsDict)) < 0) {            // Open the input file for reading.
		printf("Could not open input file '%s' (error '%s')\n", rtsp, av_make_error_string(errbuf, sizeof(errbuf), ret));
		goto EXIT;
	}

	if ((ret = avformat_find_stream_info(ifmt_ctx, NULL)) < 0) {                // Get information on the input file (number of streams etc.).
		printf("Could not open find stream info (error '%s')\n", av_make_error_string(errbuf, sizeof(errbuf), ret));
		goto EXIT;
	}

	for (i = 0; i < ifmt_ctx->nb_streams; i++) {                                // dump information
		av_dump_format(ifmt_ctx, i, rtsp, 0);
	}

	for (i = 0; i < ifmt_ctx->nb_streams; i++) {                                // find video stream index
		st = ifmt_ctx->streams[i];
		switch (st->codec->codec_type) {
		case AVMEDIA_TYPE_AUDIO: audio_st_index = i; break;
		case AVMEDIA_TYPE_VIDEO: video_st_index = i; break;
		default: break;
		}
	}
	if (-1 == video_st_index) {
		printf("No H.264 video stream in the input file\n");
		goto EXIT;
	}

	av_init_packet(&pkt);                                                       // initialize packet.
	pkt.data = NULL;
	pkt.size = 0;
	while (1)
	{
		do {
			ret = av_read_frame(ifmt_ctx, &pkt);                                // read frames

			//decode stream
			if (!nRestart)
			{
				for (int i = 0; i < ifmt_ctx->nb_streams; i++)
				{
					if ((ifmt_ctx->streams[i]->codec->codec_type == AVMEDIA_TYPE_VIDEO) && (videoindex < 0))
					{
						videoindex = i;
					}
					if ((ifmt_ctx->streams[i]->codec->codec_type == AVMEDIA_TYPE_AUDIO) && (audioindex < 0))
					{
						audioindex = i;
					}
				}
				pVst = ifmt_ctx->streams[videoindex];
				pVideoCodecCtx = pVst->codec;
				pVideoCodec = avcodec_find_decoder(pVideoCodecCtx->codec_id);
				if (pVideoCodec == NULL)
					goto EXIT;
				if (avcodec_open2(pVideoCodecCtx, pVideoCodec, NULL) < 0)
					goto EXIT;
			}

			if (pkt.stream_index == videoindex)
			{
				fprintf(stdout, "pkt.size=%d,pkt.pts=%ld, pkt.data=0x%d.", pkt.size, pkt.pts, (unsigned int)pkt.data);
				av_result = avcodec_decode_video2(pVideoCodecCtx, pFrame, &got_picture, &pkt);

				if (got_picture)
				{
					fprintf(stdout, "decode one video frame!\n");
				}

				if (av_result < 0)
				{
					fprintf(stderr, "decode failed: inputbuf = 0x%hhn , input_framesize = %d\n", pkt.data, pkt.size);
					//return;
				}
				if (got_picture)
				{
					int save_ret;
					save_ret=save_frame_as_jpeg(pVideoCodecCtx,pFrame,filename);
					printf("save_ret:%d\n\n",save_ret);
					if(ret==0){
						return 0;
					}
					
				}
			}


		} while (ret == AVERROR(EAGAIN));

		if (ret < 0) {
			printf("Could not read frame ---(error '%s')\n", av_make_error_string(errbuf, sizeof(errbuf), ret));
			goto EXIT;
		}

		if (pkt.stream_index == video_st_index) {                               // video frame
			printf("Video Packet size = %d\n", pkt.size);
		}
		else if (pkt.stream_index == audio_st_index) {                         // audio frame
			printf("Audio Packet size = %d\n", pkt.size);
		}
		else {
			printf("Unknow Packet size = %d\n", pkt.size);
		}
		av_packet_unref(&pkt);
	}

EXIT:
	if (NULL != ifmt_ctx) {
		avformat_close_input(&ifmt_ctx);
		ifmt_ctx = NULL;
	}

	return -1;
}

int main(int argc, char* argv[])
{
capture_rtsp_image("rtsp://172.17.81.95:554/stream1","aewwwdinital.jpg");

return 0;
}
