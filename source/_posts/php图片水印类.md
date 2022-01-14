---
title: php图片水印类
tags: []
date: 2015-06-25 14:45:00
---
php中图片水印处理类

<!-- more -->

<div class="cnblogs_code">
<pre>&lt;?<span style="color: #000000;">php
</span><span style="color: #0000ff;">class</span><span style="color: #000000;"> Image {
    </span><span style="color: #0000ff;">private</span> <span style="color: #800080;">$path</span><span style="color: #000000;">;
    </span><span style="color: #008000;">//</span><span style="color: #008000;">构造方法用来对图片所在位置进行初使化</span>
    <span style="color: #0000ff;">function</span> __construct(<span style="color: #800080;">$path</span>="./"<span style="color: #000000;">){
        </span><span style="color: #800080;">$this</span>-&gt;path=<span style="color: #008080;">rtrim</span>(<span style="color: #800080;">$path</span>, "/")."/"<span style="color: #000000;">;
        </span><span style="color: #800080;">$this</span> -&gt; path = ''<span style="color: #000000;">;
    }
    </span><span style="color: #008000;">/*</span><span style="color: #008000;"> 对图片进行缩放
     *
     * 参数$name: 是需要处理的图片名称
     * 参数$width:是缩放后的宽度
     * 参数$height:是缩放后的高度
     * 参数$qz: 是新图片的名称前缀
     * 返回值:就是缩放后的图片名称，失败则返回false
     *
     </span><span style="color: #008000;">*/</span>
    <span style="color: #0000ff;">function</span> thumb(<span style="color: #800080;">$name</span>, <span style="color: #800080;">$width</span>, <span style="color: #800080;">$height</span>, <span style="color: #800080;">$qz</span>="th_"<span style="color: #000000;">){
        </span><span style="color: #008000;">//</span><span style="color: #008000;">获取图片信息</span>
        <span style="color: #800080;">$imgInfo</span>=<span style="color: #800080;">$this</span>-&gt;getInfo(<span style="color: #800080;">$name</span>); <span style="color: #008000;">//</span><span style="color: #008000;">图片的宽度，高度，类型
        //获取图片资源, 各种类型的图片都可以创建资源 jpg, gif, png</span>
        <span style="color: #800080;">$srcImg</span>=<span style="color: #800080;">$this</span>-&gt;getImg(<span style="color: #800080;">$name</span>, <span style="color: #800080;">$imgInfo</span><span style="color: #000000;">);
        </span><span style="color: #008000;">//</span><span style="color: #008000;">获取计算图片等比例之后的大小, $size["width"], $size["height"]</span>
        <span style="color: #800080;">$size</span>=<span style="color: #800080;">$this</span>-&gt;getNewSize(<span style="color: #800080;">$name</span>, <span style="color: #800080;">$width</span>, <span style="color: #800080;">$height</span>, <span style="color: #800080;">$imgInfo</span><span style="color: #000000;">);
        </span><span style="color: #008000;">//</span><span style="color: #008000;">获取新的图片资源, 处理一下gif透明背景</span>
        <span style="color: #800080;">$newImg</span>=<span style="color: #800080;">$this</span>-&gt;kidOfImage(<span style="color: #800080;">$srcImg</span>, <span style="color: #800080;">$size</span>, <span style="color: #800080;">$imgInfo</span><span style="color: #000000;">);
        </span><span style="color: #008000;">//</span><span style="color: #008000;">另存为一个新的图片，返回新的缩放后的图片名称    </span>
        <span style="color: #0000ff;">return</span> <span style="color: #800080;">$this</span>-&gt;createNewImage(<span style="color: #800080;">$newImg</span>, <span style="color: #800080;">$qz</span>.<span style="color: #800080;">$name</span>, <span style="color: #800080;">$imgInfo</span><span style="color: #000000;">);    
    }

    </span><span style="color: #0000ff;">private</span> <span style="color: #0000ff;">function</span> createNewImage(<span style="color: #800080;">$newImg</span>, <span style="color: #800080;">$newName</span>, <span style="color: #800080;">$imgInfo</span><span style="color: #000000;">){
        </span><span style="color: #0000ff;">switch</span>(<span style="color: #800080;">$imgInfo</span>["type"<span style="color: #000000;">]){
        </span><span style="color: #0000ff;">case</span> 1:<span style="color: #008000;">//</span><span style="color: #008000;">gif</span>
            <span style="color: #008080;">Header</span>("Content-type: image/gif"<span style="color: #000000;">);
            </span><span style="color: #800080;">$result</span>=imageGif(<span style="color: #800080;">$newImg</span><span style="color: #000000;">);
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">case</span> 2:<span style="color: #008000;">//</span><span style="color: #008000;">jpg</span>
            <span style="color: #008080;">Header</span>("Content-type: image/jpg"<span style="color: #000000;">);
            </span><span style="color: #800080;">$result</span>=imageJPEG(<span style="color: #800080;">$newImg</span><span style="color: #000000;">);
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">case</span> 3:<span style="color: #008000;">//</span><span style="color: #008000;">png</span>
            <span style="color: #008080;">Header</span>("Content-type: image/png"<span style="color: #000000;">);
            </span><span style="color: #800080;">$return</span>=imagepng(<span style="color: #800080;">$newImg</span><span style="color: #000000;">);
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        }
        imagedestroy(</span><span style="color: #800080;">$newImg</span><span style="color: #000000;">);
        </span><span style="color: #0000ff;">return</span> <span style="color: #800080;">$newName</span><span style="color: #000000;">;
    }

    </span><span style="color: #0000ff;">private</span> <span style="color: #0000ff;">function</span> kidOfImage(<span style="color: #800080;">$srcImg</span>, <span style="color: #800080;">$size</span>, <span style="color: #800080;">$imgInfo</span><span style="color: #000000;">){
        </span><span style="color: #800080;">$newImg</span>=imagecreatetruecolor(<span style="color: #800080;">$size</span>["width"], <span style="color: #800080;">$size</span>["height"<span style="color: #000000;">]);

        </span><span style="color: #800080;">$otsc</span>=imagecolortransparent(<span style="color: #800080;">$srcImg</span><span style="color: #000000;">);

        </span><span style="color: #0000ff;">if</span>(<span style="color: #800080;">$otsc</span> &gt;=0 &amp;&amp; <span style="color: #800080;">$otsc</span> &lt;= imagecolorstotal(<span style="color: #800080;">$srcImg</span><span style="color: #000000;">)){
            </span><span style="color: #800080;">$tran</span>=imagecolorsforindex(<span style="color: #800080;">$srcImg</span>, <span style="color: #800080;">$otsc</span><span style="color: #000000;">);

            </span><span style="color: #800080;">$newt</span>=imagecolorallocate(<span style="color: #800080;">$newImg</span>, <span style="color: #800080;">$tran</span>["red"], <span style="color: #800080;">$tran</span>["green"], <span style="color: #800080;">$tran</span>["blue"<span style="color: #000000;">]);

            imagefill(</span><span style="color: #800080;">$newImg</span>, 0, 0, <span style="color: #800080;">$newt</span><span style="color: #000000;">);

            imagecolortransparent(</span><span style="color: #800080;">$newImg</span>, <span style="color: #800080;">$newt</span><span style="color: #000000;">);
        }

        imagecopyresized(</span><span style="color: #800080;">$newImg</span>, <span style="color: #800080;">$srcImg</span>, 0, 0, 0, 0, <span style="color: #800080;">$size</span>["width"], <span style="color: #800080;">$size</span>["height"], <span style="color: #800080;">$imgInfo</span>["width"], <span style="color: #800080;">$imgInfo</span>["height"<span style="color: #000000;">]);

        imagedestroy(</span><span style="color: #800080;">$srcImg</span><span style="color: #000000;">);

        </span><span style="color: #0000ff;">return</span> <span style="color: #800080;">$newImg</span><span style="color: #000000;">;
    }

    </span><span style="color: #0000ff;">private</span> <span style="color: #0000ff;">function</span> getNewSize(<span style="color: #800080;">$name</span>, <span style="color: #800080;">$width</span>, <span style="color: #800080;">$height</span>, <span style="color: #800080;">$imgInfo</span><span style="color: #000000;">){
        </span><span style="color: #800080;">$size</span>["width"]=<span style="color: #800080;">$imgInfo</span>["width"<span style="color: #000000;">];
        </span><span style="color: #800080;">$size</span>["height"]=<span style="color: #800080;">$imgInfo</span>["height"<span style="color: #000000;">];

        </span><span style="color: #008000;">//</span><span style="color: #008000;">缩放的宽度如果比原图小才重新设置宽度</span>
        <span style="color: #0000ff;">if</span>(<span style="color: #800080;">$width</span> &lt; <span style="color: #800080;">$imgInfo</span>["width"<span style="color: #000000;">]){
            </span><span style="color: #800080;">$size</span>["width"]=<span style="color: #800080;">$width</span><span style="color: #000000;">;
        }
        </span><span style="color: #008000;">//</span><span style="color: #008000;">缩放的高度如果比原图小才重新设置高度</span>
        <span style="color: #0000ff;">if</span>(<span style="color: #800080;">$height</span> &lt; <span style="color: #800080;">$imgInfo</span>["height"<span style="color: #000000;">]){
            </span><span style="color: #800080;">$size</span>["height"]=<span style="color: #800080;">$height</span><span style="color: #000000;">;
        }

        </span><span style="color: #008000;">//</span><span style="color: #008000;">图片等比例缩放的算法</span>
        <span style="color: #0000ff;">if</span>(<span style="color: #800080;">$imgInfo</span>["width"]*<span style="color: #800080;">$size</span>["width"] &gt; <span style="color: #800080;">$imgInfo</span>["height"] * <span style="color: #800080;">$size</span>["height"<span style="color: #000000;">]){
            </span><span style="color: #800080;">$size</span>["height"]=<span style="color: #008080;">round</span>(<span style="color: #800080;">$imgInfo</span>["height"]*<span style="color: #800080;">$size</span>["width"]/<span style="color: #800080;">$imgInfo</span>["width"<span style="color: #000000;">]);
        }</span><span style="color: #0000ff;">else</span><span style="color: #000000;">{
            </span><span style="color: #800080;">$size</span>["width"]=<span style="color: #008080;">round</span>(<span style="color: #800080;">$imgInfo</span>["width"]*<span style="color: #800080;">$size</span>["height"]/<span style="color: #800080;">$imgInfo</span>["height"<span style="color: #000000;">]);
        }

        </span><span style="color: #0000ff;">return</span> <span style="color: #800080;">$size</span><span style="color: #000000;">;

    }

    </span><span style="color: #0000ff;">public</span> <span style="color: #0000ff;">function</span> getInfo(<span style="color: #800080;">$name</span><span style="color: #000000;">){
        </span><span style="color: #800080;">$data</span>=<span style="color: #008080;">getImageSize</span>(<span style="color: #800080;">$this</span>-&gt;path.<span style="color: #800080;">$name</span><span style="color: #000000;">);

        </span><span style="color: #800080;">$imageInfo</span>["width"]=<span style="color: #800080;">$data</span>[0<span style="color: #000000;">];
        </span><span style="color: #800080;">$imageInfo</span>["height"]=<span style="color: #800080;">$data</span>[1<span style="color: #000000;">];
        </span><span style="color: #800080;">$imageInfo</span>["type"]=<span style="color: #800080;">$data</span>[2<span style="color: #000000;">];

        </span><span style="color: #0000ff;">return</span> <span style="color: #800080;">$imageInfo</span><span style="color: #000000;">;
    }

    </span><span style="color: #0000ff;">private</span> <span style="color: #0000ff;">function</span> getImg(<span style="color: #800080;">$name</span>, <span style="color: #800080;">$imgInfo</span><span style="color: #000000;">){
        </span><span style="color: #800080;">$srcPic</span>=<span style="color: #800080;">$this</span>-&gt;path.<span style="color: #800080;">$name</span><span style="color: #000000;">;

        </span><span style="color: #0000ff;">switch</span>(<span style="color: #800080;">$imgInfo</span>["type"<span style="color: #000000;">]){
        </span><span style="color: #0000ff;">case</span> 1: <span style="color: #008000;">//</span><span style="color: #008000;">gif</span>
            <span style="color: #800080;">$img</span>=imagecreatefromgif(<span style="color: #800080;">$srcPic</span><span style="color: #000000;">);
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">case</span> 2: <span style="color: #008000;">//</span><span style="color: #008000;">jpg</span>
            <span style="color: #800080;">$img</span>=imageCreatefromjpeg(<span style="color: #800080;">$srcPic</span><span style="color: #000000;">);
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">case</span> 3: <span style="color: #008000;">//</span><span style="color: #008000;">png</span>
            <span style="color: #800080;">$img</span>=imageCreatefrompng(<span style="color: #800080;">$srcPic</span><span style="color: #000000;">);
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">default</span>:
            <span style="color: #0000ff;">return</span> <span style="color: #0000ff;">false</span><span style="color: #000000;">;

        }

        </span><span style="color: #0000ff;">return</span> <span style="color: #800080;">$img</span><span style="color: #000000;">;
    }
    </span><span style="color: #008000;">/*</span><span style="color: #008000;"> 功能：为图片加水印图片
     * 参数$groundName: 背景图片，即需要加水印的图片
     * 参数$waterName: 水钱图片
     * 参数#aterPost：水印位置， 10种状态， 
     *  0为随机位置
     *
     *  1\. 为顶端居左  2\. 为顶端居中  3 为顶端居右
     *  4  为中部居左  5\. 为中部居中  6 为中部居右
     *  7 . 为底端居左 8\. 为底端居中， 9\. 为底端居右
     *
     * 参数$qz : 是加水印后的图片名称前缀
     * 返回值：就是处理后图片的名称
     *
     </span><span style="color: #008000;">*/</span>
    <span style="color: #0000ff;">function</span> waterMark(<span style="color: #800080;">$groundName</span>, <span style="color: #800080;">$waterName</span>, <span style="color: #800080;">$waterPos</span>=0, <span style="color: #800080;">$qz</span>="wa_"<span style="color: #000000;">){

        </span><span style="color: #800080;">$groundInfo</span>=<span style="color: #800080;">$this</span>-&gt;getInfo(<span style="color: #800080;">$groundName</span><span style="color: #000000;">);
        </span><span style="color: #800080;">$waterInfo</span>=<span style="color: #800080;">$this</span>-&gt;getInfo(<span style="color: #800080;">$waterName</span><span style="color: #000000;">);
        </span><span style="color: #008000;">//</span><span style="color: #008000;">水印的位置</span>
        <span style="color: #0000ff;">if</span>(!<span style="color: #800080;">$pos</span>=<span style="color: #800080;">$this</span>-&gt;position(<span style="color: #800080;">$groundInfo</span>, <span style="color: #800080;">$waterInfo</span>, <span style="color: #800080;">$waterPos</span><span style="color: #000000;">)){
            </span><span style="color: #0000ff;">return</span> "Picture too small!"<span style="color: #000000;">;
        }

        </span><span style="color: #800080;">$groundImg</span>=<span style="color: #800080;">$this</span>-&gt;getImg(<span style="color: #800080;">$groundName</span>, <span style="color: #800080;">$groundInfo</span><span style="color: #000000;">);
        </span><span style="color: #800080;">$waterImg</span>=<span style="color: #800080;">$this</span>-&gt;getImg(<span style="color: #800080;">$waterName</span>, <span style="color: #800080;">$waterInfo</span><span style="color: #000000;">);

        </span><span style="color: #800080;">$groundImg</span>=<span style="color: #800080;">$this</span>-&gt;copyImage(<span style="color: #800080;">$groundImg</span>, <span style="color: #800080;">$waterImg</span>, <span style="color: #800080;">$pos</span>, <span style="color: #800080;">$waterInfo</span><span style="color: #000000;">);

        </span><span style="color: #0000ff;">return</span> <span style="color: #800080;">$this</span>-&gt;createNewImage(<span style="color: #800080;">$groundImg</span>, <span style="color: #800080;">$qz</span>.<span style="color: #800080;">$groundName</span>, <span style="color: #800080;">$groundInfo</span><span style="color: #000000;">);

    }

    </span><span style="color: #0000ff;">private</span> <span style="color: #0000ff;">function</span> copyImage(<span style="color: #800080;">$groundImg</span>, <span style="color: #800080;">$waterImg</span>, <span style="color: #800080;">$pos</span>, <span style="color: #800080;">$waterInfo</span><span style="color: #000000;">){
        imagecopy(</span><span style="color: #800080;">$groundImg</span>, <span style="color: #800080;">$waterImg</span>, <span style="color: #800080;">$pos</span>["posX"], <span style="color: #800080;">$pos</span>["posY"], 0, 0, <span style="color: #800080;">$waterInfo</span>["width"], <span style="color: #800080;">$waterInfo</span>["height"<span style="color: #000000;">]);
        imagedestroy(</span><span style="color: #800080;">$waterImg</span><span style="color: #000000;">);

        </span><span style="color: #0000ff;">return</span> <span style="color: #800080;">$groundImg</span><span style="color: #000000;">;
    }

    </span><span style="color: #0000ff;">private</span> <span style="color: #0000ff;">function</span> position(<span style="color: #800080;">$groundInfo</span>, <span style="color: #800080;">$waterInfo</span>, <span style="color: #800080;">$waterPos</span><span style="color: #000000;">){
        </span><span style="color: #008000;">//</span><span style="color: #008000;">需要背景比水印图片大</span>
        <span style="color: #0000ff;">if</span>((<span style="color: #800080;">$groundInfo</span>["width"]&lt; <span style="color: #800080;">$waterInfo</span>["width"]) ||(<span style="color: #800080;">$groundInfo</span>["height"] &lt; <span style="color: #800080;">$waterInfo</span>["height"<span style="color: #000000;">])){
            </span><span style="color: #0000ff;">return</span> <span style="color: #0000ff;">false</span><span style="color: #000000;">;
        }

        </span><span style="color: #0000ff;">switch</span>(<span style="color: #800080;">$waterPos</span><span style="color: #000000;">){
        </span><span style="color: #0000ff;">case</span> 1:
            <span style="color: #800080;">$posX</span>=0<span style="color: #000000;">;
            </span><span style="color: #800080;">$posY</span>=0<span style="color: #000000;">;
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">case</span> 2:
            <span style="color: #800080;">$posX</span>=(<span style="color: #800080;">$groundInfo</span>["width"]-<span style="color: #800080;">$waterInfo</span>["width"])/2<span style="color: #000000;">;
            </span><span style="color: #800080;">$posY</span>=0<span style="color: #000000;">;
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">case</span> 3:
            <span style="color: #800080;">$posX</span>=<span style="color: #800080;">$groundInfo</span>["width"]-<span style="color: #800080;">$waterInfo</span>["width"<span style="color: #000000;">];
            </span><span style="color: #800080;">$posY</span>=0<span style="color: #000000;">;
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">case</span> 4:
            <span style="color: #800080;">$posX</span>=0<span style="color: #000000;">;
            </span><span style="color: #800080;">$posY</span>=(<span style="color: #800080;">$groundInfo</span>["height"]-<span style="color: #800080;">$waterInfo</span>["height"]) /2<span style="color: #000000;">;
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">case</span> 5:
            <span style="color: #800080;">$posX</span>=(<span style="color: #800080;">$groundInfo</span>["width"]-<span style="color: #800080;">$waterInfo</span>["width"])/2<span style="color: #000000;">;
            </span><span style="color: #800080;">$posY</span>=(<span style="color: #800080;">$groundInfo</span>["height"]-<span style="color: #800080;">$waterInfo</span>["height"]) /2<span style="color: #000000;">;
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">case</span> 6:
            <span style="color: #800080;">$posX</span>=<span style="color: #800080;">$groundInfo</span>["width"]-<span style="color: #800080;">$waterInfo</span>["width"<span style="color: #000000;">];
            </span><span style="color: #800080;">$posY</span>=(<span style="color: #800080;">$groundInfo</span>["height"]-<span style="color: #800080;">$waterInfo</span>["height"]) /2<span style="color: #000000;">;
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">case</span> 7:
            <span style="color: #800080;">$posX</span>=0<span style="color: #000000;">;
            </span><span style="color: #800080;">$posY</span>=<span style="color: #800080;">$groundInfo</span>["height"]-<span style="color: #800080;">$waterInfo</span>["height"<span style="color: #000000;">];
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">case</span> 8:
            <span style="color: #800080;">$posX</span>=(<span style="color: #800080;">$groundInfo</span>["width"]-<span style="color: #800080;">$waterInfo</span>["width"])/2<span style="color: #000000;">;
            </span><span style="color: #800080;">$posY</span>=<span style="color: #800080;">$groundInfo</span>["height"]-<span style="color: #800080;">$waterInfo</span>["height"<span style="color: #000000;">];
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">case</span> 9:
            <span style="color: #800080;">$posX</span>=<span style="color: #800080;">$groundInfo</span>["width"]-<span style="color: #800080;">$waterInfo</span>["width"<span style="color: #000000;">];
            </span><span style="color: #800080;">$posY</span>=<span style="color: #800080;">$groundInfo</span>["height"]-<span style="color: #800080;">$waterInfo</span>["height"<span style="color: #000000;">];
            </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        </span><span style="color: #0000ff;">case</span> 0:
        <span style="color: #0000ff;">default</span>:
        <span style="color: #800080;">$posX</span>=<span style="color: #008080;">rand</span>(0, (<span style="color: #800080;">$groundInfo</span>["width"]-<span style="color: #800080;">$waterInfo</span>["width"<span style="color: #000000;">]));
        </span><span style="color: #800080;">$posY</span>=<span style="color: #008080;">rand</span>(0, (<span style="color: #800080;">$groundInfo</span>["height"]-<span style="color: #800080;">$waterInfo</span>["height"<span style="color: #000000;">]));
        </span><span style="color: #0000ff;">break</span><span style="color: #000000;">;
        }

        </span><span style="color: #0000ff;">return</span> <span style="color: #0000ff;">array</span>("posX"=&gt;<span style="color: #800080;">$posX</span>, "posY"=&gt;<span style="color: #800080;">$posY</span><span style="color: #000000;">);
    }

}</span></pre>
</div>

