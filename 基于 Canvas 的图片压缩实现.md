# 基于 Canvas 的图片压缩实现

## 前言

随着手机像素越来越高，拍摄图片尺寸也越来越大，一张图片大小约为 10M，前不久小米官微晒出了其拍照样张，并表示该照片分辨率高达12032 x 9024，单张文件超过 40MB。面对动辄几十兆的图片文件，前端在弱网场景下上传图片时往往会出现用户等待时间过长或上传失败的情况，为了保证良好的用户体验，前端对图片进行压缩来降低网络通信量成为了关键。图像质量与文件大小呈正相关关系，若压缩率太高则会造成图像质量过低，用户感知体验差；而压缩率太低则不能起到降低上传时间的效果。基于此痛点，本文介绍了一种基于 Canvas 的图片压缩方案原理及其实现，能够很好地平衡压缩率与图像质量。

## 目标

在确保图像质量无明显降低的情况下，压缩图片大小达80%以上，减少用户上传等待时长、图片加载时间，提升用户体验。

## 压缩思路

基于Canvas的绘图能力，通过调整图片的分辨率或者绘图质量来达到图片压缩的效果，实现思路如下：

1. 获取上传 Input 中的图片对象 File。

``` 
<template>
  <div class="box">
    <div class="box-btn" @click="chooseFile">选择图片</div>
    <input
      type="file"
      ref="imgInput"
      name="file"
      accept="image/*"
      style="display:none"
      @change="handelChangeFile"
    />
    <div class="box-img">
      <img class="box-img-left" :src="originImgSrc" alt />
      <img class="box-img-right" :src="imgSrc" alt />
    </div>
  </div>
</template>
```

2. 将图片转换成 base64 格式。
3. base64 编码的图片通过 Canvas 转换压缩，这里会用到的 Canvas 的 drawImage 以及 toDataURL 这两个 API，一个调节图片的分辨率，一个是调节图片压缩质量并且输出。
4. 转换后的图片生成对应的新图片，然后输出。

### 压缩关键API：

toBlob(callback, [type], [encoderOptions]) 参数 encoderOptions 用于针对 image/jpeg 或 image/webp 格式的图片进行输出图片的质量设置。
toDataURL(type, encoderOptions) 参数 encoderOptions 在指定图片格式为 image/jpeg 或 image/webp 的情况下，可以从 0 到 1 的区间内选择图片的质量。

### CANVAS压缩优缺点：

优点：实现简单，参数可以配置化，自定义图片的尺寸，图片质量，指定区域裁剪等等。
缺点：只有 jpeg 、webp 支持原图尺寸下图片质量的调整来达到压缩图片的效果，其他图片格式，仅能通过调节尺寸来实现。

### 完整代码

``` js
//compressionImg.js
/**
 * 上传🆓压缩
 * @param {*} file 必填参数｜压缩文件
 * @param {*} limit 必填参数｜图片大小超过设置的limit值才会进行压缩
 * @param {*} maxWidth 可选参数｜最大宽度限制
 * @param {*} maxHeight 可选参数｜最大高度限制
 * @param {*} quality 可选参数｜画面质量
 */
const compressionImg = (file, limit, maxWidth, maxHeight, quality) => {
    if (file.size < limit) {
        return new Promise((resolve) => {
            resolve(file);
        });
    } else {
        const compressionImage = new Promise((resolve, reject) => {
            const fileName = file.name;
            const reader = new FileReader();
            reader.readAsDataURL(file);
            reader.onload = () => {
                const img = new Image();
                img.src = reader.result;
                img.onload = () => {
                    const elem = document.createElement("canvas");
                    // 图片原始尺寸
                    const originWidth = img.width;
                    const originHeight = img.height;
                    console.log("压缩前", originHeight, originWidth);
                    // 最大尺寸限制
                    maxWidth = maxWidth || 1280;
                    maxHeight = maxHeight || 1280;
                    quality = quality || 0.8;
                    // 目标尺寸
                    let targetWidth = originWidth;
                    let targetHeight = originHeight;
                    if (originWidth > maxWidth || originHeight > maxHeight) {
                        // 图片尺寸超过1280x1280的限制
                        if (originWidth / originHeight > maxWidth / maxHeight) {
                            targetWidth = maxWidth;
                            targetHeight = Math.round(
                                maxWidth * (originHeight / originWidth)
                            );
                        } else {
                            targetHeight = maxHeight;
                            targetWidth = Math.round(
                                maxHeight * (originWidth / originHeight)
                            );
                        }
                    }
                    console.log("压缩后", targetHeight, targetWidth);
                    const ctx = elem.getContext("2d");
                    // canvas对图片进行缩放
                    elem.width = targetWidth;
                    elem.height = targetHeight;
                    // 清除画布
                    // ctx.clearRect(0, 0, targetWidth, targetHeight);
                    // 图片压缩
                    ctx.drawImage(img, 0, 0, targetWidth, targetHeight);
                    document.body.append(elem);
                    // 兼容旧浏览器提供它没有原生支持的较新的功能,通过toDataURL填充支持toBlob。
                    if (!HTMLCanvasElement.prototype.toBlob) {
                        // 从HTML canvas元素创建Blob对象
                        Object.defineProperty(HTMLCanvasElement.prototype, "toBlob", {
                            value(callback, type, quality) {
                                const dataURL = this.toDataURL(type, quality).split(",")[1];
                                setTimeout(() => {
                                    const binStr = atob(dataURL);
                                    const len = binStr.length;
                                    const arr = new Uint8Array(len);
                                    for (let i = 0; i < len; i++) {
                                        arr[i] = binStr.charCodeAt(i);
                                    }
                                    callback(new Blob([arr], {
                                        type: type || "image/jpeg"
                                    }));
                                });
                            },
                        });
                    }
                    ctx.canvas.toBlob(
                        (blob) => {
                            const cFile = new File([blob], fileName, {
                                type: "image/jpeg",
                                lastModified: Date.now(),
                            });
                            resolve(cFile);
                        },
                        "image/jpeg",
                        quality
                    );
                    document.body.removeChild(elem);
                };
                reader.onerror = (error) => reject(error);
            };
        });
        return compressionImage;
    }
};

export default compressionImg;
```

## 如何使用

## Params

|参数|说明|类型|可选参数|默认值|
|file|压缩资源|object|否|-|
|limit|图片大小超过设置的limit值才会进行压缩|number|否|-|
|maxWidth|最大宽度限制|number|是|—|
|maxHeight|最大高度限制|number|是|-|
|quality|画面质量0~1之间|number|是|0.8|

## 最终效果

[![BPe3Af.png](https://s1.ax1x.com/2020/10/21/BPe3Af.png)](https://imgchr.com/i/BPe3Af)
上图是压缩前后的对比图，从视觉外观上看两张图片几乎完全一样，为了更好的量化我们的压缩效果，这里引入了结构相似性算法（SSIM），可以得出在压缩率为  98.54% 的情况下 SSIM 还能达到 83.3% ，说明可以很好的还原我们的原图。
