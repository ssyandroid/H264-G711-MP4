# Android h264视频与G711转码合并MP4文件

## 首先转码音频 ##
音频转码使用EasyAACEncoder 此是EasyDarwin开源流媒体服务团队整理、开发的一款音频转码到AAC的工具库编译的转码库。
具体库怎么编译请看我的另一篇文章：[Android G711(PCMA/PCMU)、G726、PCM音频转码到AAC](http://blog.csdn.net/susanyuanaijia/article/details/53141674)
具体Android怎么使用该库方法

使用方法示例：

		/**
	     * g711a转码为aac
	     *
	     * @param cachePath  文件路径
	     * @param inFilename 输入音频文件名
	     * @param outAacname 输出音频文件名
	     */
	    public void g711toAAC(final String cachePath, final String inFilename, final String outAacname) {
	        new Thread(new Runnable() {
	            @Override
	            public void run() {
	                JNIAACEncode.init(JNIAACEncode.Law_ALaw);
	                File audiofile = new File(cachePath, inFilename);
	                File audiofile2 = new File(cachePath, outAacname);
	                int encode = JNIAACEncode.encode(audiofile.getAbsolutePath(), audiofile2.getAbsolutePath());
	                if (encode == 0) {
	                    ToastUtil.showMessage("声音转码成功");
	               }else{
	                    ToastUtil.showMessage("声音转码失败");
	                }
	            }
	        }).start();
	    }

注意此处需要编译的so库文件，具体文件可以自行编译或者请查看我上面的文章里下载编译好的库。
然后新建个Java类例如：

		public class JNIAACEncode {
	
			public static final int Law_ULaw = 0;/**< U law */
			public static final int Law_ALaw = 1;/**< A law */
			public static final int Law_PCM16 = 2;/**< 16 bit uniform PCM values. 原始 pcm 数据 */  
			public static final int Law_G726 = 3;/**< G726 */
			
			//首先进行实例化audioCodec为上面声明的音频类型
			public static native void init(int audioCodec);
			//原音频文件路径与转码后的音频文件路径
			public static native int encode(String infilename, String outAacname);
			
			static {
				System.loadLibrary("AACEncode");
			}
		}

## h264与aac合并MP4 ##
上面方法可以把g711音频文件转码成aac文件，然后那h264文件与aac文件进行合并MP4文件，转码方法有很多，网上教程也很多，不过好多都要进行编译so库文件，整起来很是费劲而且导致项目很大。

但是java有工具进行转码，那就是isoparser，该为jar文件直接放到lib里即可使用，简直方便的不要不要，而且很小！比那编译的库小很多的！使用也很简单哈。
至于isoparser可以自行下载，不想动手的可以[点我下载isoparser-1.1.21.jar  ](http://download.csdn.net/detail/susanyuanaijia/9579146)

然后怎么使用呢，请看本人的使用方法：

		/**
	     * h264视频进行转码为MP4
	     * <p/>
	     * Creates a new Track object from a raw H264 source (DataSource fc). Whenever the timescale and frametick are set to negative value (e.g. -1) the H264TrackImpl tries to detect the frame rate. Typically values for timescale and frametick are:
	     * 23.976 FPS: timescale = 24000; frametick = 1001
	     * 25 FPS: timescale = 25; frametick = 1
	     * 29.97 FPS: timescale = 30000; frametick = 1001
	     * 30 FPS: timescale = 30; frametick = 1
	     * Parameters:
	     * fc - the source file of the H264 samples
	     * lang - language of the movie (in doubt: use "eng")
	     * timescale - number of time units (ticks) in one second
	     * frametick - number of time units (ticks) that pass while showing exactly one frame
	     *
	     * @param path      h264文件路径
	     * @param audioPath aac声音文件路径
	     * @param videoPath 保存视频文件路径
	     * @param fileName  保存视频文件名
	     */
	    public void saveVideo(final String path, final String audioPath, final String videoPath, final String fileName) {
	        new Thread(new Runnable() {
	            @Override
	            public void run() {
	                try {
	                    DataSource video_file = new FileDataSourceImpl(path);
	                    H264TrackImpl h264Track = new H264TrackImpl(video_file, "eng", fps, 1); 
	                    Movie movie = new Movie();
	                    movie.addTrack(h264Track);
	
	                    File audiofile = new File(audioPath);
						//如果音频文件不存在，则不合成音频，存在则加上音频
	                    if (audiofile.exists()) {
	                        AACTrackImpl aacTrack = null;
	                        try {
	                            DataSource audio_file = new FileDataSourceImpl(audioPath);
	                            aacTrack = new AACTrackImpl(audio_file);
	                        } catch (IOException e) {
	                            e.printStackTrace();
	                        }
	                        movie.addTrack(aacTrack);
	                    }

	                    File myCaptureFile = new File(videoPath);
	                    if (!myCaptureFile.exists() ) {
	                        myCaptureFile.mkdirs();
	                    }

	                    Container out = new DefaultMp4Builder().build(movie);
	                    final File file = new File(myCaptureFile, fileName);
	                    FileOutputStream fos = new FileOutputStream(file);
	                    out.writeContainer(fos.getChannel());
	                    fos.close();

	                    //LogMsg.i(TAG, "视频转码完成");

	                    runOnUiThread(new Runnable() {
	                        @Override
	                        public void run() {
	                            // 最后通知图库更新
	                            sendBroadcast(new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE, Uri.fromFile(file)));
	                        }
	                    });
	                } catch (Exception e) {
	                    e.printStackTrace();
	                }
	            }
	        }).start();
	    }

上面方法的fps要根据你合成的视频具体来设置。
可以参考一下：
23.976 FPS: timescale = 24000; frametick = 1001
25 FPS: timescale = 25; frametick = 1
29.97 FPS: timescale = 30000; frametick = 1001
30 FPS: timescale = 30; frametick = 1

这样就可以完成视频的合成啦，当然别忘了加上需要的权限比如读写权限。
是不是很方便啦啦，还有就是如果视频合成后播放如果没有到播放进度最后就播放完了，那是给关键帧有关系，如果你的视频最后没有关键帧话可能导致后面一段不能播放。




