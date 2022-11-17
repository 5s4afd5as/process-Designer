<script lang="ts">
import { defineComponent, onMounted, reactive, nextTick, ref } from 'vue';
import { Graph, Shape, Node, Addon, Cell, DataUri } from '@antv/x6';
import insertCss from 'insert-css'   //用来给html的节点添加样式  这个页面暂时用不到
import type { TabsPaneContext } from 'element-plus'
import type { FormInstance } from 'element-plus'

let graph = {};

//画布的基础配置
const initGraph = () => {
  graph = new Graph({
    container: document.getElementById('container') as HTMLElement,
    width: 870,
    height: 540,
    background: {//画布颜色
      color: '#f4f4f4'
    },
    grid: {//网格
      size: 15,
      visible: true,
      type: 'mesh', // 'dot' | 'fixedDot' | 'mesh'
      args: {
        color: '#a0a0a0', // 网格线/点颜色
        thickness: 1,     // 网格线宽度/网格点大小
      },
    },
    panning: { //移动画布
      enabled: true,
      modifiers: 'shift',
    },
    mousewheel: {//画布的缩放
      enabled: true,
      zoomAtMousePosition: true,
      modifiers: 'ctrl',
      minScale: 0.5,
      maxScale: 3,
    },
    connecting: {
      router: {
        name: 'manhattan',
        args: {
          padding: 1,
        },
      },
      connector: {
        name: 'rounded',
        args: {
          radius: 8,
        },
      },
      anchor: 'center',
      connectionPoint: 'anchor',
      allowBlank: false,
      snap: {
        radius: 20,
      },
      createEdge() {//边的颜色 宽度
        return new Shape.Edge({
          attrs: {
            line: {
              stroke: '#A2B1C3',
              strokeWidth: 2,
              targetMarker: {
                name: 'block',
                width: 12,
                height: 8,
              },
            },
          },
          zIndex: 0,
        })
      },
      validateConnection({ targetMagnet }) {
        return !!targetMagnet
      },
    },
    highlighting: {
      magnetAdsorbed: {
        name: 'stroke',
        args: {
          attrs: {
            fill: '#5F95FF',
            stroke: '#5F95FF',
          },
        },
      },
    },
    resizing: true, //改变尺寸
    rotating: true,//旋转
    selecting: {//选中
      enabled: true,
      rubberband: true,
      showNodeSelectionBox: true,
    },
    snapline: true,
    keyboard: true,
    clipboard: true,
  })
}
//图形的样式  及其  连接柱的样式
const ports = {
  groups: {
    top: {
      position: 'top',
      attrs: {
        circle: {
          r: 4,
          magnet: true,
          stroke: '#9f9fa0',
          strokeWidth: 1,
          fill: '#fafafa',
          style: {
            visibility: 'hidden',
          },
        },
      },
    },
    right: {
      position: 'right',
      attrs: {
        circle: {
          r: 4,
          magnet: true,
          stroke: '#9f9fa0',
          strokeWidth: 1,
          fill: '#fff',
          style: {
            visibility: 'hidden',
          },
        },
      },
    },
    bottom: {
      position: 'bottom',
      attrs: {
        circle: {
          r: 4,
          magnet: true,
          stroke: '#9f9fa0',
          strokeWidth: 1,
          fill: '#fff',
          style: {
            visibility: 'hidden',
          },
        },
      },
    },
    left: {
      position: 'left',
      attrs: {
        circle: {
          r: 4,
          magnet: true,
          stroke: '#9f9fa0',
          strokeWidth: 1,
          fill: '#fff',
          style: {
            visibility: 'hidden',
          },
        },
      },
    },
  },
  items: [
    {
      group: 'top',
    },
    {
      group: 'right',
    },
    {
      group: 'bottom',
    },
    {
      group: 'left',
    },
  ],
}
const initElment = () => {
  //定义图形 的样式
  Graph.registerNode(
    'rect',
    {
      inherit: 'rect',
      width: 66,
      height: 36,
      attrs: {
        body: {
          strokeWidth: 1,
          stroke: '#9f9fa0',
          fill: '#f5f5fe',
        },
        text: {
          fontSize: 12,
          fill: '#262626',
        },
      },
      ports: { ...ports },
    },
    true,
  )

  Graph.registerNode(
    'polygon',
    {
      inherit: 'polygon',
      width: 66,
      height: 36,
      attrs: {
        body: {
          strokeWidth: 1,
          stroke: '#9f9fa0',
          fill: '#f5f5fe',
        },
        text: {
          fontSize: 12,
          fill: '#262626',
        },
      },
      ports: {
        ...ports,
        items: [
          {
            group: 'top',
          },
          {
            group: 'bottom',
          },
        ],
      },
    },
    true,
  )

  Graph.registerNode(
    'circle',
    {
      inherit: 'circle',
      width: 45,
      height: 45,
      attrs: {
        body: {
          strokeWidth: 1,
          stroke: '#9f9fa0',
          fill: '#f5f5fe',
        },
        text: {
          fontSize: 12,
          fill: '#262626',
        },
      },
      ports: { ...ports },
    },
    true,
  )

  Graph.registerNode(
    'image',
    {
      inherit: 'rect',
      width: 52,
      height: 52,
      markup: [
        {
          tagName: 'rect',
          selector: 'body',
        },
        {
          tagName: 'image',
        },
        {
          tagName: 'text',
          selector: 'label',
        },
      ],
      attrs: {
        body: {
          stroke: '#9f9fa0',
          fill: '#f5f5fe',
        },
        image: {
          width: 26,
          height: 26,
          refX: 13,
          refY: 16,
        },
        label: {
          refX: 3,
          refY: 2,
          textAnchor: 'left',
          textVerticalAnchor: 'top',
          fontSize: 12,
          fill: '#fff',
        },
      },
      ports: { ...ports },
    },
    true,
  )
}
//定义事件  特别是让 连接柱显示和隐藏的功能
const initEvent = () => {
  // 控制连接桩显示/隐藏
  const showPorts = (ports: NodeListOf<SVGElement>, show: boolean) => {
    if (show) {
      for (let i = 0, len = ports.length; i < len; i = i + 1) {
        ports[i].style.visibility = 'hidden' ? 'visible' : 'hidden'
      }
    } else {
      for (let i = 0, len = ports.length; i < len; i = i + 1) {
        ports[i].style.visibility = 'visible' ? 'hidden' : 'visible'
      }
    }
  };
  (graph as Graph).on('node:mouseenter', () => {
    const container = document.querySelector('#container')
    nextTick(() => {
      const ports = (container as HTMLElement).querySelectorAll('.x6-port-body') as NodeListOf<SVGElement>
      // console.log(ports);
      showPorts(ports, true)
    })
  });
  (graph as Graph).on('node:mouseleave', () => {
    console.log();
    const container = document.querySelector('#container')
    nextTick(() => {
      const ports = (container as HTMLElement).querySelectorAll('.x6-port-body') as NodeListOf<SVGElement>
      // console.log(ports);
      showPorts(ports, false)
    })
  });
}


