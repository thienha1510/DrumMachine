<template>
  <span class="led" :class="{lit:active}">
  </span>
</template>

<script>
export default {
  name: 'led',
  props: ['active']
}
</script>

<style scoped>
  .led{
    display: inline-block;
    width: 34px;
    height: 20px;
    text-align: center;
    margin: 2px 3px;
    transition: all 0.2s ease-in-out;
  }

  .led:before{
    content: '';
    display: inline-block;
    width: 10px;
    height: 10px;
    border-radius: 10px;
    background-color: #bbb;
    box-shadow: inset 0px 0px 1px 2px rgba(0,0,0,0.1);
  }

  .led.lit:before{
    background-image: linear-gradient(to top, #ff0844 0%, #ffb199 100%);
    box-shadow: 0px 0px 7px 0px #ffb199;
  }
</style>

