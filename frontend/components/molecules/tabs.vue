<template>
  <article>
    <header class="tabs">
      <ul>
        <li v-for="(tab, index) in tabs" :key="index">
          <div
            class="nav-item"
            :class="{ 'is-active': tab.isActive }"
            @click="selectTab(tab)"
          >
            {{ tab.name }}
          </div>
        </li>
      </ul>
    </header>
    <section class="tabs-details">
      <slot></slot>
    </section>
  </article>
</template>

<script>
export default {
  data: () => {
    return {
      tabs: [],
    };
  },
  created() {
    this.tabs = this.$children;
  },
  methods: {
    selectTab(selectedTab) {
      this.tabs.forEach((tab) => {
        tab.isActive = tab.name === selectedTab.name;
      });
    },
  },
};
</script>

<style lang="scss" scoped>
.tabs {
  border-bottom: 1px solid #fefefe;
  margin: 0 10px;
}

.tabs-details {
  padding: 10px;
}

ul {
  display: flex;
  padding: 0;
  list-style: none;

  li {
    margin-right: 40px;
  }

  .nav-item {
    cursor: pointer;

    &:hover {
      color: #fb130a;
    }

    &.is-active {
      color: #fb130a;
    }
  }
}
</style>
