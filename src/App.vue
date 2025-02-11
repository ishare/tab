<script setup>
import { computed, onMounted, reactive, useTemplateRef, watch } from "vue"
import { VList } from "virtua/vue"
import { convertToPinyin } from "tiny-pinyin"

const data = reactive({
    keyword: '',
    curr: {},
    list: computed(() => {
        if (data.keyword) {
            return data.raw.filter(({ t, p, py }) =>
                t.includes(data.keyword)
                || p.startsWith(data.keyword.toLowerCase())
                || py.startsWith(data.keyword.toLowerCase())
            )
        }
        return data.raw
    }),
    raw: [],
    full: false
})


const init = async _ => {
    const resp = await fetch('./data/dict.json')
    const list = await resp.json()
    data.raw = list.map(({ i, t, c }) => {
        let pys = convertToPinyin(t, ' ', true).split(' ')
        return { i, t, c, p: pys.map(p => p[0]).join(''), py: pys.join('') }
    })
}

const tablature = useTemplateRef('right')

watch(() => data.curr, curr => {
    tablature.value.scrollTo({
        top: 0,
        behavior: 'smooth'
    })
})

onMounted(async () => {
    await init()
})

</script>

<template>
    <div class="main"
         :class="{ full: data.full }">
        <div class="left ani">
            <input class="search"
                   v-model="data.keyword"
                   placeholder="搜索歌曲(支持拼音搜索)" />
            <v-list v-if="data.list.length" :data="data.list"
                    :style="{ height: 'calc(100% - 40px)', width: '100%' }"
                    #default="{ item, index }">
                <div :key="item.i"
                     class="song ani"
                     :class="{ song_odd: index % 2, song_curr: item == data.curr }"
                     @click="data.curr = item">
                    {{ item.t }}
                </div>
            </v-list>
            <div v-else-if="data.raw.length">
                没有搜索结果
            </div>
            <div v-else>
                正在加载...
            </div>
        </div>
        <div class="right"
             ref="right">
            <div v-if="data.curr.i"
                 @dblclick="data.full = !data.full">
                <h1 class="title">{{ data.curr.t }}</h1>
                <p v-if="data.full">双击恢复列表</p>
                <p v-else>双击切换全屏</p>
                <hr>
                <template v-if="data.curr.c > 0">
                    <img v-for="i in data.curr.c"
                         :src="`./data/pics/${data.curr.i}/${i - 1}.webp`" />
                </template>
            </div>
            <p v-else>所有内容均来自互联网整理分享</p>
        </div>
    </div>
</template>

<style>
html,
body,
#app {
    width: 100%;
    height: 100%;
    overflow: hidden;
    padding: 0;
    margin: 0;
}

.ani {
    transition: all .3s ease;
}

.main {
    display: flex;
    width: 100%;
    height: 100%;
}

.left {
    height: 100%;
    width: 240px;
    overflow: hidden;
}

.right {
    height: 100%;
    box-shadow: -1px 0 0 rgba(0, 0, 0, .05);
    flex: 1;
    overflow-y: auto;
    text-align: center;
    user-select: none;
}

.search {
    height: 40px;
    width: 100%;
    padding: 0;
    margin: 0;
    border: 0;
    font-size: 1rem;
    text-indent: .4rem;
    color: #06C;
    outline: none;
    box-shadow: 0 1px 0 rgba(0, 0, 0, .05);
}

.song {
    height: 40px;
    line-height: 40px;
    width: 100%;
    text-indent: .4rem;
    cursor: pointer;
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
}

.song_odd {
    background: rgba(0, 0, 0, .05);
}

.song_curr {
    color: #F30;
}

.song:hover {
    background: rgba(0, 0, 0, .1);
}

.title {
    font-weight: 200;
    text-align: center;
}

.full img {
    width: 100%;
}

.full .left {
    width: 0;
}

@media screen and (orientation:portrait){
    .main{
        flex-direction: column;
    }
    .left{
        width: 100%;
        height: 160px;
        box-shadow: 0 1px 0 rgba(0, 0, 0, .05);
    }
    .full .left{
        width: 100%;
        height: 0;
    }
}
</style>
