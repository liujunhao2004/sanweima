<template>
    <div class="qr-generator-container">

        <!-- hugfix    1抠图插件有问题 2.z轴在打印的时候凹凸感低  -->
        <h1><i class="fas fa-qrcode"></i> 不规则图片三维码生成器</h1>
        <h2 style="color:#fff">注：图片在assets/img文件夹中可测试  抠图插件有问题图片有漏光问题</h2>
        <h2 style="color:#fff">注：可以在编码的时候给图片设置一个边框这样就更好看了，太累了写不动了</h2>
        <div class="upload-section">
            <div class="upload-area" @click="triggerFileInput" @dragover.prevent @drop="handleDrop">
                <i class="fas fa-cloud-upload-alt upload-icon"></i>
                <p>点击或拖拽上传不规则形状图片</p>
                <input type="file" ref="fileInput" @change="handleFileUpload" accept="image/*" style="display:none;">
            </div>

            <div class="preview-area">
                <div class="preview-box">
                    <h3>原始图片</h3>
                    <img :src="originalImage || placeholderImage" class="preview-img">
                </div>
                <div class="preview-box">
                    <h3>透明背景</h3>
                    <img :src="processedImage || placeholderImage" class="preview-img transparent-bg">
                </div>
            </div>
        </div>

        <div class="config-section">
            <div class="input-group">
                <label>三维码内容：</label>
                <input type="text" v-model="qrContent" placeholder="输入网址或文本">
            </div>

            <div class="input-group">
                <label>三维码密度：</label>
                <select v-model="qrDensity">
                    <option value="low">低</option>
                    <option value="medium" selected>中</option>
                    <option value="high">高</option>
                </select>
            </div>

            <button class="generate-btn" @click="generateQRCode" :disabled="!processedImage">
                <i class="fas fa-magic"></i> 生成环绕三维码
            </button>
        </div>

        <div class="result-section">
            <div class="qr-preview" :style="{ width: qrSize + 'px', height: qrSize + 'px' }">
                <canvas ref="qrCanvas" :width="qrSize" :height="qrSize" style="    margin-top: -16px;
    margin-left: -17px;"></canvas>
            </div>

            <div class="status-message" :class="statusClass">
                <i :class="statusIcon"></i> {{ statusMessage }}
            </div>

            <button class="download-btn" @click="downloadQRCode" :disabled="!qrGenerated">
                <i class="fas fa-download"></i> 下载三维码
            </button>
        </div>

    </div>
</template>

<script>
import QRCode from 'qrcode'

