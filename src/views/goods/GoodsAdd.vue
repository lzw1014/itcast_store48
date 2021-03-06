<template>
  <el-card class="card">
    <!-- 面包屑 -->
    <my-breadcrumb level1="商品管理" level2="商品列表"></my-breadcrumb>

    <el-alert
      class="alert"
      title="添加商品信息"
      :closable="false"
      type="info"
      center
      show-icon>
    </el-alert>

    <!-- 步骤条 -->
    <el-steps
      :active="active"
      finish-status="success"
      align-center>
      <el-step title="基本信息"></el-step>
      <el-step title="商品参数"></el-step>
      <el-step title="商品属性"></el-step>
      <el-step title="商品图片"></el-step>
      <el-step title="商品内容"></el-step>
    </el-steps>

    <!-- 选项卡 -->
    <el-form
      label-position="top"
      :model="form"
      label-width="100px">
      <el-tabs
        @tab-click="handleTabClick"
        tab-position="left"
        style="margin-top: 20px">
        <el-tab-pane label="基本信息">
          <!-- 基本信息 -->
          <el-form-item label="商品名称">
            <el-input v-model="form.goods_name"></el-input>
          </el-form-item>
          <el-form-item label="商品价格">
            <el-input v-model="form.goods_price"></el-input>
          </el-form-item>
          <el-form-item label="商品重量">
            <el-input v-model="form.goods_weight"></el-input>
          </el-form-item>
          <el-form-item label="商品数量">
            <el-input v-model="form.goods_number"></el-input>
          </el-form-item>
          <el-form-item label="商品分类">
            {{ selectedOptions }}
            <el-cascader
              placeholder="请选择商品分类"
              clearable
              expand-trigger="hover"
              :options="options"
              :props="{ label: 'cat_name', value: 'cat_id', children: 'children' }"
              v-model="selectedOptions"
              @change="handleChange">
            </el-cascader>
          </el-form-item>
        </el-tab-pane>
        <el-tab-pane label="商品参数">
          <el-form-item
            v-for="item in dynamicParams"
            :key="item.attr_id"
            :label="item.attr_name">
            {{ item.params }}
            <el-checkbox-group v-model="item.params">
              <el-checkbox
                v-for="param in item.params"
                :key="param"
                border
                :label="param">
              </el-checkbox>
            </el-checkbox-group>
          </el-form-item>
        </el-tab-pane>
        <el-tab-pane label="商品属性">
          <el-form-item
            v-for="item in staticParams"
            :key="item.attr_id"
            :label="item.attr_name">
            <el-input v-model="item.attr_vals"></el-input>
          </el-form-item>
        </el-tab-pane>
        <el-tab-pane label="商品图片">
          <!--
            action  上传的地址，此处必须写上全部路径
           -->
          <el-upload
            :headers="headers"
            action="http://localhost:8888/api/private/v1/upload"
            :on-remove="handleRemove"
            :on-success="handleSuccess"
            list-type="picture">
            <el-button size="small" type="primary">点击上传</el-button>
          </el-upload>
        </el-tab-pane>
        <el-tab-pane label="商品内容">
          <el-button @click="handleAdd">添加商品</el-button>
          <quill-editor v-model="form.goods_introduce"></quill-editor>
        </el-tab-pane>
      </el-tabs>
    </el-form>
  </el-card>
</template>

<script>
import 'quill/dist/quill.core.css';
import 'quill/dist/quill.snow.css';
import 'quill/dist/quill.bubble.css';

import { quillEditor } from 'vue-quill-editor';

