<template>
  <span class="stepbutton" :class="{active: value, highlight: highlight}" @click="changeValue">
    
  </span>
</template>

<script>
export default {
  name: 'stepbutton',
  props: ['value','highlight'],
  methods:{
    changeValue(){
      this.$emit('input', !this.value)
    }
  }
}
</script>

<style scoped>
  .stepbutton{
    display: inline-block;
    width: 32px;
    height: 32px;
    border-radius: 7px;
    border: 0px solid rgba(0,0,0,0.5);
    border-width: 0px 1px 4px 1px;
    box-shadow: 0px 5px 1px 1px rgba(0,0,0,0.3), inset 0px 0px 0px 3px rgba(0,0,0,0.1);
    background-image: linear-gradient(to top, #3f51b1 0%, #5a55ae 13%, #7b5fac 25%, #8f6aae 38%, #a86aa4 50%, #cc6b8e 62%, #f18271 75%, #f3a469 87%, #f7c978 100%);
    cursor: pointer;
    margin: 16px 3px;
    transition: all 0.1s ease-in-out;
  }
  .stepbutton.active{
    border-bottom-width: 1px;
    box-shadow: 0px 1px 0px 1px rgba(0,0,0,0.3), inset 0px 0px 0px 3px rgba(0,0,0,0.1);;
    background-image: linear-gradient(to top, #3f51b1 0%, #5a55ae 13%, #7b5fac 25%, #8f6aae 38%, #a86aa4 50%, #cc6b8e 62%, #f18271 75%, #f3a469 87%, #f7c978 100%);

}



</style>

