<template>
  <div class="transictionWrap">
     <Airdrop />
  </div>
</template>

<script>

// import Selection from './selection.vue'
import Airdrop from './airdrop.vue'
export default {
name:"Transiction",
  components: {
    Airdrop,
  },
  data() {
    return {
      
    }
  },
 
}
</script>

<style lang='scss' scoped>
.transictionWrap{
    padding: 0 1.928571rem /* 27/14 */;
    
}
</style>