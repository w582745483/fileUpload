<template>
  <div>
    <slot :upload="upload" :handlePrepareUpload="handlePrepareUpload" :fileList="fileList"></slot>
  </div>
</template>
<script>
import SparkMD5 from 'spark-md5';
export default {
  props: {
    options: {
      type: Object,
      default: () => {}
    }
  },
  data() {
    return {
      fileList: [],
      chunkList: []
    };
  },
  methods: {
    upload(e, deps) {
      console.log(e.target.files);
      this.fileList = [...e.target.files];
      deps(this.fileList);
    },
    getChunkMd5(result) {
      const spark = new SparkMD5.ArrayBuffer();
      spark.append(result);
      const chunkmd5 = spark.end();
      return chunkmd5;
      // console.log('切片的MD5', chunkmd5);
    },
    prepareUpload(file) {
      const fileName = file.name;
      const fileSize = file.size; // 文件大小
      const chunkSize = 1024 * 1024 * 3; // 切片的大小 3M
      const chunks = Math.ceil(fileSize / chunkSize); // 获取切片个数
      console.log(`文件:${fileName},切片个数${chunks}`);
      const fileReader = new FileReader();
      const spark = new SparkMD5.ArrayBuffer();
      const bolbSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice;
      let currentChunk = 0;

      fileReader.onload = async(e) => {
        spark.append(e.target.result);
        const chunkmd5 = this.getChunkMd5(e.target.result);
        const chunkItem = this.chunkList.find(item => item[fileName]);
        console.log('chunkItem', chunkItem, fileName, currentChunk);
        if (chunkItem) {
          const chunk = chunkItem[fileName].find(item => item.index === currentChunk);
          if (chunk) {
            chunk.offset = currentChunk * chunkSize;
            chunk.md5 = chunkmd5;
            console.log('chunk', this.chunkList);
          }
        }
        currentChunk++;
        // 文件分析进度
        this.$emit('analysisProgress', { fileName, progress: Math.round(currentChunk / chunks * 100) + '%' });
        if (currentChunk < chunks) {
          loadNext();
        } else {
          const md5 = spark.end();
          console.log('解析完成');
          console.log('此值为整个切片的MD', md5);
          // const params = new FormData();
          const filePreLoadParams = {
            name: fileName,
            size: fileSize,
            md5,
            origin_type: this.options.origin_type || 'INTERNAL',
            sub_type: this.options.sub_type || '',
            over_write: false,
            upload_token: '',
            external: ''
          };

          // 文件预请求
          let { data, code } = await this.filePreupload(filePreLoadParams);

          if (code === 2003) {
            await this.confirm();
            filePreLoadParams.over_write = true;
            const { data: ow_data, code: ow_code } = await this.filePreupload(filePreLoadParams);
            data = ow_data;
            code = ow_code;
          }
          // 文件预请求，返回2001，文件已经上传成功，不用再上传
          if (code === 2001) {
            this.$emit('fileUploadProgress', { fileName, progress: '100%' });
            // this.clearFileChunkList();
            return;
          }
          const file_token = data?.file_token;

          // 获得file_token 给chunks列表增加获得file_token
          const chunkItem = this.chunkList.find(item => item[fileName]);
          console.log('chunkItem', chunkItem);
          chunkItem[fileName].map(item => {
            item.file_token = file_token;
          });

          const fileProgress = [];
          // 所有遍历chunk 预请求
          Promise.all(chunkItem[fileName].map(async chunk => {
            // const params = new FormData();
            const chunkPreLoadParams = {
              file_token: chunk.file_token,
              md5: chunk.md5,
              offset: chunk.offset,
              length: chunk.length
            };

            console.log('chunkPreLoadParams', chunkPreLoadParams);
            // 并发预请求chunk
            // const { file_token, chunk_upload_token, code } = await
            const { data, code: preCode } = await this.axios({
              url: '/api/v2/file/chunk/preupload',
              method: 'post',
              data: chunkPreLoadParams,
              stopCancelSame: true
              // headers: { 'Content-Type': 'multipart/form-data' }
            });
            if (preCode === 2001) {
              return;
            }
            const params = new FormData();
            chunkPreLoadParams.chunk_upload_token = data?.chunk_upload_token;
            chunkPreLoadParams.chunk_data = chunk.chunk_data;
            for (const key in chunkPreLoadParams) {
              params.append(key, chunkPreLoadParams[key]);
            }
            const result = await this.axios({
              url: '/api/v2/file/chunk/upload',
              method: 'post',
              data: params,
              stopCancelSame: true,
              headers: { 'Content-Type': 'multipart/form-data' },
              onUploadProgress: progressEvent => {
                const chunkPro = fileProgress.find(item => item.chunk_upload_token == data?.chunk_upload_token);
                if (chunkPro) {
                  chunkPro.progress = Math.round((progressEvent.loaded / progressEvent.total) * 100);
                } else {
                  fileProgress.push({
                    chunk_upload_token: data?.chunk_upload_token,
                    progress: Math.round((progressEvent.loaded / progressEvent.total) * 100)
                  });
                }
                window.fileProgress = fileProgress;

                console.log(progressEvent);
                const accumulator = fileProgress.reduce((acc, curr) => {
                  return acc + curr.progress;
                }, 0);
                this.$emit('fileUploadProgress', { fileName, progress: Math.round(accumulator / chunks) + '%' });

                console.log('fileName:' + fileName + '-----' + accumulator / chunks);
              }
            });
            return result;
          })).then(async result => {
            console.log('result', result);
            const { code: verifyCode } = await this.axios({
              url: '/api/v2/file/verify',
              method: 'post',
              data: { file_token },
              stopCancelSame: true
            });
            if (verifyCode !== 0) {
              this.$message('文件上传失败');
            } else {
              this.$emit('fileUploadProgress', { fileName, progress: '100%' });
            }

            this.clearFileChunkList();
          }).catch(err => {
            console.log('err', err);
          });
        }
      };

      fileReader.onerror = function() {
        console.warn('oops, something went wrong');
      };

      const loadNext = () => {
        const start = currentChunk * chunkSize;
        const end = start + chunkSize > file.size ? file.size : start + chunkSize;
        const slice = bolbSlice.call(file, start, end);
        console.log('slice', slice);
        const chunkItem = this.chunkList.find(item => item[fileName]);
        if (chunkItem) {
          chunkItem[fileName].push({ index: currentChunk, length: slice.size, chunk_data: slice, offset: '', file_token: '', chunk_upload_token: '' });
        } else {
          this.chunkList.push({ [fileName]: [{ index: currentChunk, length: slice.size, chunk_data: slice, offset: '', file_token: '', chunk_upload_token: '' }] });
        }
        fileReader.readAsArrayBuffer(slice);
      };
      loadNext();
    },
    handlePrepareUpload() {
      this.fileList.map(file => {
        this.prepareUpload(file);
      });
    },
    clearFileChunkList() {
      this.chunkList = [];
      this.fileList = [];
    },
    async filePreupload(params) {
      // 文件预请求
      const { data, code } = await this.axios({
        url: '/api/v2/file/preupload',
        method: 'post',
        data: params,
        stopCancelSame: true
      });
      return {
        data,
        code
      };
    },
    async confirm() {
      const result = await this.$confirm("是否覆盖文件", "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: 'warning'
      });
      return result;
    }
  }
};
</script>