export default defineComponent({
  setup() {
    let curCel: Cell | null;
    const tabPosition = ref('middle')
    const activeName = ref('first')
    const inputIdData = ref(1)
    const ruleFormRef = ref<FormInstance>()
    const ruleForm = reactive({
      id: 1,
      text: '',
    })


    const handleClick = (tab: TabsPaneContext, event: Event) => {
      // console.log(tab, event)
    }
    onMounted(() => {
      //先画个图
      initGraph()
      //定义事件
      initEvent()
    })
    // 在此定义 右侧  点击时  创建 图形的事件  以及  保存成svg,png,json等事件
    //这里不再考虑复用性了了 直接复制 
    //这个是开始的节点
    const addStartNodeAndEdge = () => {
      // initElment()   
      const metadata: Node.Metadata = {
        x: 40,
        y: 40,
        width: 100,
        height: 40,
        label: 'rect',
        zIndex: 2,
        markup: [
          {
            tagName: 'rect',
            selector: 'body',
          },
          {
            tagName: 'image',
            selector: 'image'
          },
          {
            tagName: 'text',
            selector: 'label',
          },
        ],
        attrs: {
          body: {//成为椭圆
            rx: 20,
            ry: 26,
          },
          image: {
            'xlink:href':
              'https://img.ixintu.com/upload/jpg/20210523/c597a02f6548b4ba40159bb072e86521_24859_512_512.jpg!ys',
            width: 16,
            height: 16,
            x: 15,
            y: 12
          },
          rect: {
            stroke: '#9f9fa0', // 边框颜色
            fill: '#fafafa',   // 填充颜色
          },
          //------------------------------参考markup
          // 指定 text 元素的样式
          label: {
            text: '开始', // 文字
            fill: '#333', // 文字颜色
          },
        },
        ports: { ...ports },
      };


      (graph as Graph).addNode(metadata)

    }
    //这个是结束的节点
    const addEndNodeAndEdge = () => {
      const metadata: Node.Metadata = {
        x: 40,
        y: 40,
        width: 100,
        height: 40,
        label: 'rect',
        zIndex: 2,
        markup: [
          {
            tagName: 'rect',
            selector: 'body',
          },
          {
            tagName: 'image',
            selector: 'image'
          },
          {
            tagName: 'text',
            selector: 'label',
          },
        ],
        attrs: {
          body: {//成为椭圆
            rx: 20,
            ry: 26,
          },
          image: {
            'xlink:href':
              'https://img.ixintu.com/upload/jpg/20210523/e701eddcfba185cca1583f7ffad91294_21551_512_512.jpg!ys',
            width: 16,
            height: 16,
            x: 15,
            y: 12
          },
          rect: {
            stroke: '#9f9fa0', // 边框颜色
            fill: '#fafafa',   // 填充颜色
          },
          //------------------------------参考markup
          // 指定 text 元素的样式
          label: {
            text: '结束', // 文字
            fill: '#333', // 文字颜色
          },
        },
        ports: { ...ports },
      };


      (graph as Graph).addNode(metadata)
    }
    //这个是任务节点
    const addMissionAndEdge = () => {
      const metadata: Node.Metadata = {
        x: 40,
        y: 40,
        width: 100,
        height: 40,
        markup: [
          {
            tagName: 'rect',
            selector: 'body',
          },
          {
            tagName: 'image',
            selector: 'image'
          },
          {
            tagName: 'text',
            selector: 'label',
          },
        ],
        attrs: {
          image: {
            'xlink:href':
              'https://img-qn.51miz.com/Element/00/77/78/00/2d6d4704_E777800_26033bb1.png',
            width: 16,
            height: 16,
            x: 5,
            y: 12
          },
          rect: {
            stroke: '#9f9fa0',
            fill: '#fafafa',
          },
          label: {
            contenteditable: 'true',//开启可编辑
            text: '任务节点',
            fill: '#333',
          },
        },
        ports: { ...ports },
      };


      (graph as Graph).addNode(metadata)
    }
    //用户点击图像的时候  调用的回调函数   并对上面我们设置的数据进行赋值
    nextTick(() => {
      (graph as Graph).on('cell:click', ({ cell }) => {
        console.log(cell.getAttrs());  //这个就拿到了当前点击的图形
        //清除之前选中的其他节点的样式
        // curCel?.attr('body/stroke', null)
        //新的节点赋值
        curCel = cell
        //新的节点边框设置样式为红色
        // curCel?.attr('body/stroke', 'red')
        //将labelText进行赋值(labelText有可能在text/text中,也有可能在label/text中)
        let labelText = null;
        if (cell.getAttrs()?.text?.text) labelText = cell.getAttrs()?.text?.text
        if (cell.getAttrs()?.label?.text) labelText = cell.getAttrs()?.text?.text
        ruleForm.text = labelText as any;  //这里只更改text做一下演示就行了
      })
    })
    //修改文本节点的内容
    const nodeTextchange = () => {
      curCel?.attr('label/text', ruleForm.text)
    }


    //导出svg格式
    const toSvg = () => {
      (graph as Graph).toSVG((dataUri: string) => {
        // 下载
        DataUri.downloadDataUri(DataUri.svgToDataUrl(dataUri), 'chart.svg')
      })
    }
    //导入PNg格式
    const toPNg = () => {
      (graph as Graph).toPNG((dataUri: string) => {
        // 下载
        DataUri.downloadDataUri(dataUri, 'chart.png')
      })
    }
    //toJson   可以保存到本地  在此省略
    const toJsonData = () => {
      console.log((graph as Graph).toJSON());
    }

    return {
      tabPosition,
      activeName,
      inputIdData,
      ruleFormRef,
      ruleForm,
      handleClick,
      addStartNodeAndEdge,
      addEndNodeAndEdge,
      addMissionAndEdge,
      toSvg,
      toPNg,
      toJsonData,
      nodeTextchange

    }
  }
})

