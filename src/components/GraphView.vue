<template>
  <div>
    <svg>
      <g ref='scene'>
        <!--
          Note: I wish I could entirely use vue template to set the UI.
          Unfortunately this produces choppy layout. So I set most of the static
          components with vue, and update positions from javascript.
        -->
        <g ref='linksContainer'>
          <path v-for='edge in edges'
            stroke='#999'
            :data-from='edge.fromId'
            :data-to='edge.toId'
            marker-end='url(#Triangle)'></path>
        </g>
        <g ref='nodeContainer'>
          <image v-for='node in nodes'
            :width='node.width' :height='node.height'
            class='video-thumbnail'
            :data-id='node.id'
            :xlink:href='node.data.image.url' />
        </g>
      </g>
      <defs>
        <marker id='Triangle' viewBox='0 0 10 10' refX='5' refY='5' markerUnits='strokeWidth' markerWidth='10' markerHeight='5' orient='auto'>
          <path d='M 0 0 L 10 5 L 0 10 z' fill='#999'></path>
        </marker>
      </defs>
    </svg>
    <div class='tooltip' v-if='tooltip.visible' :style='{
       left: tooltip.x + "px",
       top: tooltip.y + "px"
      }'>
      {{tooltip.text}}
    </div>
  </div>
</template>

<script>
import panZoom from 'panzoom';
import createLayout from 'ngraph.forcelayout';

import { getFromTo } from '../lib/geom.js';
import clap from '../lib/clap.js';

