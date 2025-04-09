<template>
  <div class="canvas-container">
    <div class="business-model-canvas" ref="canvas"></div>
    
    <!-- 缩放控制按钮 -->
    <div class="zoom-controls">
      <button @click="zoomIn" class="zoom-button">+</button>
      <button @click="zoomOut" class="zoom-button">-</button>
      <button @click="resetZoom" class="zoom-button">↺</button>
    </div>
    
    <!-- 输入弹窗 -->
    <div v-if="showDialog" class="dialog-overlay">
      <div class="dialog-content">
        <h3>{{ dialogType === 'add' ? '添加便利贴' : '确认删除' }}</h3>
        <template v-if="dialogType === 'add'">
          <textarea ref="noteInput" v-model="newNoteContent" placeholder="输入内容..." />
        </template>
        <template v-else>
          <p style="color: #666; margin: 15px 0;">确认要删除这个便利贴吗？</p>
        </template>
        <div class="dialog-buttons">
          <button @click="handleConfirm">确定</button>
          <button @click="handleCancel">取消</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as d3 from 'd3';

const contentWidth = 1040;
const contentHeight = 519;
export default {
  name: 'BusinessModelCanvas',
  data() {
    return {
      // 拖拽状态存储
      dragging: null,
      // 动态计算画布尺寸（根据模块布局自动调整）
      width: 0,
      height: 0,
      // 弹窗相关数据
      showDialog: false,
      selectedBox: null,
      selectedNote: null,
      dialogType: 'add', // 'add' or 'delete'
      newNoteContent: '',
      // 存储所有便利贴 {boxId, content, x, y}
      notes: [],
      // 缩放相关
      zoom: null,
      currentZoom: 1,
      /**
       * 定义每个模块（盒子）的布局信息:
       *  - id：模块的唯一标识
       *  - title：模块显示的标题
       *  - x, y：左上角坐标（相对于SVG）
       *  - width, height：模块的宽度和高度
       *  - color：填充颜色（可根据需要自定义）
       */
      boxes: [
        // ------------------- 第一行（5个模块） -------------------
        {
          id: 'keyPartnerships',
          title: '重要伙伴',
          subtitle: 'Key Partnerships', 
          x: 0,
          y: 0,
          width: contentWidth / 5, // 固定宽度contentWidth的1/5
          height: contentHeight / 3 * 2,
          color: '#ffffff'
        },
        {
          id: 'keyActivities',
          title: '关键业务',
          subtitle: 'Key Activities',
          x: contentWidth / 5, // 固定宽度contentWidth的1/5
          y: 0,
          width: contentWidth / 5,
          height: contentHeight / 3,
          color: '#ffffff'
        },
        {
          id: 'valuePropositions',
          title: '价值主张',
          subtitle: 'Value Propositions',
          x: (contentWidth / 5) * 2, // 固定宽度累加
          y: 0,
          width: contentWidth / 5,
          height: contentHeight / 3 * 2,
          color: '#ffffff'
        },
        {
          id: 'customerRelationships',
          title: '客户关系',
          subtitle: 'Customer Relationships',
          x: (contentWidth / 5) * 3,
          y: 0,
          width: contentWidth / 5,
          height: contentHeight / 3,
          color: '#ffffff'
        },
        {
          id: 'customerSegments',
          title: '客户细分',
          subtitle: 'Customer Segments',
          x: (contentWidth / 5) * 4,
          y: 0,
          width: contentWidth / 5,
          height: contentHeight / 3 * 2,
          color: '#ffffff'
        },
        // ------------------- 第二行（4个模块） -------------------
        {
          id: 'keyResources',
          title: '核心资源',
          subtitle: 'Key Resources',
          x: contentWidth / 5,
          y: contentHeight / 3,
          width: contentWidth / 5,
          height: contentHeight / 3,
          color: '#ffffff'
        },
        {
          id: 'channels',
          title: '渠道通路',
          subtitle: 'Channels',
          x: (contentWidth / 5) * 3,
          y: contentHeight / 3,
          width: contentWidth / 5,
          height: contentHeight / 3,
          color: '#ffffff'
        },
        // ------------------- 第三行（2个模块） -------------------
        {
          id: 'costStructure',
          title: '成本结构',
          subtitle: 'Cost Structure',
          x: 0,
          y: contentHeight / 3 * 2,
          width: contentWidth / 2,
          height: contentHeight / 3,
          color: '#ffffff'
        },
        {
          id: 'revenueStreams',
          title: '收入来源',
          subtitle: 'Revenue Streams',
          x: contentWidth / 2,
          y: contentHeight / 3 * 2,
          width: contentWidth / 2,
          height: contentHeight / 3,
          color: '#ffffff'
        }
      ]
    };
  },
  mounted() {
    this.drawCanvas();
    window.addEventListener('resize', this.handleResize);
  },
  beforeUnmount() {
    window.removeEventListener('resize', this.handleResize);
  },
  methods: {
    handleResize() {
      this.drawCanvas();
    },

    getBoxIcon(boxId) {
      const icons = {
        keyPartnerships: 'M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm-4 6h-4v2h4v2h-4v2h4v2H9V7h6v2z',
        keyActivities: 'M13 9V5h-2v6H9.83L12 13.17 14.17 11H13z', 
        valuePropositions: 'M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-6h2v6zm0-8h-2V7h2v2z',
        customerRelationships: 'M16 11c1.66 0 2.99-1.34 2.99-3S17.66 5 16 5s-3 1.34-3 3 1.34 3 3 3zm-8 0c1.66 0 2.99-1.34 2.99-3S9.66 5 8 5 5 6.34 5 8s1.34 3 3 3zm0 2c-2.33 0-7 1.17-7 3.5V19h14v-2.5c0-2.33-4.67-3.5-7-3.5zm8 0c-.29 0-.62.02-.97.05 1.16.84 1.97 1.97 1.97 3.45V19h6v-2.5c0-2.33-4.67-3.5-7-3.5z',
        customerSegments: 'M12 5.5c1.38 0 2.5 1.12 2.5 2.5s-1.12 2.5-2.5 2.5-2.5-1.12-2.5-2.5 1.12-2.5 2.5-2.5zM18 17v-2.5c0-1.1-.9-2-2-2s-2 .9-2 2V17h4zM8 15c-1.1 0-2 .9-2 2v2.5h4V17c0-1.1-.9-2-2-2zm-4-2v4h4v-4H4zm14 4h4v-4h-4v4z',
        keyResources: 'M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-6h2v6zm0-8h-2V7h2v2z',
        channels: 'M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zM8 12V7h2v5H8zm6 6h-4v-2h4v2zm0-4h-4v-2h4v2zm0-4h-4v-2h4v2z',
        costStructure: 'M11 17h2v-6h-2v6zm1-15C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 18c-4.41 0-8-3.59-8-8s3.59-8 8-8 8 3.59 8 8-3.59 8-8 8zM11 9h2V7h-2v2z',
        revenueStreams: 'M11 17h2v-6h-2v6zm1-15C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 18c-4.41 0-8-3.59-8-8s3.59-8 8-8 8 3.59 8 8-3.59 8-8 8zM11 9h2V7h-2v2z'
      };
      return icons[boxId] || 'M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 15h-2v-6h2v6zm0-8h-2V7h2v2z';
    },
    calculateDimensions() {
      // 根据模块最大坐标计算画布尺寸（增加100px边距）
      const maxX = Math.max(...this.boxes.map(b => b.x + b.width));
      const maxY = Math.max(...this.boxes.map(b => b.y + b.height));
      this.width = maxX + 100;
      this.height = maxY + 100;
    },
    drawCanvas() {
      // 先清空画布
      d3.select(this.$refs.canvas).selectAll('*').remove();
      this.calculateDimensions();
      
      // 创建响应式SVG容器
      const svg = d3.select(this.$refs.canvas)
        .append('svg')
        .attr('width', '100%')
        .attr('height', '100%')
        .attr('viewBox', `-${this.initX()} -${this.initY()} ${window.innerWidth} ${window.innerHeight}`)
        .attr('preserveAspectRatio', 'xMidYMid meet');
        
      // 添加一个包含所有内容的主容器组
      const mainGroup = svg.append('g')
        .attr('class', 'main-group');
        
      // 设置缩放行为
      this.zoom = d3.zoom()
        .scaleExtent([0.3, 3]) // 设置缩放范围从30%到300%
        .on('zoom', (event) => {
          mainGroup.attr('transform', event.transform);
          this.currentZoom = event.transform.k;
        });
      
      // 应用缩放行为到SVG
      svg.call(this.zoom);
      
      // 如果有之前的缩放状态，恢复它
      if (this.currentZoom !== 1) {
        const transform = d3.zoomIdentity.scale(this.currentZoom);
        svg.call(this.zoom.transform, transform);
      }

      // 1. 绘制模块基础部分
      this.boxes.forEach(box => {
        const g = mainGroup.append('g')
          .attr('transform', `translate(${box.x}, ${box.y})`);

        // 模块矩形
        g.append('rect')
          .attr('width', box.width)
          .attr('height', box.height)
          .attr('fill', box.color)
          .attr('stroke', '#333');

        // 模块标题
        g.append('text')
          .attr('x', 10)
          .attr('y', 20)
          .attr('font-size', '16px')
          .attr('font-weight', 'bold')
          .text(box.title);

        // 模块副标题（英文）
        g.append('text')
          .attr('x', 10)
          .attr('y', 45)
          .attr('font-size', '12px')
          .attr('fill', '#666')
          .text(box.subtitle);

        // 模块图标（右上角）
        g.append('path')
          .attr('d', this.getBoxIcon(box.id))
          .attr('transform', `translate(${box.width - 40}, 20)`)
          .attr('fill', '#409EFF')
          .style('cursor', 'pointer')
          .attr('class', 'box-icon');
      });

      // 2. 绘制便利贴
      this.notes.forEach(note => {

        const g = mainGroup.append('g')
          .attr('transform', `translate(${note.x}, ${note.y})`)
          .classed('note', true) // 便于识别便利贴元素
          .call(d3.drag()
            .on('start', (event) => {
              // 在拖动时禁用画布的缩放/平移
              svg.on('.zoom', null);
              
              event.sourceEvent.stopPropagation();
              this.dragging = {
                target: note,
                offsetX: event.x - note.x,
                offsetY: event.y - note.y
              };
            })
            .on('drag', (event) => {
              if (!this.dragging) return;
              // 计算基于点击位置的偏移量
              note.x = event.x - this.dragging.offsetX;
              note.y = event.y - this.dragging.offsetY;
              g.attr('transform', `translate(${note.x}, ${note.y})`);
            })
            .on('end', () => {
              // 重新启用画布的缩放功能
              svg.call(this.zoom);
              
              this.dragging = null;
              // 触发响应式更新
              this.notes = [...this.notes];
            }));

        // 便利贴背景
        g.append('rect')
          .attr('width', 120)
          .attr('height', 80)
          .attr('fill', '#FFF9C4')
          .attr('rx', 5)
          .attr('stroke', '#FFD700');

        // 便利贴内容
        g.append('foreignObject')
          .attr('width', 110)
          .attr('height', 70)
          .attr('x', 5)
          .attr('y', 5)
          .append('xhtml:div')
          .style('font-size', '12px')
          .style('padding', '5px')
          .style('word-break', 'break-all')
          .html(note.content);

        // 删除按钮
        g.append('circle')
          .attr('cx', 110)
          .attr('cy', 5)
          .attr('r', 8)
          .attr('fill', '#ff4d4f')
          .style('cursor', 'pointer')
          .style('pointer-events', 'all')
          .on('click', function(event) {
            event.stopPropagation(); // 阻止事件冒泡
            this.selectedNote = note;
            this.dialogType = 'delete';
            this.showDialog = true;
          }.bind(this));
          
        g.append('text')
          .attr('x', 110)
          .attr('y', 5)
          .attr('text-anchor', 'middle')
          .attr('dominant-baseline', 'central')
          .attr('fill', 'white')
          .attr('font-size', '12px')
          .style('cursor', 'pointer')
          .style('pointer-events', 'none')
          .text('×');
      });

      // 3. 最后绘制按钮（确保在最顶层）
      this.boxes.forEach(box => {
        const g = mainGroup.append('g')
          .attr('transform', `translate(${box.x}, ${box.y})`)
          .style('cursor', 'pointer')
          .on('click', () => {
            this.selectedBox = box;
            this.showDialog = true;
          });

        // 加号按钮（右下角固定位置）
        const buttonSize = 30;
        // const buttonMargin = 15;
        
        const buttonGroup = g.append('g')
          .attr('transform', `translate(${box.width - 40}, ${box.height - 40})`)
          .style('cursor', 'pointer')
          .on('click', function(event) {
            event.stopPropagation();
            this.selectedBox = box;
            this.dialogType = 'add';
            this.showDialog = true;
            this.$nextTick(() => {
              this.$refs.noteInput?.focus();
            });
          }.bind(this));

        // 圆形背景
        buttonGroup.append('circle')
          .attr('cx', buttonSize/2)
          .attr('cy', buttonSize/2)
          .attr('r', buttonSize/2)
          .attr('fill', '#409EFF')
          .style('cursor', 'pointer')
          .style('pointer-events', 'all');

        // 加号图标
        buttonGroup.append('text')
          .attr('x', buttonSize/2)
          .attr('y', buttonSize/2 + 6)
          .attr('text-anchor', 'middle')
          .attr('fill', 'white')
          .attr('font-size', '24px')
          .attr('font-weight', 'bold')
          .text('+');
      });
    },
    initX() {
      if (window.innerWidth <= contentWidth||window.innerHeight <= contentHeight) {
        return 0
      }
      return window.innerWidth / 2 - contentWidth / 2
    },
    initY() {
      if (window.innerWidth <= contentWidth||window.innerHeight <= contentHeight) {
        return 0
      }
      return window.innerHeight / 2 - contentHeight / 2
    },
    // 缩放控制方法
    zoomIn() {
      const svg = d3.select(this.$refs.canvas).select('svg');
      svg.transition().duration(300).call(
        this.zoom.scaleBy, 1.2
      );
    },
    
    zoomOut() {
      const svg = d3.select(this.$refs.canvas).select('svg');
      svg.transition().duration(300).call(
        this.zoom.scaleBy, 0.8
      );
    },
    
    resetZoom() {
      const svg = d3.select(this.$refs.canvas).select('svg');
      svg.transition().duration(300).call(
        this.zoom.transform, d3.zoomIdentity
      );
      this.currentZoom = 1;
    },

    addNote(note) {
      const svg = d3.select(this.$refs.canvas).select('svg');
      const mainGroup = svg.select('.main-group');
      
      const g = mainGroup.append('g')
        .attr('transform', `translate(${note.x}, ${note.y})`)
        .classed('note', true)
        .call(d3.drag()
          .on('start', (event) => {
            svg.on('.zoom', null);
            event.sourceEvent.stopPropagation();
            this.dragging = {
              target: note,
              offsetX: event.x - note.x,
              offsetY: event.y - note.y
            };
          })
          .on('drag', (event) => {
            if (!this.dragging) return;
            note.x = event.x - this.dragging.offsetX;
            note.y = event.y - this.dragging.offsetY;
            g.attr('transform', `translate(${note.x}, ${note.y})`);
          })
          .on('end', () => {
            svg.call(this.zoom);
            this.dragging = null;
            this.notes = [...this.notes];
          }));

      // 便利贴背景
      g.append('rect')
        .attr('width', 120)
        .attr('height', 80)
        .attr('fill', '#FFF9C4')
        .attr('rx', 5)
        .attr('stroke', '#FFD700');

      // 便利贴内容
      g.append('foreignObject')
        .attr('width', 110)
        .attr('height', 70)
        .attr('x', 5)
        .attr('y', 5)
        .append('xhtml:div')
        .style('font-size', '12px')
        .style('padding', '5px')
        .style('word-break', 'break-all')
        .html(note.content);

      // 删除按钮
      g.append('circle')
        .attr('cx', 110)
        .attr('cy', 5)
        .attr('r', 8)
        .attr('fill', '#ff4d4f')
        .style('cursor', 'pointer')
        .style('pointer-events', 'all')
        .on('click', function(event) {
          event.stopPropagation();
          this.selectedNote = note;
          this.dialogType = 'delete';
          this.showDialog = true;
        }.bind(this));
        
      g.append('text')
        .attr('x', 110)
        .attr('y', 5)
        .attr('text-anchor', 'middle')
        .attr('dominant-baseline', 'central')
        .attr('fill', 'white')
        .attr('font-size', '12px')
        .style('cursor', 'pointer')
        .style('pointer-events', 'none')
        .text('×');
    },

    handleConfirm() {
      if (this.dialogType === 'add') {
        if (this.newNoteContent.trim()) {
          const note = {
            content: this.newNoteContent,
            x: this.selectedBox.x + Math.random() * (this.selectedBox.width - 120),
            y: this.selectedBox.y + Math.random() * (this.selectedBox.height - 80)
          };
          this.notes = [...this.notes, note];
          this.addNote(note);
        }
      } else if (this.dialogType === 'delete' && this.selectedNote) {
        d3.select(this.$refs.canvas).select(`g[transform="translate(${this.selectedNote.x}, ${this.selectedNote.y})"]`).remove();
        this.notes = this.notes.filter(n => n !== this.selectedNote);
      }
      
      this.showDialog = false;
      this.newNoteContent = '';
      this.selectedNote = null;
      this.dialogType = 'add';
    },

    handleCancel() {
      this.showDialog = false;
      this.newNoteContent = '';
    }
  }
};
</script>