</script>
<template>
  <el-row :gutter="24" class="headerBox">
    <el-col :span="24">
      QvFan流程编辑器
    </el-col>
  </el-row>
  <el-row :gutter="24" class="contentBox">
    <el-col :span="5" class="leftBox">
      <div class="left">
        <el-tag type="" class="mx-1" effect="dark" color="#0175c8" size=" defaul"> 工具</el-tag>
        <div class="TabBox">
          <el-button type="primary" color="#0175c8">
            <el-icon>
              <Rank />
            </el-icon>
          </el-button>
          <el-button class="rBtn">
            <el-icon>
              <Share />
            </el-icon>
          </el-button>
        </div>
        <el-tag type="" class="mx-1" effect="dark" color="#0175c8" size=" defaul">基础节点</el-tag>
        <div class="middleBox">
          <button class="btn" @click="addStartNodeAndEdge"> <em id="c1"></em>开始</button>
          <button class="btn" @click="addEndNodeAndEdge"> <em id="c2"></em>结束</button>
          <button class="btn" @click="addMissionAndEdge"> <em id="c3"></em>任务节点</button>
          <!-- 以下字体图标省略    随便代替了-->
          <button class="btn"> <em id="c4"></em>会签开始</button>
          <button class="btn"> <em id="c5"></em>会签结束</button>
          <button class="btn   btnTwo"></button>
        </div>
        <el-tag type="" class="mx-1" effect="dark" color="#0175c8" size=" defaul" id="three">冰道节点</el-tag>
        <div class="lastBox">
          <button class="btn"> <em id="c4"></em>横向泳道</button>
          <button class="btn"> <em id="c5"></em>纵向泳道</button>
        </div>
        <p id="tip">使用vue3+ts+antv x6 +element plus等<br />节省时间:样式稍微布局了一下 字体图标
          自己挑了几个比较像的<br />没有做画布的自适应
          页面缩放问题<br />支持导出Svg和PNg的格式,还可以转化成json数据保存到本地localstorage 或 导出<br />可以进行各种图形的修改颜色 文字 可编辑 以及
          图形之间的线条格式和样式的修改<br />(shift+鼠标右键可以移动画布,ctrl+滑轮可以进行缩放,鼠标右键拉取全选可调大小或旋转,只有任务节点添加了文本修改内容->但是文本可编辑可选中修改图形大小只能存在一个)
        </p>
      </div>
    </el-col>
    <el-col :span="14" class="middleBox">
      <div id="container"></div>
    </el-col>
    <el-col :span="5" class="rightBox">
      <el-tabs v-model="activeName" class="demo-tabs">
        <el-tab-pane label="流程属性" name="first">
          <p class="title">流程id</p>
          <!-- <input type="text" class="tab-input" v-model="inputIdData"> -->
          <el-form ref="ruleFormRef" :model="ruleForm" status-icon label-width="0px" class="demo-ruleForm">
            <el-form-item prop="id">
              <el-input v-model="ruleForm.id" />
            </el-form-item>
          </el-form>
          <p class="tip">只做了节点属性的修改 节点属性主要是修改 左边任务节点的图形里面的文本的</p>
        </el-tab-pane>
        <el-tab-pane label="节点属性" name="second">
          <p class="title">文本属性</p>
          <el-form ref="ruleFormRef" :model="ruleForm" status-icon label-width="0px" class="demo-ruleForm">
            <el-form-item prop="id">
              <el-input v-model="ruleForm.text" @change="nodeTextchange" />
            </el-form-item>
          </el-form>
          <p class="tip">只做了节点属性的修改 节点属性主要是修改 左边任务节点的图形里面的文本的</p>
        </el-tab-pane>
        <el-tab-pane label="连线属性" name="third">
          连线属性
        </el-tab-pane>
      </el-tabs>
    </el-col>
  </el-row>
  <el-row :gutter="24" class="buttomBox">
    <el-col :span="24">
      <div class="buttomDiv">
        <el-button type="primary">上一步</el-button>
        <el-button type="success" @click="toSvg">保存成svg</el-button>
        <el-button type="info" @click="toPNg">保存成Png</el-button>
        <el-button type="info" @click="toJsonData">保存数据</el-button>
      </div>
    </el-col>
  </el-row>


