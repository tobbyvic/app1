<template>
  <div class="home">
    <div class="upload">
      <p class="title">批量大文件上传(一次性可选取10个文件上传)</p>
      <form>
        <div class="upload-file">
          <label for="file">请选择文件</label>
          <input
            type="file"
            name="file"
            id="big-file"
            accept="application/*"
            multiple
            @change="onInputChange"
          />
        </div>

        <div v-for="(item, index) in progressList" :key="index">
          <div>{{ nameList[index] }}</div>
          <div>
            <a-progress :percent="item" />
          </div>
        </div>
      </form>
    </div>

    <div class="list">
      <a-input-search
        v-model="searchValue"
        placeholder="请输入搜索关键字"
        enter-button="搜索"
        size="large"
        @search="onSearch"
        @input="onInput"
      />
      <a-table
        style="margin-top: 20px"
        :columns="columns"
        :data-source="data"
        :pagination="{ pageSize: 50 }"
        :scroll="{ y: 600 }"
        :loading="loading"
      >
        <!-- <template slot="operation" slot-scope="text, record">
          <a-popconfirm title="确定下载?" @confirm="() => onDownload(record)">
            <a href="javascript:;">下载</a>
          </a-popconfirm>

          <a-popconfirm
            style="margin-left: 20px"
            title="确定删除?"
            @confirm="() => onDelete(record)"
          >
            <a href="javascript:;">删除</a>
          </a-popconfirm>
        </template> -->

        <!-- <a-table-column key="operation" title="Action">
          <template #default="{ record }">
            <a-popconfirm title="确定下载?" @confirm="() => onDownload(record)">
              <a href="javascript:;">下载</a>
            </a-popconfirm>

            <a-popconfirm
              style="margin-left: 20px"
              title="确定删除?"
              @confirm="() => onDelete(record)"
            >
              <a href="javascript:;">删除</a>
            </a-popconfirm>
          </template>
        </a-table-column> -->
      </a-table>
    </div>
  </div>
</template>

<script>
import axios from "axios";
import SparkMD5 from "spark-md5";
import { message, Progress } from "ant-design-vue";

// 切割文件
function sliceFile(file) {
  const files = [];
  const chunkSize = 1024 * 1024;
  for (let i = 0; i < file.size; i += chunkSize) {
    const end = i + chunkSize >= file.size ? file.size : i + chunkSize;
    let currentFile = file.slice(i, end > file.size ? file.size : end);
    files.push(currentFile);
  }
  return files;
}

// 获取文件md5值
function md5File(files) {
  const spark = new SparkMD5.ArrayBuffer();
  let fileReader;
  for (var i = 0; i < files.length; i++) {
    fileReader = new FileReader();
    fileReader.readAsArrayBuffer(files[i]);
  }
  return new Promise((resolve) => {
    fileReader.onload = function (e) {
      spark.append(e.target.result);
      if (i == files.length) {
        resolve(spark.end());
      }
    };
  });
}

function debounce(fn, delay) {
  let timer = null; //借助闭包
  return function () {
    if (timer) {
      clearTimeout(timer);
    }
    timer = setTimeout(fn, delay); // 简化写法
  };
}

const columns = [
  {
    title: "文件名称",
    dataIndex: "name",
  },
  {
    title: "操作",
    dataIndex: "operation",
    scopedSlots: { customRender: "operation" },
  },
];

const data = [];
for (let i = 0; i < 100; i++) {
  data.push({
    key: i,
    name: `Edward King ${i}`,
    age: 32,
  });
}

