<template>
  <div>
    <div class="img-content">
      <div class="img-inline" @click="sign" v-for="(img,index) in imgList" :key="index" @mouseover="mover(index)" @mouseout="mout('')">
       <div class="item">
         <div class="box">
           <div v-if="isState&&isEdit=='edit'&&index===activeIndexs" class="percentage-box-edit">
             <el-progress :width="100" type="circle" :color="colors" :percentage="percentage"></el-progress>
           </div>
           <el-image v-else :src="http+img" style="max-width: 148px;max-height: 148px;">
                 <div slot="error" class="image-slot">
                   <i class="el-icon-picture-outline"></i>
                 </div>
                 <div slot="placeholder" class="image-slot">
                    <i class="el-icon-loading" style="font-size: 28px;"></i>
                    <p>加载中</p>
                 </div>
           </el-image>
         </div>
       </div>
       <!-- v-show="index===thisIndex" -->
       <div class="show" v-show="index===thisIndex">
          <div class="btn-list">
             <div class="btn-box">
               <!-- @click="maxShowImg(index)" -->
                <div class="btn-item" title="查看" @click="maxShowImg(index)"><i class="el-icon-view"></i></div>
                <div class="btn-item" title="编辑">
                  <el-upload :http-request="httpRequest" :on-change="change" :accept="accept"  class="edit-upload-el" :action="http+parame.action" :data="parame.data" list-type="picture" :before-upload="beforeUpload" :on-success="editSuccess" :show-file-list="false">
                    <i class="el-icon-edit-outline"></i>
                  </el-upload>
                </div>
                <div class="btn-item" title="删除" @click="delImg(index)"><i class="el-icon-delete"></i></div>
             </div>
          </div>
       </div>
      </div>
      <div @mouseover="addMover" class="img-inline" v-show="imgList.length<parame.limit">
        <UploadList @onSaveAll="saveAll" ref="MyUploadList" :parame="parame" :limit="parame.limit-this.imgList.length" @setBatchHash="batchHash" v-if="isBatch"></UploadList>
        <div v-else>
          <el-upload :http-request="httpRequest" :on-change="change" :ref="name" :accept="accept" :action="http+parame.action" :data="parame.data" list-type="picture" :before-upload="beforeUpload" :on-success="handSuccess" :show-file-list="false">
            <div class="el-upload el-upload--picture-card">
              <div v-if="isState&&isEdit=='add'" class="percentage-box">
                <el-progress :width="100" type="circle" :color="colors" :percentage="percentage"></el-progress>
              </div>
              <i v-else class="el-icon-upload" v-bind:class="{'el-icon-plus':parame.limit>1}"></i>
             </div>
          </el-upload>
        </div>

      </div>
    </div>
    <!-- 图片放大 -->
    <el-dialog
        :visible.sync="dialogVisible"
        :append-to-body="true"
        :center="true"
        top="15vh"
        width="60%">

         <div class="size-img">
           <div class="size-img-box">
             <el-image title="点击预览"  ref="size_img" :src="http+imgSrc" :preview-src-list="[http+imgSrc]"> </el-image>
             <p class="red">点击图片预览</p>
           </div>
         </div>
    </el-dialog>
    <!-- 图片裁剪  -->
    <div v-if="isCrop">
        <el-dialog
          :visible.sync="drawer"
          :close-on-click-modal="false"
          :append-to-body="true"
          :top="croppParame.topCrop"
          :width="croppParame.croppersW">
          <div  v-bind:style="{height:croppParame.croppersH}">
            <ImgCrop :img="cropimg" :croppParame="croppParame" @getCropDatas="getCropDatas"/>
          </div>
        </el-dialog>
        <!-- 裁剪显示框 -->
        <el-dialog
          :visible.sync="isShowBlob"
          :append-to-body="true"
          :top="croppParame.top"
          :center="true"
          :close-on-click-modal="true">
            <div class="cropper-img">
              <el-image title="点击预览" :src="blobimg" :preview-src-list="[blobimg]"> </el-image>
            </div>
            <div slot="footer">
             <div class="pmsg">点击图片预览</div>
             <el-button title="上传到服务器" size="small" type="primary" @click="uploadCropper">立即上传<i class="el-icon-upload el-icon--right"></i></el-button>
            </div>
        </el-dialog>
    </div>


  </div>