export default {
  components: {
    quillEditor
  },
  created() {
    // 加载商品分类
    this.loadOptions();
  },
  methods: {
    // 点击tab栏的tab项
    handleTabClick(tab, event) {
      // tab 是当前点击的组件   index=“0”
      // console.log(tab);
      // console.log(event);
      this.active = tab.index - 0;

      // 判断当前点击的tab是否是 商品参数，商品属性
      if (tab.index === '1' || tab.index === '2') {
        // 判断基本信息中的多级下拉是否选中了3级分类
        if (this.selectedOptions.length !== 3) {
          this.$message.warning('请先选择商品的分类');
        } else {
          // 加载商品分类的参数列表
          this.loadParams(tab.index);
        }
      }
    },
    // 多级下拉选中项变化的时候执行
    handleChange() {
      // 让多级下拉，只能选择3级分类
      if (this.selectedOptions.length !== 3) {
        this.$message.warning('请选择3级分类');
        // 清空多级下拉中的内容
        this.selectedOptions.length = 0;
      }
    },
    // 加载多级下拉的数据
    async loadOptions() {
      const response = await this.$http.get('categories?type=3');
      this.options = response.data.data;
    },
    // 加载动态参数
    // 加载静态参数
    async loadParams(index) {
      // 当index='1' 动态参数   index='2'静态参数
      const sel = index === '1' ? 'many' : 'only';

      // many 动态参数  only静态参数
      const response = await this.$http.get(`categories/${this.selectedOptions[2]}/attributes?sel=${sel}`);

      // 判断是动参数
      if (sel === 'many') {
        this.dynamicParams = response.data.data;
        // attr_vals: "4G,8G,16G"

        this.dynamicParams.map((item) => {
          // 给对象新加一个属性
          // 动态给对象增加的成员，无法做双向绑定
          const arr = item.attr_vals.length === 0 ? [] : item.attr_vals.split(',');

          this.$set(item, 'params', arr);
        });
        // this.dynamicParams --> [{attr_vals:'1,2', params: [1, 2]},{attr_vals:''},{attr_vals:''}]
        // console.log(this.dynamicParams);
      } else if (sel === 'only') {
        // 静态参数
        this.staticParams = response.data.data;
      }
    },
    // 图片上传的方法
    handleRemove(file, fileList) {
      // file.response.tmp_path
      // console.log(file);
      // console.log(fileList);
      // 从数组中删除数据，要知道，删除项的索引是多少？
      const index = this.form.pics.findIndex((item) => {
        if (item.pic === file.response.data.tmp_path) {
          return true;
        }
      });
      this.form.pics.splice(index, 1);
      console.log(this.form.pics);
    },
    handleSuccess(response, file, fileList) {
      console.log(response);
      // console.log(file);
      // console.log(fileList);
      // 图片上传成功，在pics中记录当前图片的临时路径
      if (response.meta.status === 200) {
        this.form.pics.push({
          pic: response.data.tmp_path
        });

        console.log(this.form.pics);
      } else {
        this.$message.error('图片上传失败');
      }
    },
    // 添加商品
    async handleAdd() {
      // goods_cat
      this.form.goods_cat = this.selectedOptions.join(',');
      // attrs
      // console.log(this.dynamicParams);
      // console.log(this.staticParams);

      // 基于dynamicParams数组生成一个新的数组
      const arr1 = this.dynamicParams.map((item) => {
        return { 'attr_id': item.attr_id, 'attr_value': item.params.join(',') };
      });
      // 基于staticParams数组生成一个新的数组
      const arr2 = this.staticParams.map((item) => {
        return { 'attr_id': item.attr_id, 'attr_value': item.attr_vals };
      });
      this.form.attrs = [...arr1, ...arr2];

      const response = await this.$http.post('goods', this.form);
      // 添加是否成功
      const { meta: { status, msg } } = response.data;
      if (status === 201) {
        this.$message.success(msg);
        this.$router.push('/goods');
      } else {
        this.$message.error(msg);
      }
    }
  },
  data() {
    return {
      // 绑定步骤条，显示第几步完成
      active: 0,
      // 表单数据
      form: {
        goods_name: '',
        // 商品分类id,  1,2,3
        goods_cat: '',
        goods_price: '',
        goods_number: '',
        goods_weight: '',
        pics: [],
        goods_introduce: '',
        attrs: []
      },
      // 多级选择器绑定的数据
      options: [],
      selectedOptions: [],
      // 动态参数
      dynamicParams: [],
      // 静态参数
      staticParams: [],
      // 绑定上传组件
      headers: {
        'Authorization': sessionStorage.getItem('token')
      }
    };
  }
};
</script>

<style>
.alert {
  margin-top: 10px;
  margin-bottom: 10px;
}
.el-step__title {
  font-size: 12px;
}
.ql-editor {
  height: 400px;
}
</style>
