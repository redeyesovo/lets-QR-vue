<template>
  <div id="app">
    <el-container>
      <el-aside style="padding-top: 50px; padding-left: 100px; width:650px; height:700px;">
        <el-upload
          action
          v-if="!imageUrl"
          id="muban"
          class="upload-demo"
          drag
          multiple
          :auto-upload="false"
          :show-file-list="false"
          :on-change="onchange"
        >
          <i class="el-icon-upload" style="margin-top: 250px;"></i>
          <div class="el-upload__text">
            将文件拖到此处，或
            <em>点击上传</em>
          </div>
        </el-upload>
          <canvas v-if="imageUrl" id="poster"></canvas>
          <div
            v-if="dragCom"
            style="position: relative;"
            :style="{
              height: dragcfg.height + 'px',
              width: dragcfg.width + 'px',
              'margin-top': dragcfg.top + 'px'
            }">
            <vue-draggable-resizable
              :lock-aspect-ratio="lockAspect"
              v-if="imageUrl"
              :w="100"
              :h="100"
              :x="x"
              :y="y"
              @dragging="onDrag"
              @resizing="onResize"
              :parent="true"
              style="margin-top:50px,margin-left:100px"
              :draggable="!loading"
              :resizable="!loading"
            >
              <img ref="imgRef" height="98" width="98" src="./QR-Code.png" alt="no Picutres" />
            </vue-draggable-resizable>
          </div>
        <span v-if="imageUrl" class @click="handleRemove(imageUrl)">
          <i class="el-icon-delete"></i>
        </span>
      </el-aside>

      <el-main>
        <div id="fileuploadrect">
          <file-upload
            class="btn-btn-primary"
            :multiple="true"
            :drop="true"
            :drop-directory="true"
            :value="qrlist"
            ref="upload"
            @input-filter="inputFilter"
          >
            <div v-if="thumbs.length==0">
              <i class="fa-fa-plus"></i>
              拖入目录或<em style="color:#409EFF">点击上传</em>
            </div>

            <div v-if="thumbs.length!=0" class="thumbParent" >
              <div class="thumbChild" v-for="thumb in thumbs">
                <img style="height:50px;width:90px" :src="thumb">
              </div>
            </div>
            <div class="pointpoint" v-if="thumbs.length >= 8">
              ……
            </div>
          </file-upload>
          <span class @click="clearQRcode(qrlist)">
            <i style="margin-left: 20px" class="el-icon-delete"></i>
          </span>
        </div>

        <!-- <el-progress
          v-if="!isNaN(parseFloat((loadprogress.finished / loadprogress.total * 100).toFixed(1)))"
          style="margin-left:120px"
          :status="loaddone ? 'success' : '' "
          :percentage="parseInt((loadprogress.finished / loadprogress.total * 100).toFixed(1))"
        ></el-progress> -->
        <el-steps style="margin-left:90px, margin-top:30px"
        v-if="qrlist.length != 0"
        :align-center="true"
        :space="200"
        :active="active"
        finish-status="success">
          <el-step title="读取二维码"></el-step>
          <el-step title="生成二维码"></el-step>
          <el-step title="下载"></el-step>
        </el-steps>

        <el-container style="width:600px">
          <el-row :gutter="20" style="width:100%;">
            <el-col :span="20" style="width:50%">
              <el-button :loading="loading" type="primary" :disabled="!available" style="margin: 10px 0px 20px 270px" @click="paint">生成</el-button>
            </el-col>
            <!-- <el-col :span="20" style="width:50%">
              <el-button type="primary" :disabled="!candownload" style="margin: 10px 0px 20px 70px" @click="downloadZip">下载</el-button>
            </el-col> -->
          </el-row>
        </el-container>

        <el-table ref="table" :data="outlog" height="320px" border style="width: 600px;margin-top: 0px">
          <el-table-column prop="name" label="图片名" width="150"></el-table-column>
          <el-table-column prop="text" label="状态" width="400"></el-table-column>
        </el-table>
      </el-main>
    </el-container>

    <template></template>
  </div>
</template>
  </div>
</template>