</template>

<script>
  import ImgCrop from '@/components/imgCrop';//图片裁剪
  import ImgPreview from '@/components/preview';//图片放大
  let SparkMD5 = require('spark-md5'); //图片加密
  //批量上传
  import UploadList from '@/components/uploadList';
   export default {
     components:{ImgCrop,ImgPreview,UploadList},
     name:"upload",
     data(){
       return{
        // 单独上传进度管理
        percentage:0,
        isState:false,
        colors: [
            {color: '#f56c6c', percentage: 20},
            {color: '#e6a23c', percentage: 40},
            {color: '#5cb87a', percentage: 60},
            {color: '#1989fa', percentage: 80},
            {color: '#409EFF', percentage: 100}
        ],
        maxSize:2, //上传分片大所有文件统一设置
        isShowBlob:false,
        dialogVisible: false, //查看大图盒子
        thisIndex:"",//当前遮罩下标
        activeIndex:"",//选中编辑的下标
        activeIndexs:"",//图片编辑图片选中后的下标（避免防止图片下标错误）
        imgSrc:"",      //放大图片当前地址
        loading:null,   //当前加载框
        isEdit:"",      //编辑||添加
        http:this.$http, //配置当前服务器地址
        accept:"image/png,image/jpeg,image/jpg,image/gif,image/pjpeg,image/bmp,image/x-png",//上传图片类型限制
        // 裁剪相关
        cropimg:"",     //需要裁剪图片
        drawer:false,   //显示裁剪图片框
        blobimg:"",
        croppParames:{   //裁剪配置
            croppersW:"800px",//裁剪显示区域大小（像素）
            croppersH:"400px",//裁剪显示区域大小（像素）
            autoCropWidth:"600", //裁剪框的宽度（无）
            autoCropHeight:"200", //裁剪框的高度（无）
            top:"30vh",           //裁剪后弹窗顶部距离
            topCrop:"20vh"       //裁剪前弹窗顶部距离
        },
       }
     },
     props:{
       parame:Object,   //上传参数
       imgList:Object|Array, //已经上传的图片列表
       item:Number|String,   //标记唯一上传组件
       name:String,          //标记唯一组name
       isCrop:Boolean,       //是否开启裁剪图片功能
       croppParame:Object    ,//裁剪需要的参数
       isBatch:Boolean //是否开启批量上传
     },
     methods:{
       sign(){
         this.activeIndexs=this.thisIndex;
       },
       saveAll({res}){
         if(res.code===200){
            this.setData({type:"add",index:this.imgList.length},res.data.path);
         }
       },
       uploadCropper(){
         let {action,data}=this.parame;
         //上传图片裁剪
         this.loading = this.$loading({
            lock: true,
            text: '正在上传',
            spinner: 'el-icon-loading',
            background: 'rgba(0, 0, 0, 0.7)'
         });
          this.$request({
              type:"post",
              url:action,
              data:{path:data.path,isFileType:'base64',base64:this.blobimg},
              success:(res)=>{
                this.loading.close();
                if(res.code===200){
                  if(this.isEdit==="add"){
                    this.setData({type:"add",index:this.imgList.length},res.data.path);
                  }
                  if(this.isEdit==="edit"){
                    this.setData({type:"edit",index:this.activeIndexs},res.data.path);
                  }
                  this.isShowBlob=false;
                  this.drawer=false;
                }else{
                  this.$message({showClose: true,message: res.msg,type: 'error'});
                }
              },error:(err)=>{
                this.$message.error(err);
              }
          });

       },
       getCropDatas({data}){
         //裁剪完成
         this.isShowBlob=true;
         this.blobimg=data;
       },
       handSuccess(res,file, fileList){
          //图片添加上传成功
          setTimeout(()=>{
            this.loading.close();
            if(res.code===200){
              this.setData({type:"add",index:this.imgList.length},res.data.path);
            }else{
              this.$message.error(res.msg);
            }
           // this.$refs[this.name].clearFiles();
          }, 300);
        },
        editSuccess(res,file, fileList){
          //编辑图片重新上传成功
          setTimeout(()=>{
            this.loading.close();
            if(res.code===200){
               this.setData({type:"edit",index:this.activeIndexs},res.data.path);
            }else{
              this.$message.error(res.msg);
            }
          }, 300);
        },
        mover(index){
          //移入当前元素
           this.thisIndex=index;
           this.activeIndex=index;
           this.isEdit="edit";
           this.isshow=true;
        },
        mout(){
          //移除当前元素
           this.thisIndex="";
        },
        addMover(){
          this.isEdit="add";
        },
        change(file){
          if(this.isCrop){
            this.drawer=true;
            this.cropimg=file.url;
          }
        },
        beforeUpload(file,fileList){
          //return false;
         // this.activeIndexs=this.activeIndex;
          if(this.isCrop){
            return false;
          }
          this.loading = this.$loading({
             lock: true,
             text: '正在上传',
             spinner: 'el-icon-loading',
             background: 'rgba(0, 0, 0, 0.7)'
          });
        },
        maxShowImg(index){
          //查看大图
          this.imgSrc=this.imgList[index];
          this.dialogVisible=true;
        },
        delImg(index){
          this.setData({type:"del",index:index});
          this.mout()
        },
        setData(set,imgUrl=""){
          //console.log(set,imgUrl);
           let data={
             name:this.name,
             imgUrl:imgUrl, //图片地址
             index:this.item, //第几个上传对象
             set:set          //上传第几张图片
           }
           this.$emit("onSuccess",data);
        },
        //自定义分片上传
        upFile(data,info){
           let parame=this.parame;
           info=Object.assign({},info,parame.data);
           info.isFileType="sheet";

           this.$request({
               type:"file",
               url:parame.action,
               data:{
                 arrForm:data,
                 info:info,
                 uploading:(e,lent,sum)=>{
                   //进度条
                   this.isState=true;
                   this.percentage=(lent/sum)*100;
                 }
               },
               success:(res)=>{
                 setTimeout(()=>{
                  this.isState=false;
                  this.percentage=0;
                 },300)

                 if(this.isEdit==='add'){
                    this.handSuccess(res);
                 }
                 if(this.isEdit==='edit'){
                   this.editSuccess(res);
                 }
               },error:(err)=>{
                  this.$message.error(err);
               }
           });

        },
        httpRequest(d){

          let file=d.file;
          let type=file.type;

          var blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice;//兼容
          let bytesPerPiece=this.maxSize*1024 * 1024;//每个文件切片大小定为1MB .1048576=1MB
          let chunks = Math.ceil(file.size /bytesPerPiece); //计算文件切片总份数

          // let sizes = Math.ceil(file.size/1024);
          // if(sizes>500){
          //   this.$message({message: '请上传不要超过500KB的文件,请分批上传！', type: 'warning'});
          //   return false;
          // }


          this.fielSlice(file,(hexHash)=>{
            let arrForm=[];
            for(var i=0;i<chunks;i++){
                const start = i * bytesPerPiece;
                const end = Math.min(file.size, start + bytesPerPiece);
               // console.log(blobSlice.call(file, start, end));
                let form=new FormData();
                //图片切片
                form.append('file',blobSlice.call(file, start, end));
                form.append('isFileType','sheet');
                form.append('index', i);
               // console.log(start,file.size);
                arrForm.push(form)
            }
            this.upFile(arrForm,{
              chunks:chunks, //分片总数
              hash:hexHash, //机密的唯一文件
              size:file.size,  //大小
              name:file.name   //文件名称
            });

          },(err)=>{
            this.$message({message: '文件加载失败！', type: 'error'});
          });

         // this.loading.close();
        },
        //加密文件唯一标示
        fielSlice(file,success,err){
          var blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice;//兼容
          let bytesPerPiece=this.maxSize*1024 * 1024;//每个文件切片大小定为1MB .1048576=1MB
          let fileSize=file.size; //文件大小
          let fileName=file.name; //文件名称
          let chunks = Math.ceil(fileSize /bytesPerPiece); //计算文件切片总份数
          let currentChunk = 0; //读取第几块

          let spark = new SparkMD5.ArrayBuffer(); //二进制加密对象
          let fileReader = new FileReader();  //读取文件对象

          fileReader.onload=(e)=>{
            //读取文件操作完成
            //console.log(`第${currentChunk}分片解析完成，开始解析${currentChunk + 1}分片`,'共'+chunks+'份数');
            spark.append(e.target.result);
            currentChunk++;
            if (currentChunk < chunks) {
                var start = currentChunk * bytesPerPiece;
                let end = ((start + bytesPerPiece) >= fileSize) ? fileSize : start + bytesPerPiece;
                fileReader.readAsArrayBuffer(blobSlice.call(file, start, end));
            } else {
                const result=spark.end();
                const sparkMd5 = new SparkMD5();
                sparkMd5.append(result);
                sparkMd5.append(file.name);
                const hexHash = sparkMd5.end();
                success(hexHash)
            }
          }
          fileReader.onerror = function () {
            //读取文件发生错误
             err('oops, something went wrong.')
          };
          var start = currentChunk * bytesPerPiece;
          let end = ((start + bytesPerPiece) >= fileSize) ? fileSize : start + bytesPerPiece;
          fileReader.readAsArrayBuffer(blobSlice.call(file, start, end));
        },
        batchHash({file}){
          var blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice;//兼容
          let bytesPerPiece=this.maxSize*1024 * 1024;//每个文件切片大小定为1MB .1048576=1MB
          let chunks = Math.ceil(file.size /bytesPerPiece); //计算文件切片总份数

            this.fielSlice(file,(hexHash)=>{
              let arrForm=[];
              for(var i=0;i<chunks;i++){
                  const start = i * bytesPerPiece;
                  const end = Math.min(file.size, start + bytesPerPiece);
                  let form=new FormData();
                  //图片切片
                  form.append('file',blobSlice.call(file, start, end));
                  form.append('isFileType','sheet');
                  form.append('index', i);
                  arrForm.push(form)
              }
              let sizes = Math.ceil(file.size/1024);
              this.$refs.MyUploadList.setAllHash({
                 arrForm:arrForm,
                 chunks:chunks,
                 hash:hexHash,
                 size:sizes,
                 name:file.name,
                 isFileType:'sheet'
              });
            },(err)=>{
              this.$message({message: '文件加载失败！', type: 'error'});
            });
        }



     }
   }
