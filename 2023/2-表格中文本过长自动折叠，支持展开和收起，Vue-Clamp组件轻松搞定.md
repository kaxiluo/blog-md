当表格中内容过长、占用多行，十分影响用户体验。如何实现文本自动截断、展开、收起呢？vue-clamp可以轻松实现。

## 安装 vue-clamp

```bash
 npm i --save vue-clamp
```

## 封装组件，方便在 Element-UI Table 中使用

```html
<template>
    <el-table-column :prop="prop" :label="label">
        <template slot-scope="scope">
            <v-clamp autoresize :max-lines="maxLines">{{scope.row[prop]}}
                <template #after="{ toggle, expanded, clamped }">
                    <el-button plain v-if="expanded || clamped"
                               @click="toggle"
                               style="margin-left: 5px"
                               size="small">{{ expanded ? '隐藏' : '展开' }}</el-button>
                </template>
            </v-clamp>
        </template>
    </el-table-column>
</template>

<script>
import VClamp from 'vue-clamp'

export default {
    components: {
        VClamp
    },
    name: "table-column-v-clamp",
    props: {
        prop: String,
        label: String,
        maxLines: { type: Number, default: 2 }
    }
}
</script>
```

## 轻松使用，一行搞定

```html
<el-table 
    :data="dataList" 
    border 
    style="width: 100%">
    <el-table-column prop="id" label="ID"></el-table-column>
    <el-table-column prop="name" label="名字"></el-table-column>
    <table-column-v-clamp prop="content" label="内容"></table-column-v-clamp>
</el-table>
```

## 关于 vue-clamp

可以选择限制行数与/或最大高度，无需指定行高。

支持在布局变化时自动更新。

支持展开/收起被截断部分内容。

支持自定义截断文本前后内容，并且进行响应式更新。

支持在文本末尾、中间或开始位置进行截断

[**vue-clamp官网体验**](https://vue-clamp.vercel.app/?lang=zh)