<script>
import { fabric } from "fabric";
import VueDraggableResizable from "vue-draggable-resizable";
import { axios } from "axios";
import "vue-upload-component/dist/vue-upload-component.part.css";
import FileUpload from "vue-upload-component";
import QRCode from "qrcode";
import pLimit from "p-limit";
import JSZip from "jszip";
import { saveAs } from "file-saver";

export default {
  name: "App",
  components: {
    QRCode: QRCode,
    FileUpload
  },
  data() {
    return {
      log: [],
      outlog: [],
      thumbs: [],
      imageUrl: "",
      dialogImageUrl: "",
      dialogVisible: false,
      disabled: false,
      width: 100,
      height: 100,
      x: 0,
      y: 0,
      dialogheight: 0,
      files: [],
      qrlist: [],
      qrlistfile: [],
      reqrcodearr: [],
      loadprogress: {
        total: 0,
        finished: 0
      },
      loading: false,
      canvas: null,
      content: null,
      canvascfg: {
        multiple: 0,
        width: 0,
        height: 0
      },
      dragcfg: {
        width: 0,
        height: 0,
        top: 0
      },
      loading: false,
      active: 0,
      lockAspect: false,
      dragCom: false,
    };
  },
  computed: {
    loaddone() {
      return (
        this.loadprogress.total > 0 &&
        this.loadprogress.finished === this.loadprogress.total
      );
    },
    available() {
      if(this.qrlist.length > 0 && this.imageUrl != '') {
        return true;
      }
      return false;
    },
    candownload() {
      if(this.content) {
        return true;
      }
      return false;
    },
    dragParent() {
      // this.$nextTick(() => {
        return {
          height: this.canvascfg.multiple * this.canvascfg.height,
          width: Math.round(this.canvascfg.multiple * this.canvascfg.width)
        }
      // });
    }
  },
  watch: {
    qrlist() {
      this.$nextTick(() => {
        console.log(this.qrlist);
      });
    },
    imageUrl() {
      this.$nextTick(() => {
        // console.log(this.imageUrl);
      });
    },
    canvascfg() {

    },
  },
  methods: {
    inputFilter(newFile, oldFile, prevent) {
      // this.qrlist = [];
      if (newFile && !oldFile) {
        if (!/\.(jpeg|jpe|jpg|gif|png|webp)$/i.test(newFile.name)) {
          return prevent();
        }
        this.qrlist.push(newFile.file);

        newFile.blob = ''
        let URL = window.URL || window.webkitURL
        if (URL && URL.createObjectURL) {
          newFile.blob = URL.createObjectURL(newFile.file);
        }
        if(this.thumbs.length < 9){
          this.thumbs.push(newFile.blob);
        }
      }
    },
    handleRemove(file) {
      const card = this.canvas;
      card.dispose();
      this.imageUrl = "";
      this.dragCom = false;
    },
    clearQRcode(file) {
      this.qrlist = [];
      this.$refs.upload.clear();
      this.thumbs = [];
      this.outlog = [];
      this.content = null;
      this.active = 0;
    },
    //当上传图片后，调用onchange方法，获取图片本地路径
    onchange(file, fileList) {
      var _this = this;
      var event = event || window.event;
      if (typeof event.dataTransfer != "undefined") {
        var file = event.dataTransfer.files[0];
      } else {
        var file = event.target.files[0];
      }
      var reader = new FileReader();
      reader.onload = function(e) {
        _this.imageUrl = e.target.result; //将图片路径赋值给src
      };
      reader.readAsDataURL(file);
      setTimeout(() => {
        this.doDraw(this.imageUrl);
      }, 100);
    },
    handleOpen(key) {
      if (this.currentPage === key) {
        return;
      } else {
        this.currentPage = key;
      }
    },
    doDraw(imageUrl) {
      const card = new fabric.Canvas("poster");
      const that = this;
      fabric.Image.fromURL(imageUrl,async img => {
        that.canvascfg.b = img.height / 600;
        // that.$set(that.dragcfg, 'height', img.height);
        // that.$set(that.dragcfg, 'width', img.width);
        that.canvascfg.height = img.height;
        that.canvascfg.width = img.width;
        let rectX = 0;
        let rectY = 0
        if(img.height > img.width){
          that.canvascfg.multiple = 600 / img.height;
          card.setWidth(img.width * that.canvascfg.multiple);
          card.setHeight(img.height * that.canvascfg.multiple);
          rectX = 50 + img.width * that.canvascfg.multiple / 2;
          rectY = 300;
          that.$nextTick(() => {
            that.dragCom = true;
          })
        } else {
          that.canvascfg.multiple = 500 / img.width;
          card.setWidth(img.width * that.canvascfg.multiple);
          card.setHeight(img.height * that.canvascfg.multiple);
          rectX = 250;
          rectY = img.height * that.canvascfg.multiple / 2;
          that.$nextTick(() => {
            that.dragCom = true;
          })
        }
        img.set({
          // 通过scale来设置图片大小，这里设置和画布一样大
          scaleX: card.width / img.width,
          scaleY: card.height / img.height
        });
        // 设置背景
        that.x = Math.round(rectX) -100;
        that.y = Math.round(rectY) -50;
        card.setBackgroundImage(img, card.renderAll.bind(card));
        card.renderAll();
        if(img.height > img.width){
          await that.dragSet(Math.round(600 / img.height * img.width), 600 , 600);
        } else {
          await that.dragSet(500, Math.round(500 / img.width * img.height) , Math.round(500 / img.width * img.height));
        }
      });
      this.canvas = card;
      this.lockAspect = true;
      console.log(this.dragcfg);
    },
    dragSet(width, height ,top) {
      return new Promise(resolve => {
        this.$set(this.dragcfg, 'height', height);
        this.$set(this.dragcfg, 'width', width);
        this.$set(this.dragcfg, 'top', -top);
        // this.onResize(100, 50, 100, 100);
        this.lockAspect = true;
      })
    },
    onResize(x, y, width, height) {
      this.x = x;
      this.y = y;
      this.w = width;
      this.h = height;
      this.$refs.imgRef.width = width;
      this.$refs.imgRef.height = height;
      localStorage.setItem("axiscfg", JSON.stringify({ width, height }));
    },
    onDrag(x, y) {
      this.x = x;
      this.y = y;
      localStorage.setItem("axis", JSON.stringify({ x, y }));
    },
    BeforeQRUpload(file) {
      const isIMAGE = file.type === "image/jpeg" || "image/gif" || "image/png";
      const isLt1M = file.size / 1024 / 1024 < 1;

      if (!isIMAGE) {
        this.$message.error("上传文件只能是图片格式!");
      }
      if (!isLt1M) {
        this.$message.error("上传文件大小不能超过 1MB!");
      }
      return isIMAGE && isLt1M;
    },
    getQRfile(file, fileList) {
      var _this = this;
      var event = event || window.event;
      console.log(event.dataTransfer.files[0]);
      var reader = new FileReader();
      reader.onload = function(e) {
        console.log(e);
      };
    },
    dataURItoBlob(dataURI) {
      // base64 解码
      let byteString = window.atob(dataURI.split(",")[1]);
      let mimeString = dataURI
        .split(",")[0]
        .split(":")[1]
        .split(";")[0];
      let T = mimeString.split("/")[1];
      let ab = new ArrayBuffer(byteString.length);
      let ia = new Uint8Array(ab);
      for (let i = 0; i < byteString.length; i++) {
        ia[i] = byteString.charCodeAt(i);
      }
      return new Blob([ab], { type: mimeString });
    },
    readFileAsync(file) {
      return new Promise(resolve => {
        const reader = new FileReader();
        reader.onload = evt => resolve(this.dataURItoBlob(evt.target.result));
        reader.readAsDataURL(file);
      });
    },
    async readAllFile() {
      let qrlistfile = [];
      for (let k in this.qrlist) {
        qrlistfile[k] = await this.readFileAsync(this.qrlist[k]);
      }
      this.active++;
      return qrlistfile;
    },
    backCanvasSize(canvas) {
      return new Promise(resolve => {
        canvas.backgroundImage.set({
          scaleX: this.canvascfg.multiple,
          scaleY: this.canvascfg.multiple
        })
        resolve();
      })
    },
    async paint() {
      this.loading = true;
      this.outlog = [];
      this.content = null;
      this.active = 0;
      this.loadprogress.finished = 0;
      this.loadprogress.total = this.qrlist.length;
      let files = await this.readAllFile();
      let cfg = await this.getAxis();
      let x = cfg.axis.x;
      let y = cfg.axis.y;
      let width = cfg.axiscfg.width;
      let height = cfg.axiscfg.height;
      var zip = new JSZip();
      const limit = pLimit(3);

      await Promise.all(
        files.map((file, key) => {
          return limit(async() => {
            let result = await this.qrdecode(file, key);
            if(result != '解析失败') {
              let reqrcode = await this.reqrcode(result, key);
              let card = await this.qrcodeCanvas(reqrcode, x, y, width, height);
              card.backgroundImage.set({
                scaleX: 1,
                scaleY: 1
              });
              let url = card.toDataURL({
                format: "jpeg", // jpeg或png
                quality: 1,
                height: this.canvascfg.height,
                width: this.canvascfg.width
              });
              await this.qrcodeCanvasDeleteImg(card._objects[0]);
              await this.addInZip(zip, url.substring(22), this.qrlist[key].name);
            }
          })
        })
      );
      this.active++;
      // let reqrcodearr = await this.reqrcodeAll();

      // let urls = [];
      // var zip = new JSZip();
      // for (let k in reqrcodearr) {
      //   this.outlog.push(this.log[k]);
      //   let card = await this.qrcodeCanvas(reqrcodearr[k], x, y, width, height);
      //   let url = card.toDataURL();
      //   await this.qrcodeCanvasDeleteImg(card._objects[0]);
      //   await this.addInZip(zip, url.substring(22), this.qrlist[k].name);
      //   // this.loadprogress.finished++;
      // }
      const content = await zip.generateAsync({
        type: "blob",
        compression: "DEFLATE"
      });
      this.content = content;
      await this.downloadZip(content);
    },
    addInZip(zip, canvas, name) {
      return new Promise((resolve, reject) => {
        try {
          zip.file(name, canvas, { base64: true });
          resolve();
        } catch (err) {
          console.log(err);
        }
      });
    },
    async qrcodeCanvasDeleteImg(canvas) {
      let card = this.canvas;
      card.backgroundImage.set({
        scaleX: this.canvascfg.multiple,
        scaleY: this.canvascfg.multiple
      });
      card.remove(canvas);
      card.renderAll();
    },
    qrcodeCanvas(imgsrc, x, y, width, height) {
      return new Promise(resolve => {
        const card = this.canvas;
        const that = this;
        let canvas = card.getObjects();
        fabric.Image.fromURL(imgsrc, img => {
          img
            .scaleToHeight(width)
            .scaleToWidth(height)
            .set({
              // width: width / img.scaleX,
              // height: height / img.scaleY,
              // top: y - 50,
              // left: x - 100
              width: (width / img.scaleX) / that.canvascfg.multiple,
              height: (height / img.scaleY) / that.canvascfg.multiple,
              top: y / that.canvascfg.multiple,
              left: x / that.canvascfg.multiple
            });
          card.add(img);
          resolve(card);
        });
      });
    },
    getAxis() {
      return new Promise(resolve => {
        let axis = JSON.parse(localStorage.getItem("axis"));
        let axiscfg = JSON.parse(localStorage.getItem("axiscfg"));
        resolve({ axis, axiscfg });
      });
    },
    async reqrcodeAll() {
      let result = await this.qrdecodeAll();
      let reqrcodearr = [];
      for (let k in result) {
        reqrcodearr[k] = await this.reqrcode(result[k], k);
      }
      return reqrcodearr;
    },
    reqrcode(content, key) {
      return new Promise(resolve => {
        let that = this;
        let opts = {
          errorCorrectionLevel: "H",
          type: "image/jpeg",
          quality: 0.5,
          margin: 1
        };
        QRCode.toDataURL(content, opts, function(err, url) {
          if (err) throw err;
          resolve(url);
        });
      });
    },
    async qrdecodeAll() {
      let files = await this.readAllFile();
      return await Promise.all(
        files.map((file, key) => this.qrdecode(file, key))
      );
    },
    async qrdecode(file, key) {
      try {
        const formData = new FormData();
        formData.append("file", file);
        const { data } = await this.axios({
          method: "post",
          url: url,
          //https://oms.keqin-it.com/pyqrcode/decode
          //https://box3.sdpku.com/pyqrcode/decode
          headers: {
            "Content-Type": `multipart/form-data; boundary=${formData._boundary}`
          },
          data: formData,
        });
        this.outlog.push({
          name: this.qrlist[key].name,
          text: data.data[0]
        });
        this.$refs.table.bodyWrapper.scrollTop = this.$refs.table.bodyWrapper.scrollHeight + 100;
        this.loadprogress.finished++;
        return data.data[0];
      } catch (err) {
        return false;
      }
    },
    downloadZip(content) {
      // let content = this.content;
      saveAs(content, "二维码海报"+ (new Date).toLocaleDateString() +".zip");
      this.loading = false;
      this.active++;
    }
  },
  mounted() {
    // this.createCanvas();
    // this.qrdecode();
    console.dir((new Date).toLocaleTimeString())
  }
};
</script>