</script>

<style scoped>
  .red{
    color: #b3b3b3;
    padding-top: 15px;
    padding-bottom: 15px;
  }
  .img-content{
    overflow: hidden;
  }
  .img-inline{
    float: left;
    width: 148px;
    height: 148px;
    display: table;
    margin-right: 15px;
    margin-bottom: 15px;
    border-radius: 6px;
    overflow: hidden;
    position: relative;
  }
  .img-inline .item{
     display: table-cell;
     vertical-align: middle;
     text-align: center;
     background: #EBEEF5;
     border-radius: 6px;
     border: 1px solid #EBEEF5;
  }
  .box{
     background: #EBEEF5;
     max-height: 148px;
     overflow: hidden;
  }
  .el-image{
    display: inline;
  }
  .show{
    position:absolute;
    left: 0;
    bottom: 0;
    background:rgba(0,0,0,.6) ;
    width: 100%;
    height: 100%;
    /* height: 27px; */
    display: table;
  }
  .btn-list{
    display: table-cell;
    vertical-align: middle;
    text-align: center;
  }
  .btn-box{
    overflow: hidden;
  }
  .btn-item{
    display: inline;
    margin-left: 5px;
    margin-right: 5px;
    cursor: pointer;
    position: relative;
  }
  .sizeElimg{
    position: absolute;
    width:20px;
    height: 20px;
    left: 0;
    top: 0;

    z-index: 9;
  }

  .btn-item i{
    color: #FFFFFF;
    font-size: 20px;
    font-family: element-icons!important;
    font-weight: 400;
    z-index: 1;
    position: relative;
  }

  .edit-upload-el{
    display: inline;
  }
  .size-img{
    width: 100%;
    /* height: 500px; */
    display: table;
  }
  .size-img-box{
    display: table-cell;
    vertical-align: middle;
    text-align: center;
  }
  .size-img-box .el-image>>>img{
    height: auto;
    max-height: 500px;
    width: auto;
    max-width: 100%;
  }
  .cropper-img .el-image>>>img{
    height: auto;
    max-height: 500px;
    width: auto;
    max-width: 100%;
  }
  .percentage-box{
    margin-top: 22px;
  }
  .percentage-box-edit{
     margin-top: 6px;
  }
</style>