</template>
<style lang="less" scoped>
@font-face {
  font-family: 'icomoon';
  src: url('@/assets/fonts/icomoon.eot?8kkdz6');
  src: url('@/assets/fonts/icomoon.eot?8kkdz6#iefix') format('embedded-opentype'),
    url('@/assets/fonts/icomoon.ttf?8kkdz6') format('truetype'),
    url('@/assets/fonts/icomoon.woff?8kkdz6') format('woff'),
    url('@/assets/fonts/icomoon.svg?8kkdz6#icomoon') format('svg');
  font-weight: normal;
  font-style: normal;
  font-display: block;
}

body em {
  font-family: 'icomoon' !important;
  font-style: normal;
  font-size: 10px;
}

.el-row {
  // margin-bottom: 20px;
}

.el-row:last-child {
  margin-bottom: 0;
}

.el-col {
  border-radius: 4px;
}

.grid-content {
  border-radius: 4px;
  min-height: 36px;
}

.headerBox {
  height: 60px;
  font-size: 25px;
  text-align: center;
  line-height: 40px;
  border-bottom: 2px solid gray;
}

.contentBox {
  height: 550px;
  border-bottom: 2px solid gray;

  .leftBox {
    border-right: 2px solid gray;

    .left {
      height: 100%;
      border-left: 2px solid gray;
      padding-left: 2px;

      .TabBox {
        width: 100%;
        height: 40px;
        text-align: center;
        padding-top: 15px;

        .rBtn {
          margin: 0;
          // padding: 0;
        }
      }

      .middleBox,
      .lastBox {
        display: flex;
        flex-direction: row;
        flex-wrap: wrap;
        justify-content: space-around;

        .btn {
          width: 40%;
          height: 25px;
          border: 0;
          margin-top: 5px;
          font-size: 12px;
          text-align: left;
          font-weight: 500;
          cursor: pointer;
        }


        .btnTwo {
          visibility: hidden;
        }
      }

      #three {
        margin-top: 7px;
      }

      #tip {
        margin-top: 20px;
        color: red;
        font-size: 10px;
      }

    }
  }

  .rightBox {
    border-left: 2px solid gray;
  }
}

.buttomBox {
  width: 100vw;
  height: 50px;

  .buttomDiv {
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
  }

}

.el-tabs__header {
  margin: 0;
}

.demo-tabs>.el-tabs__content {}

.title {
  margin: 0;
  padding: 0;
}

.tip {
  font-size: 10px;
  color: red;
}

.el-form {
  margin-top: 5px !important;
}

.el-input__inner {
  width: 90%;
  border: 1px solid #eeedf2;
  background-color: #f6f7fb !important;
}
</style>