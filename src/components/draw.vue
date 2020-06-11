<template>
    <div class="layout">
        <div class="content">
            <div class="content-left">
                <div class="color-picker">
                    <vue-color-picker v-model="color" v-if="isChooseColor" @ok="setColor()"
                                      @cancel="falseColor()"></vue-color-picker>
                </div>
                <div class="pen-size" v-show="showLineSize">
                    <div class="line-size">{{penSize}}</div>
                    <div class="block">
                        <mu-slider class="demo-slider" v-model="penSize" @change="setLineSize" :max="20"></mu-slider>
                    </div>
                </div>
                <div class="print-tools">
                    <a href="javascript: void (0);" v-for="(tool, index) in tools" :key="index" @click="drawType(tool)">
                        <mu-menu :class="{'selected':tool.isChoose}">
                            <mu-list-item :index="'drawing_type_'+index">
                                <template slot="title">
                                    <v-icon :name="tool.icon"></v-icon>
                                    <span>{{tool.name}}</span>
                                </template>
                            </mu-list-item>
                        </mu-menu>
                    </a>
                </div>
            </div>
            <div class="content-right">
                <div class="body" :style="{width:canvasSize.width + 'px',height: canvasSize.height + 'px' }">
                    <canvas id="canvas" ref="canvas" :style="{cursor:cursorStyle}"></canvas>
                    <canvas id="canvas_bak" ref="canvas_bak" :style="{cursor:cursorStyle}"></canvas>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
  import {Photoshop} from 'vue-color'

  export default {
    name: 'drawing-brand',
    props: {
      'opened': Boolean
    },
    components: {
      'VueColorPicker': Photoshop,
    },
    data () {
      return {
        activeDrawType: null,
        textContent: '',
        textFontSize: '24',
        canvasSize: {
          width: window.screen.width - 100,
          height: window.screen.height
        },
        showLineSize: false,
        canvas: this.$refs.canvas,
        canvasTop: 0,
        canvasLeft: 100,
        context: null,
        canvas_bak: this.$refs.canvas_bak,
        context_bak: null,
        isChooseColor: false,
        toolsToggle: false,
        color: {
          hex: '#11507D',
          hsl: {
            h: 205,
            s: 86,
            l: 0,
            a: 1
          },
          hsv: {
            h: 205,
            s: 86,
            v: 92,
            a: 1
          },
          rgba: {
            r: 33,
            g: 150,
            b: 243,
            a: 1
          },
          a: 1
        },
        penSize: 4,
        lineType: [0, 0],
        canDraw: false,
        cursorStyle: 'auto',
        cancelList: [],
        createTrack: null,
        lineTracks: [],
        mucUsers: [],
        cancelIndex: 0,
        tools: [{
          name: '清空',
          icon: 'regular/file',
          fun: 'clear',
          isChoose: false
        }, {
          name: '线宽',
          icon: 'ruler-vertical',
          fun: 'line-size',
          isChoose: false
        }, {
          name: '颜色',
          icon: 'palette',
          fun: 'palette',
          isChoose: false
        }, {
          name: '画笔',
          icon: 'paint-brush',
          fun: 'pencil',
          isChoose: false
        }, {
          name: '直线',
          icon: 'slash',
          fun: 'line',
          isChoose: false
        }, {
          name: '圆形',
          icon: 'regular/circle',
          fun: 'circle',
          isChoose: false
        }, {
          name: '矩形',
          icon: 'regular/object-group',
          fun: 'square',
          isChoose: false
        }, {
          name: '刷子',
          icon: 'paint-roller',
          fun: 'handwriting',
          isChoose: false
        }, {
          name: '橡皮',
          icon: 'eraser',
          fun: 'rubber',
          isChoose: false
        }, {
          name: '文本',
          icon: 'font',
          fun: 'text',
          isChoose: false
        }, {
          name: '保存',
          icon: 'regular/save',
          fun: 'save',
          isChoose: false
        }, {
          name: '关闭',
          icon: 'times',
          fun: 'close',
          isChoose: false
        }]
      }
    },
    methods: {
      sendActionMsg (_action, _data, _type, receiver = '', delay = false) {
        let _this = this
        if (delay) {
          return true
        }
        switch (_type) {
          case 'create':
            // clean lineTracks and store create msg
            // _this.lineTracks = []
            _this.createTrack = {action: _action, data: _data, type: _type}
            break
          case 'clear':
            _this.lineTracks = []
            break
          case 'destroy':
            _this.lineTracks = []
            _this.createTrack = null
            break
          case 'pencil':
          case 'rubber':
            _this.lineTracks.push({action: _action, data: _data, type: _type})
            break
          case 'text':
            this.curcursor = 'text'
        }
        // console.log('-------+++++++lineTracks++++++--------', _this.lineTracks)
      },
      initCanvas () {
        let _this = this
        this.canvas = document.getElementById('canvas')
        this.canvas.width = this.canvasSize.width
        this.canvas.height = this.canvasSize.height
        this.context = this.canvas.getContext('2d')
        this.canvas_bak = document.getElementById('canvas_bak')
        this.canvas_bak.width = this.canvasSize.width
        this.canvas_bak.height = this.canvasSize.height
        this.context_bak = this.canvas_bak.getContext('2d')
        let size = this.canvas.width + 'x' + this.canvas.height
        _this.sendActionMsg('create', size, 'create')
      },
      setLineSize (val) {
        this.penSize = val
        this.falseLineSize()
      },
      setColor () {
        this.showLineSize = false
        this.isChooseColor = !this.isChooseColor
        if (this.activeDrawType) {
          this.drawType(this.activeDrawType)
        }
      },
      falseColor () {
        this.isChooseColor = false
      },
      falseLineSize () {
        this.showLineSize = false
        this.drawType(this.activeDrawType)
      },
      doCloseDrawingBoard () {
        let _this = this
        // clear board and close it
        _this.drawType(_this.tools[0])
        _this.cancelList = []
        _this.lineTracks = []
        _this.sendActionMsg('destroy', '', 'destroy')
        _this.$emit('switchDrawingBoard', false)
      },
      closeDrawingBoard () {
        let _this = this
        console.log('will close board')
        _this.$toast.question('是否关闭画图板?', '提示', {
          timeout: 20000,
          close: false,
          overlay: true,
          toastOnce: true,
          id: 'question',
          zindex: 999,
          position: 'center',
          buttons: [
            ['<button><b>确定</b></button>', function (instance, toast) {
              instance.hide({ transitionOut: 'fadeOut' }, toast, 'button')
              console.log('handle yes click')
              _this.doCloseDrawingBoard()
            }, true],
            ['<button>取消</button>', function (instance, toast) {
              instance.hide({ transitionOut: 'fadeOut' }, toast, 'button')
              console.log('handle no click')
            }]
          ]
        })
      },
      drawType (pen) {
        console.log('chose pen [', pen.fun, '] at: ', new Date())
        switch (pen.fun) {
          case 'close':
            this.closeDrawingBoard()
            break
          case 'palette':
            this.setColor()
            break
          case 'line-size':
            this.isChooseColor = false
            this.showLineSize = !this.showLineSize
            break
          case 'clear':
          case 'save':
            this.boardAction(pen.fun)
            break
          case 'pencil':
            this.activeDrawType = pen
            this.cursorStyle = "url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABgAAAAYCAQAAABKfvVzAAAAZ0lEQVR4AdXOrQ2AMBRF4bMc/zOUOSrYoYI5cQQwpAieQDW3qQBO7Xebxx8bWAk5/CASmRHzRHtB+d0Bkw0W5ZiT0SYbFcl6u/2eeJHbxIHOhWO6Er6/y9syXpMul5PLefAGKZ1/rwtTimwbWLpiCgAAAABJRU5ErkJggg==') 3 24,  auto"
            break
          case 'rubber':
            this.activeDrawType = pen
            this.cursorStyle = 'pointer'
            break
          case 'circle':
          case 'square':
          case 'line':
          case 'handwriting':
            this.activeDrawType = pen
            this.cursorStyle = 'crosshair'
            break
          default:
            this.cursorStyle = 'auto'
            break
        }
        this.draw_graph(pen.fun)
        this.chooseImg(pen)
      },
      // 选择功能按钮 修改样式
      chooseImg (obj) {
        for (let i = 0; i < this.tools.length; i++) {
          this.tools[i].isChoose = false
        }
        obj.isChoose = true
        console.log('set chose img at: ', new Date())
      },
      boardAction (fun) {
        if (fun === 'clear') {
          this.clearContext('1')
        } else if (fun === 'save') {
          this.downloadImage()
        }
      },
      draw_graph (graphType) {
        this.canvas_bak.style.zIndex = 1
        // 先画在蒙版上 再复制到画布上
        this.canDraw = false
        let startX, startY
        // 鼠标按下获取 开始xy开始画图
        let mousedown = (e) => {
          this.context.strokeStyle = this.color.hex
          this.context_bak.strokeStyle = this.color.hex
          this.context_bak.lineWidth = this.penSize
          e = e || window.event
          // console.log('drawing start: ', e, this.penSize, this.color.hex, graphType)
          startX = e.clientX - this.canvasLeft
          startY = e.clientY - this.canvasTop
          this.context_bak.moveTo(startX, startY)
          this.lastXCoordinate = startX
          this.lastYCoordinate = startY
          this.canDraw = true
          if (graphType === 'text') {
            let _this = this
            _this.drawText(this.context_bak, startX, startY, function () {
              _this.saveDta2Canvas(e)
            })
          } else if (graphType === 'pencil') {
            this.context_bak.beginPath()
          } else if (graphType === 'circle') {
            this.context.beginPath()
            this.context.moveTo(startX, startY)
            this.context.lineTo(startX + 2, startY + 2)
            this.context.stroke()
          } else if (graphType === 'rubber') {
            this.context.clearRect(startX - this.penSize * 10, startY - this.penSize * 10, this.penSize * 20, this.penSize * 20)
          }
        }
        // 鼠标离开 把蒙版canvas的图片生成到canvas中
        let mouseup = (e) => {
          e = e || window.event
          // console.log('drawing stop', e, graphType)
          this.canDraw = false
          // let image = new Image()
          if (graphType !== 'rubber') {
            this.saveDta2Canvas(e)
          }
        }
        // 鼠标移动
        let mousemove = (e) => {
          let _this = this
          e = e || window.event
          let x = e.clientX - this.canvasLeft
          let y = e.clientY - this.canvasTop
          // this.lastXCoordinate = x
          // this.lastYCoordinate = y
          this.context_bak.setLineDash(this.lineType)
          // console.log('drawing: ', e, this.penSize, this.color.hex, graphType)
          // 方块  即4条直线
          if (graphType === 'square') {
            if (this.canDraw) {
              this.context_bak.beginPath()
              this.clearContext()
              this.context_bak.moveTo(startX, startY)
              this.context_bak.lineTo(x, startY)
              this.context_bak.lineTo(x, y)
              this.context_bak.lineTo(startX, y)
              this.context_bak.lineTo(startX, startY)
              this.context_bak.stroke()
            }
            // 直线
          } else if (graphType === 'line') {
            if (this.canDraw) {
              this.context_bak.beginPath()
              this.clearContext()
              this.context_bak.moveTo(startX, startY)
              this.context_bak.lineTo(x, y)
              this.context_bak.stroke()
            }
            // 画笔
          } else if (graphType === 'pencil') {
            if (this.canDraw) {
              // record drawing path
              this.xCoordinate = e.clientX - this.canvasLeft
              this.yCoordinate = e.clientY - this.canvasTop
              this.context_bak.lineTo(this.xCoordinate, this.yCoordinate)
              this.context_bak.stroke()
              let penData = {'message': {'mouse': [
                    [(_this.lastXCoordinate || _this.xCoordinate), (_this.lastYCoordinate || _this.yCoordinate)],
                    [(_this.xCoordinate), (_this.yCoordinate)]
                  ],
                  'penColor': _this.color.hex,
                  'penSize': _this.penSize,
                  'penType': 'source-over'},
                'type': 'Annotate'
              }
              _this.sendActionMsg('message', JSON.stringify(penData), graphType)
              this.lastXCoordinate = _this.xCoordinate
              this.lastYCoordinate = _this.yCoordinate
            }
            // 圆 未画得时候 出现一个小圆
          } else if (graphType === 'circle') {
            this.clearContext()
            if (this.canDraw) {
              this.context_bak.beginPath()
              let radii = Math.sqrt((startX - x) * (startX - x) + (startY - y) * (startY - y))
              this.context_bak.arc(startX, startY, radii, 0, Math.PI * 2, false)
              this.context_bak.stroke()
            } else {
              this.context_bak.beginPath()
              this.context_bak.arc(x, y, 20, 0, Math.PI * 2, false)
              this.context_bak.stroke()
            }
            // 涂鸦 未画得时候 出现一个小圆
          } else if (graphType === 'handwriting') {
            if (this.canDraw) {
              this.context_bak.beginPath()
              this.context_bak.strokeStyle = this.color.hex
              this.context_bak.fillStyle = this.color.hex
              this.context_bak.arc(x, y, this.penSize * 10, 0, Math.PI * 2, false)
              this.context_bak.fill()
              this.context_bak.stroke()
            } else {
              this.clearContext()
              this.context_bak.beginPath()
              this.context_bak.fillStyle = this.color.hex
              this.context_bak.arc(x, y, this.penSize * 10, 0, Math.PI * 2, false)
              this.context_bak.fill()
              this.context_bak.stroke()
            }
            // 橡皮擦 不管有没有在画都出现小方块 按下鼠标 开始清空区域
          } else if (graphType === 'rubber') {
            this.context_bak.setLineDash([0, 0])
            this.context_bak.lineWidth = 1
            this.clearContext()
            this.context_bak.beginPath()
            this.context_bak.strokeStyle = '#dddddd'
            this.context_bak.moveTo(x - this.penSize * 10, y - this.penSize * 10)
            this.context_bak.lineTo(x + this.penSize * 10, y - this.penSize * 10)
            this.context_bak.lineTo(x + this.penSize * 10, y + this.penSize * 10)
            this.context_bak.lineTo(x - this.penSize * 10, y + this.penSize * 10)
            this.context_bak.lineTo(x - this.penSize * 10, y - this.penSize * 10)
            this.context_bak.stroke()
            if (this.canDraw) {
              this.context.clearRect(x - this.penSize * 10, y - this.penSize * 10, this.penSize * 20, this.penSize * 20)
              let startX = x - this.penSize * 10
              let startY = y - this.penSize * 10
              let clearData = {
                'message': {
                  'mouse': [[startX, startY], [startX, startY]],
                  'penColor': 'rgba(255,255,255,1.0)',
                  'penSize': _this.penSize * 20,
                  'penType': 'destination-out'},
                'type': 'Annotate'
              }
              _this.sendActionMsg('message', JSON.stringify(clearData), graphType)
            }
            this.context_bak.setLineDash(this.lineType)
          }
        }
        // 鼠标离开区域以外 除了涂鸦 都清空
        let mouseout = () => {
          if (graphType !== 'handwriting') {
            this.clearContext()
          }
        }
        this.canvas_bak.onmousedown = () => mousedown()
        this.canvas_bak.onmousemove = () => mousemove()
        this.canvas_bak.onmouseup = () => mouseup()
        this.canvas_bak.onmouseout = () => mouseout()
      },

      // 将蒙版上的数据持久化到画布
      saveDta2Canvas (e) {
        let x = e.clientX - this.canvasLeft
        let y = e.clientY - this.canvasTop
        this.canDraw = false
        let image = new Image()
        image.src = this.canvas_bak.toDataURL()
        image.onload = () => {
          this.context.drawImage(image, 0, 0, image.width, image.height, 0, 0, this.canvasSize.width, this.canvasSize.height)
          this.clearContext()
          this.saveImageToAry()
        }
        this.context.beginPath()
        this.context.moveTo(x, y)
        this.context.lineTo(x + 2, y + 2)
        this.context.stroke()
      },

      drawText (ctx, x, y, cb) {
        let _this = this
        _this.showTextInput(function (text) {
          console.log('handle input text: ', text)
          _this.textContent = ''
          if (text) {
            ctx.font = _this.textFontSize + 'px Roboto,Lato,sans-serif'
            ctx.fillStyle = _this.color.hex
            ctx.fillText(text, x, y + 6)
          } else {
            ctx.stroke()
            this.clearContext()
            return 0
          }
          if (cb) {
            cb()
          }
        })
      },

      showTextInput (cb) {
        let _this = this
        let selectElement = '<select>'
        for (let i = 2; i < 32; i++) {
          let value = 2 * i
          let selected = (value.toString() === _this.textFontSize) ? 'selected="selected"' : ''
          let optionItem = '<option value="' + value + '" ' + selected + '>' + value + '</option>'
          selectElement += optionItem
        }
        selectElement += '</select>'

        _this.$toast.info('字号:', '请输入文本文案', {
          timeout: 0,
          overlay: true,
          displayMode: 'once',
          id: 'draw_text_inputs',
          zindex: 999,
          position: 'center',
          drag: false,
          inputs: [
            [selectElement, 'change', function (instance, toast, select, e) {
              _this.textFontSize = select.value
            }],
            ['<input type="text">', 'keyup', function (instance, toast, input, e) {
              _this.textContent = input.value
            }, true]
          ],
          buttons: [
            ['<button><b>确定</b></button>', function (instance, toast) {
              instance.hide({ transitionOut: 'fadeOut' }, toast, 'button')
              if (cb && _this.textContent) {
                cb(_this.textContent)
              }
            }, false]
          ]
        })
      },

      clearContext (type) {
        let _this = this
        if (!type) {
          this.context_bak.clearRect(0, 0, this.canvasSize.width, this.canvasSize.height)
        } else {
          this.context.clearRect(0, 0, this.canvasSize.width, this.canvasSize.height)
          this.context_bak.clearRect(0, 0, this.canvasSize.width, this.canvasSize.height)
          let data = {
            'type': 'ClearAnnotate'
          }
          _this.sendActionMsg('message', JSON.stringify(data), 'clear')
        }
      },
      downloadImage () {
        this.$refs.download.href = this.canvas.toDataURL()
        this.$refs.download.click()
      },
      cancel () {
        this.cancelIndex++
        this.context.clearRect(0, 0, this.canvasSize.width, this.canvasSize.height)
        let image = new Image()
        let index = this.cancelList.length - 1 - this.cancelIndex
        let url = this.cancelList[index]
        image.src = url
        image.onload = () => {
          this.context.drawImage(image, 0, 0, image.width, image.height, 0, 0, this.canvasSize.width, this.canvasSize.height)
        }
      },
      next () {
        this.cancelIndex--
        this.context.clearRect(0, 0, this.canvasSize.width, this.canvasSize.height)
        let image = new Image()
        let index = this.cancelList.length - 1 - this.cancelIndex
        let url = this.cancelList[index]
        image.src = url
        image.onload = () => {
          this.context.drawImage(image, 0, 0, image.width, image.height, 0, 0, this.canvasSize.width, this.canvasSize.height)
        }
      },
      // 保存历史 用于撤销
      saveImageToAry () {
        this.cancelIndex = 0
        let dataUrl = this.canvas.toDataURL()
        this.cancelList.push(dataUrl)
      },
      // 处理文件拖入事件，防止浏览器默认事件带来的重定向
      handleDragOver (evt) {
        evt.stopPropagation()
        evt.preventDefault()
      },
      isImage (type) {
        switch (type) {
          case 'image/jpeg':
          case 'image/png':
          case 'image/gif':
          case 'image/bmp':
          case 'image/jpg':
            return true
          default:
            return false
        }
      },
      // 处理拖放文件列表
      handleFileSelect (evt) {
        evt.stopPropagation()
        evt.preventDefault()
        let files = evt.dataTransfer.files
        for (let i = 0, f; (f = files[i]); i++) {
          let t = f.type ? f.type : 'n/a'
          let reader = new FileReader()
          let isImg = this.isImage(t)
          // 处理得到的图片
          if (isImg) {
            reader.onload = (() => {
              return (e) => {
                let image = new Image()
                image.src = e.target.result
                image.onload = () => {
                  this.context.drawImage(image, 0, 0, image.width, image.height, 0, 0, this.canvasSize.width, this.canvasSize.height)
                }
              }
            })(f)
            reader.readAsDataURL(f)
          }
        }
      },
      // 初始化拖入效果
      initDrag () {
        let dragDiv = document.getElementById('canvas_bak')
        dragDiv.addEventListener('dragover', this.handleDragOver, false)
        dragDiv.addEventListener('drop', this.handleFileSelect, false)
      },
      addkeyBoardlistener () {
        document.onkeydown = (event) => {
          let e = event || window.event || event.arguments.callee.caller.arguments[0]
          if (e.keyCode === 89 && e.ctrlKey) {
            // ctrl+Y
            this.next()
          }
          if (e.keyCode === 90 && e.ctrlKey) {
            // ctrl+Z
            this.cancel()
          }
        }
      }
    },
    mounted () {
      console.log('handle drawing board component created')
      this.cancelList = []
      this.lineTracks = []
      this.mucUsers = []
      this.initCanvas()
      this.initDrag() // ignore drag image into board
      this.addkeyBoardlistener() // ignore keyboard events
      this.drawType(this.tools[3])
      this.canvas_bak.addEventListener('click', this.falseColor)
      this.canvas_bak.addEventListener('click', this.falseLineSize)
      window.addEventListener('resize', () => {
        this.canvasSize = {
          width: window.screen.width - 100,
          height: window.screen.height
        }
      })
    }
  }
