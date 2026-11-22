<template>
  <div>
    <button @click="getList">刷新数据</button>
    <button @click="destroyOb">关闭监听</button>
    <button @click="createOb">开启监听器</button>
    <div class="list" ref="scrollToLoadMoreContent">
      <div class="item" v-for="item in tableList" :key="item.title">
        {{ item.title }}
      </div>
      <div class="load-more-btn" @click="loadMore" ref="loadMoreBtn">
        {{ !tableList.length ? '' : tableList.length >= total ? '没有更多了' : isLoading ? '加载中...' : '加载更多' }}
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted, onUnmounted } from 'vue';

const allMockData = new Array(97).fill(null).map((item, index) => ({
  title: `课程${index + 1}`,
  desc: `课程${index + 1}的描述`,
}));
const getMockData = ({pageIndex = 1, pageSize = 20} = {}) => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve({
        list: allMockData.slice((pageIndex - 1) * pageSize, pageIndex * pageSize),
        totalCount: allMockData.length,
      });
    }, 1000);
  });
}

const scrollToLoadMoreContent = ref<HTMLDivElement | null>(null);

const tableList = ref([]);
const total = ref(0);
const pageIndex = ref(1);
const pageSize = ref(20);

const isLoading = ref(false);
const getList = async () => {
  isLoading.value = true;
  const listElement = scrollToLoadMoreContent.value;
  if (listElement) listElement.scrollTop = 0;
  pageIndex.value = 1;
  tableList.value = [];
  const {list, totalCount} = await getMockData({pageIndex: pageIndex.value, pageSize: pageSize.value});
  tableList.value = list;
  total.value = totalCount;
  isLoading.value = false;
  if (tableList.value.length >= total.value) {
    destroyOb();
  } else {
    createOb();
  }
}
const loadMore = async () => {
  if (tableList.value.length >= total.value || tableList.value.length === 0) {
    return;
  }
  if (isLoading.value) {
    return;
  }
  isLoading.value = true;
  pageIndex.value++;
  const {list, totalCount} = await getMockData({pageIndex: pageIndex.value, pageSize: pageSize.value});
  tableList.value.push(...list);
  total.value = totalCount;
  isLoading.value = false;
  if (tableList.value.length >= total.value) {
    destroyOb();
  }
}

const loadMoreBtn = ref<HTMLDivElement | null>(null);
let ob: IntersectionObserver | null = null;
const destroyOb = () => {
  if (ob) {
    ob.disconnect();
  }
}
const createOb = () => {
  destroyOb();

  const root = scrollToLoadMoreContent.value;
  const target = loadMoreBtn.value;

  if (!root || !target) {
    return;
  }

  ob = new IntersectionObserver((entries) => {
    console.log(entries);
    if (entries[0].isIntersecting) {
      loadMore();
    }
  }, {
    root,
    threshold: 0,
    rootMargin: '0px 0px 120px 0px',
  });

  ob.observe(target);
};

onMounted(() => {
  getList();
});

onUnmounted(() => {
  destroyOb();
});
</script>

<style scoped>
.list {
  height: 500px;
  overflow: auto;
  border: 1px solid #ccc;
}
.item {
  padding: 10px;
  background-color: #f5f5f5;
  margin: 10px 0;
}
.load-more-btn {
  color: #292626;
  padding-bottom: 10px;
  text-align: center;
}
</style>
