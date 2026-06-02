<template>
  <div v-if="isOpen"></div>
</template>

<script>
export default {
  name: 'PurchaseOrderModal',
  props: {
    isOpen: {
      type: Boolean,
      default: false
    },
    backlogItem: {
      type: Object,
      default: null
    },
    mode: {
      type: String,
      default: ''
    }
  },
  emits: ['close', 'po-created']
}
</script>
