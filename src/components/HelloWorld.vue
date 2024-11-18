<template>
  <canvas :width="this.width" :height="this.height" ref="canvas"></canvas>
  <input type="file" @change="changeImg" ref="image" />
  <select ref="effectList" multiple="multiple" size="9" @change="changeSelect">
    <option v-for="item in this.typeList" :key="item.name" :value="item.value">
      {{ item.name }}
    </option>
  </select>
  <div id="matrix">
    <template v-for="(item, index) in this.list" :key="index">
      <input v-model="this.list[index]" @change="changeInput" />
    </template>
  </div>
  <label>divisor(除数):<input id="divisor" v-model="this.divisor" /></label>
  <label>offset(偏移量):<input id="offset" v-model="this.offset" /></label>
  <button @click="renderImage">应用</button>
</template>

<script>
export default {
  name: '图像处理demo',
  data() {
    return {
      img: null,
      width: 0,
      height: 0,
      list: [0, 0, 0, 0, 1, 0, 0, 0, 0],
      typeList: [
        { value: '0,0,0,0,1,0,0,0,0,1,0', name: 'none(原图)' },
        { value: '0,0,0,0,-1,0,0,0,0,1,255', name: 'inverse(反色)' },
        { value: '0,1,0,1,1,1,0,1,0,5,0', name: 'blur(模糊)' },
        { value: '0,-1,0,-1,5,-1,0,-1,0,1,0', name: 'sharpen(锐化)' },
        { value: '0,0,0,-1,1,0,0,0,0,1,127', name: 'hEdge(横向边缘增强)' },
        { value: '0,-1,0,0,1,0,0,0,0,1,127', name: 'vEdge(纵向边缘增强)' },
        { value: '0,-1,0,-1,4,-1,0,-1,0,1,127', name: 'edge(边缘增强)' },
        { value: '0,-1,0,-1,4,-1,0,-1,0,-1,255', name: 'edge(白色边缘增强)' },
        { value: '-6,-3,0,-3,1,3,0,3,6,1,0', name: 'emboss(浮雕)' },
        { value: '0,0,0,0,2,0,0,0,0,1,-255', name: 'contrast(增加对比度)' },
        { value: '0,-2,0,-2,20,-2,0,-2,0,10,-40', name: 'enhance(颜色加深)' },
        { value: '1,0,0,0,0,0,0,0,0,1,0', name: 'shift(位移)' },
        { value: '-1', name: 'grey(心理学灰度)' },
        { value: '-2', name: '绿色(单色灰度)' },
        { value: '-3', name: 'grey(平均色灰度)' },
      ],
      divisor: 1,
      offset: 0,
    }
  },
  methods: {
    changeImg(input) {
      let self = this
      const files = this.$refs.image.files[0]
      let reader = new FileReader()

      reader.readAsDataURL(files)

      reader.onload = function () {
        const canvas = self.$refs.canvas,
          context = canvas.getContext('2d')
        self.img = new Image()
        self.img.src = reader.result
        self.img.onload = function () {
          self.width = self.img.width
          self.height = self.img.height
          setTimeout(() => {
            context.drawImage(self.img, 0, 0, 500, 500)
          }, 200)
        }
      }

      reader.onerror = function () {
        console.log(reader.error)
      }
    },
    changeSelect() {
      const list = this.$refs.effectList.value.split(',')
      this.list = list.slice(0, 9)
      this.divisor = list[9]
      this.offset = list[10]
      this.renderImage()
    },
    // 计算卷积矩阵的函数
    ConvolutionMatrix(input, matrix, divisor, offset) {
      // 创建一个输出的 imageData 对象
      const output = this.$refs.canvas.getContext('2d').createImageData(input)

      const w = input.width,
        h = input.height
      const iD = input.data,
        oD = output.data
      const m = matrix.map((v) => +v)

      // 对除了边缘的点之外的内部点的 RGB 进行操作，透明度在最后都设为 255
      for (let y = 1; y < h - 1; y += 1) {
        for (let x = 1; x < w - 1; x += 1) {
          for (let c = 0; c < 3; c += 1) {
            let i = (y * w + x) * 4 + c
            oD[i] =
              +offset +
              (m[0] * iD[i - w * 4 - 4] +
                m[1] * iD[i - w * 4] +
                m[2] * iD[i - w * 4 + 4] +
                m[3] * iD[i - 4] +
                m[4] * iD[i] +
                m[5] * iD[i + 4] +
                m[6] * iD[i + w * 4 - 4] +
                m[7] * iD[i + w * 4] +
                m[8] * iD[i + w * 4 + 4]) /
                divisor
          }
          oD[(y * w + x) * 4 + 3] = 255 // 设置透明度
        }
      }
      return output
    },
    greyData(type, input) {
      const output = this.$refs.canvas.getContext('2d').createImageData(input)

      const w = input.width,
        h = input.height
      const iD = input.data,
        oD = output.data

      // 对除了边缘的点之外的内部点的 RGB 进行操作，设置灰度
      for (let y = 1; y < h - 1; y += 1) {
        for (let x = 1; x < w - 1; x += 1) {
          let i = (y * w + x) * 4
          if (type == 0) {
            //心理学灰度
            const grey = iD[i] * 0.299 + iD[i + 1] * 0.587 + iD[i + 2] * 0.114
            oD[i] = oD[i + 1] = oD[i + 2] = grey
          } else if (type == 1) {
            //单绿色灰度
            const grey = iD[i + 1]
            oD[i] = oD[i + 1] = oD[i + 2] = grey
          } else if (type == 2) {
            //平均色灰度
            const grey = (iD[i] + iD[i + 1] + iD[i + 2]) / 3
            oD[i] = oD[i + 1] = oD[i + 2] = grey
          }
          oD[i + 3] = 255
        }
      }
      return output
    },
    //渲染函数
    renderImage() {
      // 获取矩阵数据以及除数因子和偏移
      const matrix = this.list,
        divisor = +this.divisor,
        offset = +this.offset

      // 绘制原始图片
      const canvas = this.$refs.canvas
      const context = canvas.getContext('2d')
      context.drawImage(this.img, 0, 0)

      // 获取图片数据，并把处理后的数据重新绘制到 canvas上去
      const input = context.getImageData(0, 0, canvas.width, canvas.height)

      let output
      if (this.list.length === 1 && this.list[0] == -1) {
        output = this.greyData(0, input)
      } else if (this.list.length === 1 && this.list[0] == -2) {
        output = this.greyData(1, input)
      } else if (this.list.length === 1 && this.list[0] == -3) {
        output = this.greyData(2, input)
      } else {
        output = this.ConvolutionMatrix(input, matrix, divisor, offset)
      }
      context.putImageData(output, 0, 0)

      return false
    },
  },
}
</script>
