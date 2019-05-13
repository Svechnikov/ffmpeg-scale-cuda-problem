FFmpeg scale_cuda problem
---

# <a name="about"></a>Description

When hardware transcoding with CUDA and using filter `scale_cuda` the output videos may be partially cropped out either on the right or on the bottom.
For instance, when scaling HD video (1920x1080) down to HD-ready (1280x720) the resulting video will be cropped out by 16 pixels on the bottom (here's a [screenshot with the problem](https://raw.githubusercontent.com/Svechnikov/ffmpeg-scale-cuda-problem/master/screenshots/720/002.png) and here's [what it should look like](https://raw.githubusercontent.com/Svechnikov/ffmpeg-scale-cuda-problem/master/screenshots/720-fixed/002.png)).
Another example: when scaling down to SD (720x576) the resulting video will be cropped out by 16 pixels on the right ([screenshot with the problem](https://raw.githubusercontent.com/Svechnikov/ffmpeg-scale-cuda-problem/master/screenshots/576/002.png), [what it should look like](https://raw.githubusercontent.com/Svechnikov/ffmpeg-scale-cuda-problem/master/screenshots/576-fixed/002.png)).

**Demonstration of the problem when scaling to 720x576**
![Demonstration of the problem](https://raw.githubusercontent.com/Svechnikov/ffmpeg-scale-cuda-problem/master/screenshots/576/002.png)

**What it should look like**
![What it should look like](https://raw.githubusercontent.com/Svechnikov/ffmpeg-scale-cuda-problem/master/screenshots/576-fixed/002.png)

This repository is intended to prove that such problem exists and to show, how to solve it.

# <a name="preparing"></a>Preparation steps

In order to reproduce the problem, you should have:

1. Linux environment;
2. Nvidia graphic card with cuvid/nvenc support and latest drivers installed (I tested on GTX 1080, GTX 1050 and 430.09 drivers);
3. Docker installed (for building and running FFmpeg). Using docker is not mandatory, but recommended, as it simplifies the process of building;
4. [Nvidia docker runtime](https://github.com/NVIDIA/nvidia-docker/) installed (if using docker).

Building the image:

`docker build images -f images/Dockerfile -t ffmpeg-scale-test`

# <a name="running"></a>Reproducing the problem

I prepared a sample video for which the undesired cropping should be easily visually detected. It's just [a simple mpegts h264 full HD progressive video](https://raw.githubusercontent.com/Svechnikov/ffmpeg-scale-cuda-problem/master/samples/input.ts).

Let's try to scale it down to HD-ready:

`docker run -it -v $(pwd)/samples:/samples --rm ffmpeg-scale-test ffmpeg -y -hwaccel cuvid -c:v h264_cuvid -i /samples/input.ts -map 0:0 -c:v h264_nvenc -vf scale_cuda=1280:720 /samples/720.ts`

After the ffmpeg process completes try playing the output file `samples/720.ts`.
You will see, that the "1080" line is cropped on the bottom to "108".

You can examine each frame as a jpg image:

`docker run -it -v $(pwd)/samples:/samples --rm ffmpeg-scale-test sh -c 'mkdir -p /samples/720-frames && rm -rf /samples/720-frames/* && ffmpeg -i /samples/720.ts /samples/720-frames/%03d.jpg'`

You will see, that the first frame is correct, but the consecutive frames are cropped out.

Now let's try to scale the sample down to SD resolution:

`docker run -it -v $(pwd)/samples:/samples --rm ffmpeg-scale-test ffmpeg -y -hwaccel cuvid -c:v h264_cuvid -i /samples/input.ts -map 0:0 -c:v h264_nvenc -vf scale_cuda=720:576 /samples/576.ts`

After the ffmpeg process completes try playing the output file `samples/576.ts`.
You will see, that the "1920x1080" line is completely cropped out on the right.

You can examine each frame as a jpg image:

`docker run -it -v $(pwd)/samples:/samples --rm ffmpeg-scale-test sh -c 'mkdir -p /samples/576-frames && rm -rf /samples/576-frames/* && ffmpeg -i /samples/576.ts /samples/576-frames/%03d.jpg'`

You will see, that the first frame is correct, but the consecutive frames are cropped out.

# <a name="running"></a>Fixing the problem

`AVHWFramesContext` has [aligned width and height](https://github.com/FFmpeg/FFmpeg/blob/master/libavfilter/vf_scale_cuda.c#L162).
When initializing a new `AVFrame`, it [receives](https://github.com/FFmpeg/FFmpeg/blob/master/libavfilter/vf_scale_cuda.c#L462) these aligned values, which leads to incorrect scaling (so, for instance, 720 becomes 736).
When running [CUDA code](https://github.com/FFmpeg/FFmpeg/blob/master/libavfilter/vf_scale_cuda.cu#L27), this invalid value leads to invalid calculations.
As a fix we can overwrite the dimensions to original values right after `av_hwframe_get_buffer`.

I prepared a patch [images/fix.patch](https://github.com/Svechnikov/ffmpeg-scale-cuda-problem/blob/master/images/fix.patch). You can test the patch using a special docker-image:

`docker build images -f images/Dockerfile.fixed -t ffmpeg-scale-test`

If you run the commands again, all the output videos will be correct.
