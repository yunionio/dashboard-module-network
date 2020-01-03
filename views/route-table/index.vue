<template>
  <div>
    <page-header title="路由表" />
    <page-body>
      <route-list :id="listId" />
    </page-body>
  </div>
</template>

<script>
import RouteList from './components/List'

export default {
  name: 'NetworkIndex',
  components: {
    RouteList,
  },
  data () {
    return {
      listId: 'RouteList',
    }
  },
}
</script>
