<script setup>
import { binarize, Decoder, Detector, grayscale } from '@nuintun/qrcode';
import {ref} from "vue";

const parsedResult = ref("");

// 监听粘贴事件
document.addEventListener('paste', function(event) {
  // 获取剪贴板数据
  const clipboardItems = event.clipboardData.items;

  // 遍历剪贴板中的项目
  for (let i = 0; i < clipboardItems.length; i++) {
    const item = clipboardItems[i];

    // 检查是否是图片类型
    if (item.type.indexOf('image') !== -1) {
      // 获取图片的Blob对象
      const blob = item.getAsFile();

      // 创建FileReader来读取Blob
      const reader = new FileReader();

      reader.onload = function(e) {
        // 创建Image对象
        const image = new Image();

        // 设置图片源为读取的数据
        image.crossOrigin = 'anonymous';

        image.addEventListener('error', () => {
          console.error('image load error');
        });

        image.addEventListener('load', () => {
          const { width, height } = image;
          const canvas = new OffscreenCanvas(width, height);
          const context = canvas.getContext('2d');

          context.drawImage(image, 0, 0);

          const luminances = grayscale(context.getImageData(0, 0, width, height));
          const binarized = binarize(luminances, width, height);
          const detector = new Detector();
          // Notice: the detect result are possible combinations of QR Code regions,
          // which may not necessarily be successfully decoded.
          const detected = detector.detect(binarized);
          const decoder = new Decoder();

          let current = detected.next();

          console.log(current, current.done)

          while (!current.done) {
            let succeed = false;

            const detect = current.value;

            try {
              const { size, finder, alignment } = detect;
              const decoded = decoder.decode(detect.matrix);
              // Finder
              const { topLeft, topRight, bottomLeft } = finder;
              // Corners
              const topLeftCorner = detect.mapping(0, 0);
              const topRightCorner = detect.mapping(size, 0);
              const bottomRightCorner = detect.mapping(size, size);
              const bottomLeftCorner = detect.mapping(0, size);
              // Timing
              const topLeftTiming = detect.mapping(6.5, 6.5);
              const topRightTiming = detect.mapping(size - 6.5, 6.5);
              const bottomLeftTiming = detect.mapping(6.5, size - 6.5);

              console.log({
                content: decoded.content,
                finder: [topLeft, topRight, bottomLeft],
                alignment: alignment ? alignment : null,
                timing: [topLeftTiming, topRightTiming, bottomLeftTiming],
                corners: [topLeftCorner, topRightCorner, bottomRightCorner, bottomLeftCorner]
              });

              parsedResult.value = decoded.content
              console.log(parsedResult.value)

              succeed = true;
            } catch (e) {
              // Decode failed, skipping...
              parsedResult.value = "失败：" + e
            }

            // Notice: pass succeed to next() is very important,
            // this can significantly reduce the number of detections.

            // do not parse next one
            // current = detected.next(succeed);
            break;
          }
        });

        image.src = e.target.result;
        console.log(e.target.result);
      };

      // 读取Blob数据为DataURL
      reader.readAsDataURL(blob);

      // 找到一个图片后就停止遍历
      break;
    }
  }
});


</script>

<template>
  <div class="body">
    <p style="margin-bottom: 20px;">Ctrl-V粘贴二维码图片以运行解析</p>
    <p>解析结果：{{parsedResult}}</p>
  </div>
  <div style="position: absolute; bottom: 200px; left: 0; width: 100vw; text-align: center;">
    <a href="https://github.com/nuintun/qrcode?tab=readme-ov-file">nuintun/qrcode</a>
  </div>
</template>

<style scoped>
.body {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  width: 100vw;
  height: 100vh;
}
</style>
