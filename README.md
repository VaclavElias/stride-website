## Build instructions

### Windows

A precompiled Ruby and Jekyll installation is available in _jekyll.

Simply run `_deploy\run_local_website.bat` and open `http://localhost:4000`

### Mac

Install XCode with the App Store, then run in a command line:

```
xcode-select --install

sudo gem update --system
sudo gem install bundler -n /usr/local/bin

cd <xenko-website-dir>
bundle install --path ~/.gem
```

Then, you can run the website with:

```
~/.gem/ruby/2.0.0/bin/jekyll serve
```

and open `http://localhost:4000`

## Videos

### How to encode videos

Videos can be generated by many software in various formats & size, so they might end up being incompatible with web browsers or mobile, or simply be way too large.
It is better to stick to a format with low requirements such as H264 baseline profile (works almost everywhere).

To do so, process the file using [fmpeg](https://ffmpeg.org/download.html):
```
ffmpeg -i myvideo_original.mp4 -profile:v baseline -level 3.0 -an myvideo.mp4
```

Also, generate a static thumbnail so that people can preview it before downloading the video (very important on mobile):
```
ffmpeg -i myvideo.mp4 -vframes 1 -f image2 -y myvideo.jpg
```

TODO: Maybe we could provide a simple tool to do that without using command line.

### How to embed video in blog post

Please make a subfolder for each blog post in `images/blog` so that it doesn't become too crowded.

The "video" tag must be wrapped to tag "p" 

Here is the syntax to include a video in a blog post (please replace `2016-12-01-game-templates/templateFPS` with the actual directory and filename):

```
<p>
  <div class="embed-responsive-anyratio"><div class="video-play-button"></div>
    <video autoplay loop class="responsive-video" poster="../../images/blog/2016-12-01-game-templates/templateFPS.jpg" onplay="feature_video_onplayevent" onpause="feature_video_onpauseevent">
      <source src="../../images/blog/2016-12-01-game-templates/templateFPS.mp4" type="video/mp4">
    </video>
  </div>
</p>
```