<template>
  <Card :class="'ivu-card-' + theme" :padding="0" dis-hover>
    <ul>
      <slot></slot>
    </ul>
  </Card>
</template>

<script>
  export default {
    name: 'VerticalMenu'
  }
</script>

<style scoped lang="less">
</style>