export default {
  name: 'GraphView',
  props: ['graph'],
  data() {
    return {
      nodes: [],
      edges: [],
      tooltip: {
        visible: false,
        text: '',
        x: 0,
        y: 0
      }
    };
  },
  watch: {
    graph(newGraph, oldGraph) {
      if (oldGraph) {
        this.disposeGraph();
      }

      this.initializeGraph(newGraph);
    }
  },

  mounted() {
    const { scene } = this.$refs;
    this.zoomHandle = panZoom(scene);
    const rect = this.$el.getBoundingClientRect();
    this.zoomHandle.moveTo(rect.width / 2, rect.height / 2);
    this.initEvents(scene);
    this.initializeGraph(this.graph);
  },

  methods: {
    handleMouseEnter(e) {
      const nodeId = getNodeIdFromDOM(e.target);
      if (!nodeId) return;

      this.clearHighligh();

      forAll(this.$refs.scene, `path[data-from="${nodeId}"], path[data-to="${nodeId}"]`, addClass('hl'));

      const { title } = this.graph.getNode(nodeId).data;
      this.showTooltip(e.target, title);
    },

    handleMouseClick(e) {
      const nodeId = getNodeIdFromDOM(e.target);
      if (!nodeId) return;
      if (e.shiftKey) {
        this.$emit('loadMore', nodeId);
      } else {
        this.$emit('selected', nodeId);
      }
    },

    showTooltip(el, text) {
      const rect = el.getBoundingClientRect();
      const { tooltip } = this;
      tooltip.text = text;
      tooltip.x = rect.left + rect.width / 2;
      tooltip.y = rect.top - 42 / 2;
      tooltip.visible = true;
    },

    handleMouseLeave() {
      this.clearHighligh();
      this.tooltip.visible = false;
    },

    clearHighligh() {
      forAll(this.$refs.scene, '.hl', removeClass('hl'));
    },

    onGraphChanged(changes) {
      this.iterations = 0;
      changes.forEach(change => {
        if (change.changeType === 'add') {
          if (change.node) this.addNode(change.node);
          if (change.link) this.addLink(change.link);
        } else {
          throw new Error('Not implemented');
        }
      });

      this.frame();
    },

    addNode(graphNode) {
      const { id, data } = graphNode;
      const pos = this.layout.getNodePosition(id);

      this.nodes.push({
        pos,
        id,
        data,
        width: data.width,
        height: data.height
      });
    },

    addLink(graphLink) {
      const { id, fromId, toId } = graphLink;
      const pos = this.layout.getLinkPosition(id);
      const fromHeight = this.graph.getNode(fromId).data.height;
      const toHeight = this.graph.getNode(toId).data.height;
      this.edges.push({ pos, fromId, toId, fromHeight, toHeight });
    },

    disposeGraph() {
      if (this.graph) {
        this.graph.off('changed', this.onGraphChanged, this);
        this.nodes = [];
        this.edges = [];
      }
      if (this.layout) {
        this.layout.dispose();
        this.layout = null;
      }
      if (this.layoutLoop) {
        cancelAnimationFrame(this.layoutLoop);
        this.layoutLoop = null;
      }
    },

    initEvents(scene) {
      // TODO: This should be disposed properly
      scene.addEventListener('mouseenter', this.handleMouseEnter.bind(this), true);
      scene.addEventListener('mouseleave', this.handleMouseLeave.bind(this), true);
      this.clapDispose = clap(scene, this.handleMouseClick.bind(this), true);
    },

    initializeGraph(newGraph) {
      if (!newGraph) return;

      this.layout = createLayout(newGraph, {
        springLength: 80,
        springCoeff: 0.0004,
        dragCoeff: 0.05,
        gravity: -50,
        theta: 0.5
      });

      this.nodes = [];
      this.edges = [];

      const graph = newGraph;

      graph.forEachNode((node) => {
        this.addNode(node);
      });

      graph.forEachLink((link) => {
        this.addLink(link);
      });

      graph.on('changed', this.onGraphChanged, this);

      this.boundFrame = this.frame.bind(this);
      this.frame();
    },

    frame() {
      if (!this.layout) return;

      this.layout.step();
      this.edges.forEach(edge => {
        let ui = edge.ui;
        if (!ui) {
          ui = this.$refs.linksContainer.querySelector(`path[data-from="${edge.fromId}"][data-to="${edge.toId}"]`);
          edge.ui = ui;
        }
        if (ui) ui.setAttributeNS(null, 'd', computeLinkPath(edge.pos, edge.fromHeight, edge.toHeight));
      });

      this.nodes.forEach(node => {
        let ui = node.ui;
        if (!ui) {
          ui = this.$refs.nodeContainer.querySelector(`[data-id="${node.id}"]`);
          node.ui = ui;
        }
        if (ui) {
          ui.setAttributeNS(null, 'x', node.pos.x - node.width / 2);
          ui.setAttributeNS(null, 'y', node.pos.y - node.height / 2);
        }
      });

      this.iterations += 1;
      if (this.layout.lastMove > 20 || this.iterations < 100) {
        // we assume we are note converged yet.
        this.layoutLoop = requestAnimationFrame(this.boundFrame);
      }
    }
  },

  destroyed() {
    if (this.zoomHandle) {
      this.zoomHandle.dispose();
    }
    this.disposeGraph();
    if (this.clapDispose) {
      this.clapDispose();
    }
  }
};

function computeLinkPath(edge, fromHeight, toHeight) {
  const { from, to } = getFromTo(edge, fromHeight, toHeight);
  let data = 'M';

  data += Math.round(from.x) + ',' + Math.round(from.y);
  data += 'L' + Math.round(to.x) + ',' + Math.round(to.y);

  return data;
}

function getNodeIdFromDOM(el) {
  const isNode = (el && el.classList.contains('video-thumbnail'));
  if (!isNode) return;

  return el.getAttribute('data-id');
}

function forAll(scene, query, action) {
  (Array.from(scene.querySelectorAll(query))).forEach(action);
}

function addClass(className) {
  return el => el.classList.add(className);
}

function removeClass(className) {
  return el => el.classList.remove(className);
}
</script>

<style scoped>
svg {
  width: 100%;
  height: 100%;
}
path.hl {
  stroke: RGB(51, 182, 121);
  stroke-width: 2px;
}
.video-thumbnail {
  cursor: pointer;
}
.tooltip {
  position: absolute;
  background: rgba(255, 255, 255, 0.8);
  pointer-events: none;
  transform: translate(-50%, 0);
  white-space: nowrap;
}
</style>
