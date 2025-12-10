<template>
  <div class="app-container">
    <el-row :gutter="0">
      <el-col :span="24">
        <el-card class="box full-width">
          <div slot="header" class="clearfix">
            <span>项目信息</span>
          </div>
          <div style="text-align: center; margin-top: 10px; margin-bottom: 20px">
            <img src="@/assets/image.png" style="width: 360px; height: auto" />
          </div>

          <div style="line-height: 32px">
            <div class="title" style="padding-top: 0px">系统介绍</div>
            USTC考试管理系统基于SpringBoot+Vue开发，包含在线考试、用户体系、错题训练、考试规则、智能算分等考试功能。

            <div class="title">技术栈</div>
            <div class="tags">
              <el-tag>SpringBoot</el-tag>
              <el-tag>Shiro</el-tag>
              <el-tag>MyBatis Plus</el-tag>
              <el-tag>JWT</el-tag>
              <el-tag>Vue</el-tag>
              <el-tag>ElementUI</el-tag>
            </div>

            <div class="title">产品功能</div>
            <div>
              <div>系统完善：完善的前后端分离架构、规范的接口文档</div>
              <div>权限控制：基于Shiro和JWT开发的权限控制功能。</div>
              <div>基础功能：系统配置、用户管理、部门管理、角色管理等。</div>
              <div>题库管理：支持单选题、多选题、判断题、支持试题批量导入导出</div>
              <div>考试权限：支持指定考试范围：完全公开、指定某些部门人员考试</div>
              <div>在线考试：学员在线考试、查看分数、考试错题训练。</div>
            </div>
          </div>
          <div class="title">团队信息</div>
          <div>
            <el-row>
              <el-col :span="12">刘畅 SA25218112</el-col>
              <el-col :span="12">林晖祖 SA25218110</el-col>
              <el-col :span="12">高宇轩 SA25218104</el-col>
              <el-col :span="12">刘雨桐 SA25218167</el-col>
              <el-col :span="12">曾经纬 SA25218130</el-col>
              <el-col :span="12">张远卓 SA25218128</el-col>
              <el-col :span="12">黄胤臻 SA25218106</el-col>
              <el-col :span="12">刘擎 SA25218165</el-col>
            </el-row>
          </div>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import { mapGetters } from "vuex";
import config from "../../../package.json";

export default {
  name: "Dashboard",
  data() {
    return {
      currentRole: "adminDashboard",
      version: config.version,
    };
  },
  computed: {
    ...mapGetters(["roles", "siteData"]),
  },
  created() {},
};
</script>

<style scoped>
.box {
  min-height: calc(100vh - 120px);
  font-size: 14px;
  line-height: 28px;
}

.box.full-width {
  width: 100%;
}
.title {
  font-size: 18px;
  font-weight: 700;
  padding-bottom: 10px;
  padding-top: 20px;
  margin-bottom: 10px;
  border-bottom: #eee 1px dotted;
}

.tags span {
  margin-right: 10px;
}

.box a {
  color: #20a0ff;
}

.box a:hover {
  color: #ff0000;
}
</style>