</script>
<style scoped>
    * {
        -webkit-user-select: none;
        -moz-user-select: none;
        -ms-user-select: none;
        user-select: none;
    }

    canvas {
        position: absolute;
        z-index: 0;
        left: 0;
        right: 0;
        width: 100%;
        height: 100%;
    }

    .layout {
        background-color: rgb(236, 236, 236);
        height: 100%;
    }

    .content {
        overflow: hidden;
        width: 100vw;
        height: 100vh;
        display: flex;
    }

    .content-left {
        width: 100px;
        background-color: #ffffff;
        border-right: 1px solid #999999;
    }

    .content-right {
        width: calc(100% - 100px);
        height: 100%;
        background-color: rgba(0, 0, 0, 0)
    }

    .body {
        position: relative;
        background-color: white;
        border-radius: 5px;
        height: 100%;
        margin: 0 auto;
    }

    .setterSize {
        background: #ffffff;
        text-align: center;
    }

    .el-menu {
        border: 0px;
    }

    .content-left a {
        text-decoration: none;
        color: #606266;
    }

    ul.el-menu.selected {
        background: #5ED055;
    }

    div.pen-size {
        position: absolute;
        left: 100px;
        top: 56px;
        z-index: 2;
        width: 200px;
    }

    div.pen-size > div.block {
        background: #DCDCDC;
        border: 1px solid #D0D0D0;
        border-top: 0px;
        border-bottom-left-radius: 4px;
        border-bottom-right-radius: 4px;
        /*padding: 20px;*/
    }

    .line-size {
        color: #606266;
        border: 1px solid #D0D0D0;
        border-bottom: 0px;
        border-top-left-radius: 4px;
        border-top-right-radius: 4px;
        background: #DCDCDC;
        padding: 10px 0px;
        font-size: 1.2rem;
        text-align: center;
    }

    .color-picker {
        position: absolute;
        z-index: 2;
        top: 56px;
        left: 100px
    }

    .mu-menu.selected {
        background: lightblue;
    }
</style>
