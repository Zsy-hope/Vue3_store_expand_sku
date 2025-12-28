<template>
  <div class="goods-sku">
    <dl v-for="item in goods.specs" :key="item.id">
      <dt>{{ item.name }}</dt>
      <dd>
        <template v-for="val in item.values" :key="val.name">
          <!-- 图片类型规格 -->
          <img
            :class="{ selected: val.selected, disabled: val.disabled }"
            @click="changeSelectedStatus(item, val)"
            v-if="val.picture"
            :src="val.picture"
            :title="val.name"
          />
          <!-- 文字类型规格 -->
          <span :class="{ selected: val.selected, disabled: val.disabled }" @click="changeSelectedStatus(item, val)" v-else>{{ val.name }}</span>
        </template>
      </dd>
    </dl>
  </div>
</template>

<script setup>
  import { onMounted, ref } from 'vue'
  import axios from 'axios'
  import subsetAlgorithm from './subset-algorithm'

  // 商品数据
  const goods = ref({})
  let pathMap = {}
  const getGoods = async () => {
    // 1135076  初始化就有无库存的规格
    // 1369155859933827074 更新之后有无库存项（蓝色-20cm-中国）
    const res = await axios.get('http://pcapi-xiaotuxian-front-devtest.itheima.net/goods?id=1369155859933827074')
    goods.value = res.data.result
    console.log(goods.value)
    pathMap = getPathMap(goods.value)
    console.log(pathMap)
    //初始化禁用状态
    initDisabledStatus(goods.value.specs, pathMap)
  }
  onMounted(() => getGoods())

  //切换选中状态
  const changeSelectedStatus = (item, val) => {
    //item：同一排对象
    //val：当前点击项

    //库存不足时不可选中
    if (val.disabled) return

    if (val.selected) {
      val.selected = false
    } else {
      //第一次为所有对象添加selected属性
      //遍历同一排其他对象
      item.values.forEach((element) => {
        element.selected = false
      })
      val.selected = true
    }

    //更新禁用状态
    updateDisabledState(goods.value.specs, pathMap)
  }

  //生成有效路径字典对象
  const getPathMap = (goods) => {
    //1.根据skus字段生成有效的sku数组   获取所有库存大于0的组合
    const effectiveSkus = goods.skus.filter((sku) => sku.inventory > 0)
    // console.log(effectiveSkus)

    //2.根据有效的sku使用算法(子集算法) 获取子集算法
    //[1,2] => [[1],[2],[1,2]]
    effectiveSkus.forEach((sku) => {
      //2.1获取匹配的valueName组成的数组 生成包含各个组合的大数组
      const seletedValArr = sku.specs.map((val) => val.valueName)
      // console.log(seletedValArr)
      //2.2使用子集算法拆解大数组
      const valueArrSubSet = subsetAlgorithm(seletedValArr)
      // console.log(valueArrSubSet)
      //3.生成最终的路径字典对象
      valueArrSubSet.forEach((arr) => {
        //生成键值对key
        //join   ['黑色','中国'] => '黑色-中国'
        const key = arr.join('-')
        if (pathMap[key]) {
          pathMap[key].push(sku.id)
        } else {
          pathMap[key] = [sku.id]
        }
      })
    })
    return pathMap
  }

  //初始化禁用状态
  const initDisabledStatus = (specs, pathMap) => {
    specs.forEach((specs) => {
      specs.values.forEach((val) => {
        if (pathMap[val.name]) {
          val.disabled = false
        } else {
          val.disabled = true
        }
      })
    })
  }

  //获取选中项的匹配数组
  const getSelectedValues = (specs) => {
    const arr = []
    specs.forEach((spec) => {
      //找到values中selected为true的项，添加到数组中对应的位置
      const selectedVal = spec.values.find((item) => item.selected)
      arr.push(selectedVal ? selectedVal.name : undefined)
    })
    return arr
  }

  //切换时更新禁用状态
  const updateDisabledState = (specs, pathMap) => {
    // 约定：每一个按钮的状态由自身的disabled进行控制
    specs.forEach((spec, index) => {
      const selectedValues = getSelectedValues(specs)
      console.log(spec)
      //每次点完一个 就进行判断
      spec.values.forEach((val) => {
        selectedValues[index] = val.name
        const key = selectedValues.filter((value) => value).join('-')
        if (pathMap[key]) {
          val.disabled = false
        } else {
          val.disabled = true
        }
      })
    })
  }
</script>

<style scoped lang="scss">
  @mixin sku-state-mixin {
    border: 1px solid #e4e4e4;
    margin-right: 10px;
    cursor: pointer;

    &.selected {
      border-color: #27ba9b;
    }

    &.disabled {
      opacity: 0.6;
      border-style: dashed;
      cursor: not-allowed;
    }
  }

  .goods-sku {
    padding-left: 10px;
    padding-top: 20px;

    dl {
      display: flex;
      padding-bottom: 20px;
      align-items: center;

      dt {
        width: 50px;
        color: #999;
      }

      dd {
        flex: 1;
        color: #666;

        > img {
          width: 50px;
          height: 50px;
          margin-bottom: 4px;
          @include sku-state-mixin;
        }

        > span {
          display: inline-block;
          height: 30px;
          line-height: 28px;
          padding: 0 20px;
          margin-bottom: 4px;
          @include sku-state-mixin;
        }
      }
    }
  }
</style>