<style scoped>
.canvas-container {
  overflow: hidden;
  width: 100%;
  height: 100%;
}

/* 缩放控制按钮样式 */
.zoom-controls {
  position: fixed;
  bottom: 20px;
  right: 20px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  z-index: 100;
}

.zoom-button {
  width: 40px;
  height: 40px;
  background: #409EFF;
  color: white;
  border: none;
  border-radius: 50%;
  font-size: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  box-shadow: 0 2px 5px rgba(0,0,0,0.2);
  transition: all 0.2s;
}

.zoom-button:hover {
  background: #2d8cf0;
  transform: scale(1.05);
}

.zoom-button:active {
  transform: scale(0.95);
}

/* 弹窗样式 */
.dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.dialog-content {
  background: white;
  padding: 20px;
  border-radius: 8px;
  width: 400px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.dialog-content h3 {
  margin: 0 8px 15px;
  color: #333;
}

.dialog-content textarea {
  width: 368px; /* 左右各留8px间距 */
  height: 100px;
  padding: 8px;
  margin: 0 8px 15px; /* 左右对称8px边距 */
  border: 1px solid #ddd;
  border-radius: 4px;
  resize: vertical;
  display: block; /* 确保居中显示 */
}

.dialog-buttons {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
}

.dialog-buttons button {
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.dialog-buttons button:first-child {
  background: #409EFF;
  color: white;
}

.dialog-buttons button:last-child {
  background: #f0f0f0;
  color: #666;
}

.box-icon {
  transition: all 0.2s ease;
}

.box-icon:hover {
  fill: #2d8cf0;
  transform: scale(1.1);
}
</style>