<style>
.el-steps {
  margin-top: 20px
}
.pointpoint {
  float: right;
  margin: -40px 40px 0 0;
  font-size: 25px
}
.thumbParent {
  margin-top: 5px;
  width: 500px;
  height: 120px;
  display: flex;
  flex-flow: row wrap;
  align-content: flex-start;
}
.thumbChild {
  height: 60px;
  width: 90px;
  flex: 0 0 20%;
}
#btn-btn-primary {
}
#fileuploadrect {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 30px;
  width: 565px;
  height: 100px;
  padding: 1em;
  border: 1px dashed transparent;
  background: linear-gradient(white, white) padding-box,
    repeating-linear-gradient(
      -45deg,
      #ccc 0,
      #ccc 0.25em,
      white 0,
      white 0.75em
    );
}
.el-dialog__body {
  /* height: 800px; */
  overflow: auto;
}
body {
  -webkit-touch-callout: none;
  font-family: -apple-system-font, BlinkMacSystemFont, "Helvetica Neue",
    "PingFang SC", "Hiragino Sans GB", "Microsoft YaHei UI", "Microsoft YaHei",
    Arial, sans-serif;
  color: #333;
  letter-spacing: 0.034em;
}
html,
body,
#app,
.app-container {
  min-height: 100%;
  height: 100%;
  box-sizing: border-box;
  margin: 0;
}
.menu,
.menu > .el-menu {
  height: 100%;
}
#muban .el-upload-dragger {
  /* display: flex; */
  /* margin-top: 50px;
  margin-left: 200px; */
  height: 600px;
  width: 500px;
}
.el-upload-dragger .el-icon-upload {
  margin-top: 250px;
}
#qrcode .el-upload-dragger {
  margin-top: 30px;
  height: 200px;
  width: 400px;
}

.el-progress {
  margin-top: 20px;
  margin-bottom: 20px;
  width: 400px;
}
.logo img {
  width: 40px;
  height: 19px;
  vertical-align: text-bottom;
  color: #555;
}
.el-menu-item {
  border-left: solid 6px transparent;
}
.el-menu-item.is-active {
  border-left: solid 6px #409eff;
  background-color: #ecf5ff;
}
.link {
  cursor: pointer;
  margin-left: 20px;
  font-size: 12px;
  color: #002f81;
  text-decoration: underline;
}
.example-drag label.btn {
  margin-bottom: 0;
  margin-right: 1rem;
}
.example-drag .drop-active {
  top: 0;
  bottom: 0;
  right: 0;
  left: 0;
  position: fixed;
  z-index: 9999;
  opacity: 0.6;
  text-align: center;
  background: #000;
}
.example-drag .drop-active h3 {
  margin: -0.5em 0 0;
  position: absolute;
  top: 50%;
  left: 0;
  right: 0;
  -webkit-transform: translateY(-50%);
  -ms-transform: translateY(-50%);
  transform: translateY(-50%);
  font-size: 40px;
  color: #fff;
  padding: 0;
}
</style>