export default {
    name: 'IrregularShapeQRCode',
    data() {
        return {
            qrContent: 'https://www.baidu.com',
            originalImage: null,
            processedImage: null,
            qrGenerated: false,
            qrSize: 400,
            qrDensity: 'medium',
            statusMessage: '请上传不规则形状图片',
            statusClass: 'info',
            placeholderImage: 'data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="150" height="150" viewBox="0 0 24 24" fill="none" stroke="%23ccc" stroke-width="1"><rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect><circle cx="8.5" cy="8.5" r="1.5"></circle><polyline points="21 15 16 10 5 21"></polyline></svg>'
        }
    },
    computed: {
        statusIcon() {
            return {
                'fas fa-info-circle': this.statusClass === 'info',
                'fas fa-spinner fa-spin': this.statusClass === 'loading',
                'fas fa-check-circle': this.statusClass === 'success',
                'fas fa-times-circle': this.statusClass === 'error'
            }
        },
        qrOptions() {
            return {
                errorCorrectionLevel: 'H',
                margin: 1,
                width: this.qrSize,
                color: {
                    dark: '#000000',
                    light: '#ffffff00' // 透明背景
                }
            }
        }
    },
    methods: {
        triggerFileInput() {
            this.$refs.fileInput.click()
        },
        handleFileUpload(event) {
            const file = event.target.files[0]
            if (file && file.type.match('image.*')) {
                const reader = new FileReader()
                reader.onload = (e) => {
                    this.originalImage = e.target.result
                    this.processImage()
                }
                reader.readAsDataURL(file)
            }
        },
        handleDrop(event) {
            event.preventDefault()
            const file = event.dataTransfer.files[0]
            if (file && file.type.match('image.*')) {
                const reader = new FileReader()
                reader.onload = (e) => {
                    this.originalImage = e.target.result
                    this.processImage()
                }
                reader.readAsDataURL(file)
            }
        },
        processImage() {
            this.statusMessage = '正在移除背景...'
            this.statusClass = 'loading'

            const canvas = document.createElement('canvas')
            const ctx = canvas.getContext('2d')
            const img = new Image()

            img.onload = () => {
                canvas.width = img.width
                canvas.height = img.height
                ctx.drawImage(img, 0, 0)

                const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
                const data = imageData.data

                for (let i = 0; i < data.length; i += 4) {
                    const r = data[i]
                    const g = data[i + 1]
                    const b = data[i + 2]

                    if (r > 220 && g > 220 && b > 220) {
                        data[i + 3] = 0
                    }
                }

                ctx.putImageData(imageData, 0, 0)
                this.processedImage = canvas.toDataURL('image/png')

                this.statusMessage = '背景移除完成！'
                this.statusClass = 'success'
            }

            img.src = this.originalImage
        },
        async generateQRCode() {
            if (!this.qrContent) {
                this.statusMessage = '请输入三维码内容';
                this.statusClass = 'error';
                return;
            }

            this.statusMessage = '正在生成彩色点状三维码...';
            this.statusClass = 'loading';

            try {
                const canvas = this.$refs.qrCanvas;
                const ctx = canvas.getContext('2d');
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                ctx.fillStyle = '#ffffff';
                ctx.fillRect(0, 0, canvas.width, canvas.height);
                const qrData = await QRCode.create(this.qrContent, {
                    errorCorrectionLevel: 'H'
                });

                const moduleCount = qrData.modules.size;
                const moduleSize = canvas.width / moduleCount;
                const dotSize = moduleSize * 0.6;

                for (let y = 0; y < moduleCount; y++) {
                    for (let x = 0; x < moduleCount; x++) {
                        if (!this.isInFinderPattern(x, y, moduleCount)) {
                            if (qrData.modules.data[y * moduleCount + x]) {
                                const hue = (x * y + x + y) % 360;
                                const saturation = 80 + Math.random() * 20;
                                const lightness = 30 + Math.random() * 20;

                                ctx.fillStyle = `hsl(${hue}, ${saturation}%, ${lightness}%)`;

                                const centerX = x * moduleSize + moduleSize / 2;
                                const centerY = y * moduleSize + moduleSize / 2;

                                ctx.beginPath();
                                ctx.arc(centerX, centerY, dotSize / 2, 0, Math.PI * 2);
                                ctx.fill();
                            }
                        }
                    }
                }

                this.drawCustomFinderPatterns(ctx, moduleCount, moduleSize);

                if (this.processedImage) {
                    const shapeImg = new Image();
                    shapeImg.src = this.processedImage;
                    await new Promise((resolve) => { shapeImg.onload = resolve; });

                    const maxHeight = canvas.height * 0.65;
                    const ratio = Math.min(
                        canvas.width / shapeImg.width,
                        maxHeight / shapeImg.height
                    );
                    const width = shapeImg.width * ratio;
                    const height = shapeImg.height * ratio;
                    const x = (canvas.width - width) / 2;
                    const y = canvas.height - height;

                    ctx.drawImage(shapeImg, x, y, width, height);
                    this.addSurroundingCodes(ctx, canvas.width / 2, y, width / 2);
                }

                this.qrGenerated = true;
                this.statusMessage = '三维维码生成成功！';
                this.statusClass = 'success';
            } catch (error) {
                this.statusMessage = '生成失败: ' + error.message;
                this.statusClass = 'error';
                console.error(error);
            }
        },
        isInFinderPattern(x, y, moduleCount) {
            const finderPatternSize = 7;
            const positions = [
                { x: 0, y: 0 }, // 左上
                { x: moduleCount - finderPatternSize, y: 0 }, // 右上
                { x: 0, y: moduleCount - finderPatternSize } // 左下
            ];

            return positions.some(pos =>
                x >= pos.x && x < pos.x + finderPatternSize &&
                y >= pos.y && y < pos.y + finderPatternSize
            );
        },

        drawCustomFinderPatterns(ctx, moduleCount, moduleSize) {
            const finderPatternSize = 7;
            const positions = [
                { x: 0, y: 0 }, // 左上
                { x: moduleCount - finderPatternSize, y: 0 }, // 右上
                { x: 0, y: moduleCount - finderPatternSize } // 左下
            ];

            const outerColor = '#FF5722'; // 橙色外框
            const innerColor = '#FFFFFF'; // 白色内框
            const centerColor = '#FF5722'; // 橙色中心

            positions.forEach(pos => {
                // 外框
                ctx.fillStyle = outerColor;
                ctx.fillRect(
                    pos.x * moduleSize,
                    pos.y * moduleSize,
                    finderPatternSize * moduleSize,
                    finderPatternSize * moduleSize
                );

                // 内框
                ctx.fillStyle = innerColor;
                ctx.fillRect(
                    (pos.x + 1) * moduleSize,
                    (pos.y + 1) * moduleSize,
                    (finderPatternSize - 2) * moduleSize,
                    (finderPatternSize - 2) * moduleSize
                );

                // 中心点
                ctx.fillStyle = centerColor;
                ctx.fillRect(
                    (pos.x + 2) * moduleSize,
                    (pos.y + 2) * moduleSize,
                    (finderPatternSize - 4) * moduleSize,
                    (finderPatternSize - 4) * moduleSize
                );
            });
        },
        // async generateQRCode() {
        //     if (!this.qrContent) {
        //         this.statusMessage = '请输入三维码内容'
        //         this.statusClass = 'error'
        //         return
        //     }

        //     this.statusMessage = '正在生成环绕三维码...'
        //     this.statusClass = 'loading'

        //     try {
        //         const canvas = this.$refs.qrCanvas
        //         const ctx = canvas.getContext('2d')

        //         ctx.clearRect(0, 0, canvas.width, canvas.height)

        //         const qrDataUrl = await QRCode.toDataURL(this.qrContent, this.qrOptions)

        //         const qrImg = new Image()
        //         qrImg.src = qrDataUrl

        //         await new Promise((resolve) => {
        //             qrImg.onload = resolve
        //         })

        //         ctx.drawImage(qrImg, 0, 0, canvas.width, canvas.height)
        //         const centerX = canvas.width / 2
        //         const centerY = canvas.height / 2
        //         const maxRadius = Math.min(canvas.width, canvas.height) * 0.35

        //         const shapeImg = new Image()
        //         shapeImg.src = this.processedImage

        //         await new Promise((resolve) => {
        //             shapeImg.onload = resolve
        //         })

        //         // 计算图片尺寸以保持比例
        //         const ratio = Math.min(
        //             (maxRadius * 2) / shapeImg.width,
        //             (maxRadius * 2) / shapeImg.height
        //         )
        //         const width = shapeImg.width * ratio
        //         const height = shapeImg.height * ratio

        //         ctx.save()
        //         ctx.globalCompositeOperation = 'destination-out'
        //         ctx.drawImage(
        //             shapeImg,
        //             centerX - width / 2,
        //             centerY - height / 2,
        //             width,
        //             height
        //         )
        //         ctx.restore()

        //         ctx.drawImage(
        //             shapeImg,
        //             centerX - width / 2,
        //             centerY - height / 2,
        //             width,
        //             height
        //         )
        //         this.addSurroundingCodes(ctx, centerX, centerY, maxRadius)

        //         this.qrGenerated = true
        //         this.statusMessage = '环绕三维码生成成功！'
        //         this.statusClass = 'success'
        //     } catch (error) {
        //         this.statusMessage = '生成失败: ' + error.message
        //         this.statusClass = 'error'
        //         console.error(error)
        //     }
        //     const shapeImg = new Image()
        //     shapeImg.src = this.processedImage

        //     await new Promise((resolve) => {
        //         shapeImg.onload = resolve
        //     })

        //     // 计算图片尺寸以保持比例，同时确保图片高度不超过三维码高度的一定比例
        //     const maxHeight = canvas.height * 0.65 // 限制图片最大高度为三维码高度的65%
        //     const ratio = Math.min(
        //         canvas.width / shapeImg.width,
        //         maxHeight / shapeImg.height
        //     )
        //     const width = shapeImg.width * ratio
        //     const height = shapeImg.height * ratio

        //     const x = (canvas.width - width) / 2 // 水平居中
        //     const y = canvas.height - height     // 底部对齐

        //             //     ctx.save()
        //     ctx.globalCompositeOperation = 'destination-out'
        //     ctx.drawImage(shapeImg, x, y, width, height)
        //     ctx.restore()

        //     ctx.drawImage(shapeImg, x, y, width, height)
        // },
        addSurroundingCodes(ctx, centerX, topY, radius) {
            const dotSize = 5
            const rows = 3 // 在图片上方添加3行编码点

            ctx.fillStyle = '#000000'

            for (let row = 1; row <= rows; row++) {
                const y = topY - row * 15
                const dotCount = Math.floor(radius / 10) * 2

                for (let i = 0; i < dotCount; i++) {
                    const x = centerX - radius + (i * 20)
                    if (x > centerX - radius && x < centerX + radius) {
                        ctx.beginPath()
                        ctx.arc(x, y, dotSize, 0, Math.PI * 2)
                        ctx.fill()
                    }
                }
            }

            const sideDots = 5
            const sideOffset = 15

            for (let i = 0; i < sideDots; i++) {
                const y = topY + (i * 20)
                ctx.beginPath()
                ctx.arc(centerX - radius - sideOffset, y, dotSize, 0, Math.PI * 2)
                ctx.fill()
                ctx.beginPath()
                ctx.arc(centerX + radius + sideOffset, y, dotSize, 0, Math.PI * 2)
                ctx.fill()
            }
        },
        // addSurroundingCodes(ctx, centerX, centerY, radius) {
        //     const dotSize = 5
        //     const ringCount = 3
        //     const dotsPerRing = [8, 16, 24] // 每圈的点的数量

        //     ctx.fillStyle = '#000000'

        //     for (let ring = 1; ring <= ringCount; ring++) {
        //         const currentRadius = radius + (ring * 10)
        //         const dotCount = dotsPerRing[ring - 1] || 16

        //         for (let i = 0; i < dotCount; i++) {
        //             const angle = (i / dotCount) * Math.PI * 2
        //             const x = centerX + Math.cos(angle) * currentRadius
        //             const y = centerY + Math.sin(angle) * currentRadius


        //             if (Math.abs(Math.cos(angle)) > 0.7 || Math.abs(Math.sin(angle)) > 0.7) {
        //                 ctx.beginPath()
        //                 ctx.arc(x, y, dotSize, 0, Math.PI * 2)
        //                 ctx.fill()
        //             }
        //         }
        //     }
        // },
        downloadQRCode() {
            if (!this.qrGenerated) return

            const link = document.createElement('a')
            link.download = 'surrounding-qrcode.png'
            link.href = this.$refs.qrCanvas.toDataURL('image/png')
            link.click()
        }
    }
}
</script>