// @ is an alias to /src
export default {
  name: "Home",
  components: {},
  data() {
    return {
      href: "javascript: void(0)",

      sliceList: [], //
      progressList: [], //
      md5List: [],
      extList: [],

      nameList: [],

      // list
      data: [],
      columns,
      searchValue: "",
      loading: false,
    };
  },
  created() {
    this.getFileList();
    this.onInput = debounce(this.getFileList, 300);
  },
  methods: {
    onInputChange(e) {
      // 清空暂存数据
      this.sliceList = [];
      this.progressList = [];
      this.md5List = [];
      this.extList = [];

      let arr = Object.values(e.target.files);
      this.nameList = arr.map((f) => f.name);
      // 限制个数
      if (arr.length > 10) {
        message.error("单次最多上传10个文件");
        return false;
      }

      // 单个限制大小为5个G
      if (
        arr.some((file) => {
          return file.size / (1000 * 1000) > 5 * 1024;
        })
      ) {
        message.error("单个文件最大为5G!");
        return false;
      }

      arr.forEach((file, index) => {
        this.uploadBig(file, index);
      });

      // let file = e.target.files[0];
      // if (file.type.indexOf("application") == -1) {
      //   return alert("文件格式只能是文档应用！");
      // }
      // if (file.size / (1000 * 1000) > 2048) {
      //   return alert("文件不能大于2048MB！");
      // }
      // this.uploadBig(file);
    },

    // 上传大文件
    async uploadBig(file, index) {
      let strArray = file.name.split(".");
      let ext = strArray[strArray.length - 1];
      console.log("后缀名", ext);
      // 暂存后缀名
      this.$set(this.extList, index, ext);
      // 切割文件
      let fileArr = sliceFile(file);
      this.$set(this.sliceList, index, fileArr);
      // 加md5值
      let md5Val = await md5File(fileArr);
      this.$set(this.md5List, index, md5Val);
      // 上传切片
      this.uploadSlice(0, index);
    },

    // 上传分片文件
    async uploadSlice(chunkIndex = 0, index) {
      let md5Val = this.md5List[index];
      let fileArr = this.sliceList[index];

      let formData = new FormData();
      formData.append("sampleFile", fileArr[chunkIndex]);

      let data = await axios({
        url: `/api/uploadbig?type=upload&current=${chunkIndex}&md5Val=${md5Val}&total=${fileArr.length}`,
        method: "post",
        data: formData,
      });

      if (data.status == 200) {
        if (chunkIndex < fileArr.length - 1) {
          this.$set(
            this.progressList,
            index,
            Math.round(((chunkIndex + 1) / fileArr.length) * 100)
          );
          ++chunkIndex;
          this.uploadSlice(chunkIndex, index);
        } else {
          this.mergeFile(index);
        }
      }
    },

    // 合并文件
    async mergeFile(index) {
      let md5Val = this.md5List[index];
      let ext = this.extList[index];
      let fileArr = this.sliceList[index];
      let name = this.nameList[index];

      let data = await axios.post(
        `/api/uploadbig?type=merge&md5Val=${md5Val}&total=${fileArr.length}&ext=${ext}&originname=${name}`
      );
      if (data.data.code == 200) {
        // alert("上传成功！");
        message.success(this.nameList[index] + "上传成功!");
        this.progress = 100;

        this.$set(this.progressList, index, 100);
        // 刷新列表
        this.getFileList();
        // this.href = data.data.data.url;
        // this.currURL = data.data.data.url;
        // bigUrls.innerText = data.data.data.url;
      } else {
        message.warn("出了一点问题...");
        console.log(data.data.data.info);
      }
    },

    // 文件列表
    onSearch(val) {
      this.getFileList();
    },
    onDownload(record) {
      // console.log(record)
      window.open(`${window.location.origin}/file/${record.name}`);
    },
    async onDelete(record) {
      try {
        let res = await axios.delete(`/api/filelist?name=${record.name}`);
        message.success(res.data.message);
        this.getFileList();
      } catch (error) {
        message.error(JSON.stringify(error));
      }
    },
    // 获取文件列表
    async getFileList() {
      try {
        this.loading = true;
        let res = await axios.get(`/api/filelist?name=${this.searchValue}`);
        console.log("res", res);
        this.data = res.data.files.map((file, index) => {
          return {
            key: index,
            name: file,
          };
        });
        this.loading = false;
      } catch (error) {
        message.error(JSON.stringify(error));
        this.loading = false;
      }
    },
  },
};
</script>

<style scoped>
body {
  margin: 0;
  font-size: 16px;
  background: #f8f8f8;
}
h1,
h2,
h3,
h4,
h5,
h6,
p {
  margin: 0;
}

/* * {
    outline: 1px solid pink;
} */
.home {
  display: flex;
}

.title {
  font-weight: bold;
  font-size: 18px;
}

.upload {
  flex: 1;
  box-sizing: border-box;
  text-align: left;
  padding: 20px;
  width: 500px;
  border-radius: 15px;
  background: #fff;
}

.list {
  flex: 1;
  padding: 20px;
}

.upload h3 {
  font-size: 20px;
  line-height: 2;
  text-align: center;
}

.upload .upload-file {
  position: relative;
  margin: 10px auto;
}

.upload .upload-file label {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 150px;
  border: 1px dashed #ccc;
}

.upload .upload-file input {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  opacity: 0;
}

.upload-progress {
  display: flex;
  align-items: center;
}

.upload-progress p {
  position: relative;
  display: inline-block;
  flex: 1;
  height: 15px;
  border-radius: 10px;
  background: #ccc;
  overflow: hidden;
}

.upload-progress p span {
  position: absolute;
  left: 0;
  top: 0;
  width: 0;
  height: 100%;
  background: linear-gradient(
    to right bottom,
    rgb(114, 175, 21),
    rgb(14, 111, 93)
  );
  transition: all 0.4s;
}

.upload-link {
  margin: 30px auto;
}

.upload-link a {
  text-decoration: none;
  color: rgb(6, 102, 192);
}

@media all and (max-width: 768px) {
  .home {
    flex-flow: column;
  }
  .upload {
    width: 300px;
  }
}
</style>