<style scoped>
.qr-generator-container {
    max-width: 900px;
    margin: 0 auto;
    padding: 20px;
    font-family: 'Arial', sans-serif;
}

h1 {
    text-align: center;
    color: #2c3e50;
    margin-bottom: 30px;
}

.upload-section {
    background: #f8f9fa;
    padding: 20px;
    border-radius: 10px;
    margin-bottom: 20px;
}

.upload-area {
    border: 2px dashed #3498db;
    border-radius: 8px;
    padding: 40px;
    text-align: center;
    cursor: pointer;
    transition: all 0.3s;
    margin-bottom: 20px;
}

.upload-area:hover {
    background: rgba(52, 152, 219, 0.1);
}

.upload-icon {
    font-size: 48px;
    color: #3498db;
    margin-bottom: 10px;
}

.preview-area {
    display: flex;
    justify-content: space-around;
    margin-top: 20px;
    flex-wrap: wrap;
}

.preview-box {
    text-align: center;
    margin: 10px;
    flex: 1;
    min-width: 150px;
}

.preview-img {
    width: 150px;
    height: 150px;
    object-fit: contain;
    border: 1px solid #ddd;
    background: #fff;
}

.transparent-bg {
    background:
        linear-gradient(45deg, #eee 25%, transparent 25%, transparent 75%, #eee 75%),
        linear-gradient(45deg, #eee 25%, transparent 25%, transparent 75%, #eee 75%);
    background-size: 20px 20px;
    background-position: 0 0, 10px 10px;
}

.config-section {
    background: #f8f9fa;
    padding: 20px;
    border-radius: 10px;
    margin-bottom: 20px;
}

.input-group {
    margin-bottom: 15px;
}

.input-group label {
    display: block;
    margin-bottom: 5px;
    font-weight: bold;
}

.input-group input[type="text"],
.input-group select {
    width: 100%;
    padding: 10px;
    border: 1px solid #ddd;
    border-radius: 4px;
}

.generate-btn {
    width: 100%;
    padding: 12px;
    background: #3498db;
    color: white;
    border: none;
    border-radius: 4px;
    font-size: 16px;
    cursor: pointer;
    transition: background 0.3s;
}

.generate-btn:hover {
    background: #2980b9;
}

.generate-btn:disabled {
    background: #95a5a6;
    cursor: not-allowed;
}

.result-section {
    text-align: center;
    margin-top: 20px;
}

.qr-preview {
    margin: 0 auto 20px;
    border: 15px solid white;
    border-radius: 10px;
    box-shadow: 0 5px 15px rgba(0, 0, 0, 0.1);
    background: white;
}

.status-message {
    padding: 10px;
    border-radius: 4px;
    margin-bottom: 15px;
}

.status-message.info {
    background: #d6eaf8;
    color: #3498db;
}

.status-message.loading {
    background: #fcf3cf;
    color: #f39c12;
}

.status-message.success {
    background: #d5f5e3;
    color: #27ae60;
}

.status-message.error {
    background: #fadbd8;
    color: #e74c3c;
}

.download-btn {
    padding: 12px 25px;
    background: #2ecc71;
    color: white;
    border: none;
    border-radius: 4px;
    font-size: 16px;
    cursor: pointer;
    transition: background 0.3s;
}

.download-btn:hover {
    background: #27ae60;
}

.download-btn:disabled {
    background: #95a5a6;
    cursor: not-allowed;
}
</style>