<script setup lang="ts">
import { ref, shallowRef, onMounted, onUnmounted, watch, computed, nextTick, markRaw } from 'vue';
import * as d3 from 'd3';
import { 
  Search, ZoomIn, ZoomOut, Maximize, RefreshCw, Info, Upload, Layers 
} from 'lucide-vue-next';

interface Node extends d3.SimulationNodeDatum {
  id: string;
  group: number;
  radius: number;
  color: string;
  label: string;
}

interface Link extends d3.SimulationLinkDatum<Node> {
  source: string | Node;
  target: string | Node;
  value: number;
}

interface GraphData {
  nodes: Node[];
  links: Link[];
}

const COLORS = [
  '#3b82f6', '#ef4444', '#10b981', '#f59e0b', '#8b5cf6', 
  '#ec4899', '#06b6d4', '#84cc16', '#f97316', '#6366f1'
];

const canvasRef = ref<HTMLCanvasElement | null>(null);
const containerRef = ref<HTMLDivElement | null>(null);
const data = shallowRef<GraphData>({ nodes: [], links: [] });
const loading = ref(false);
const stats = ref({ nodes: 0, links: 0, groups: 0 });
const kValue = ref(10);
const searchQuery = ref('');
const hoveredNode = shallowRef<Node | null>(null);
const selectedNode = shallowRef<Node | null>(null);
const transform = shallowRef<d3.ZoomTransform>(d3.zoomIdentity);
const isSimulating = ref(false);
const dimensions = ref({ width: 0, height: 0 });
const graphUrl = ref('');

// Simulation and Zoom refs
const simulationRef = shallowRef<d3.Simulation<Node, Link> | null>(null);
const zoomRef = shallowRef<d3.ZoomBehavior<HTMLCanvasElement, unknown> | null>(null);

// Reusable graph data processor
const processGraphData = (content: string) => {
  const lines = content.split('\n');
  const nodeMap = new Map<string, Node>();
  const links: Link[] = [];

  lines.forEach((line) => {
    const trimmed = line.trim();
    if (!trimmed) return;

    const parts = trimmed.split('<-f-');
    if (parts.length === 2) {
      const targetId = parts[0].trim();
      const sourceId = parts[1].trim();

      if (!nodeMap.has(sourceId)) {
        nodeMap.set(sourceId, {
          id: sourceId,
          group: Math.floor(Math.random() * 10),
          radius: 4,
          color: COLORS[Math.floor(Math.random() * COLORS.length)],
          label: sourceId,
          x: Math.random() * 2000 - 1000,
          y: Math.random() * 2000 - 1000,
        });
      }
      if (!nodeMap.has(targetId)) {
        nodeMap.set(targetId, {
          id: targetId,
          group: Math.floor(Math.random() * 10),
          radius: 4,
          color: COLORS[Math.floor(Math.random() * COLORS.length)],
          label: targetId,
          x: Math.random() * 2000 - 1000,
          y: Math.random() * 2000 - 1000,
        });
      }

      links.push({
        source: sourceId,
        target: targetId,
        value: 1,
      });
    }
  });

  const nodes = Array.from(nodeMap.values());
  data.value = markRaw({ nodes, links });
  stats.value = { 
    nodes: nodes.length, 
    links: links.length, 
    groups: new Set(nodes.map(n => n.group)).size 
  };
  
  // Reset zoom to see the whole graph
  if (zoomRef.value && canvasRef.value) {
    d3.select(canvasRef.value).call(zoomRef.value.transform, d3.zoomIdentity);
  }
};

// Handle file upload and parsing
const handleFileUpload = (event: Event) => {
  const target = event.target as HTMLInputElement;
  const file = target.files?.[0];
  if (!file) return;

  loading.value = true;
  const reader = new FileReader();
  reader.onload = (e) => {
    const content = e.target?.result as string;
    processGraphData(content);
    loading.value = false;
  };
  reader.onerror = () => {
    loading.value = false;
    alert('Failed to read file');
  };
  reader.readAsText(file);
};

// Handle URL load
const handleUrlLoad = async () => {
  if (!graphUrl.value.trim()) return;
  
  loading.value = true;
  try {
    const response = await fetch(graphUrl.value);
    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
    const content = await response.text();
    processGraphData(content);
  } catch (error) {
    console.error('Failed to load graph from URL:', error);
    alert('Failed to load graph from URL. Ensure the URL is accessible and CORS is enabled.');
  } finally {
    loading.value = false;
  }
};

// K-means Clustering Algorithm (Edge-Aware)
const runKMeans = (k: number) => {
  if (data.value.nodes.length === 0) return;
  
  loading.value = true;
  const nodes = data.value.nodes;
  const links = data.value.links;
  
  // 1. Calculate structural features for each node
  const nodeFeatures = new Map<string, { 
    x: number, y: number, 
    degree: number, 
    nx: number, ny: number 
  }>();
  
  // Initialize with spatial data
  nodes.forEach(node => {
    nodeFeatures.set(node.id, { 
      x: node.x || 0, y: node.y || 0, 
      degree: 0, nx: 0, ny: 0 
    });
  });
  
  // Calculate degree and neighbor centroids
  links.forEach(link => {
    const s = typeof link.source === 'string' ? link.source : (link.source as Node).id;
    const t = typeof link.target === 'string' ? link.target : (link.target as Node).id;
    
    const fs = nodeFeatures.get(s);
    const ft = nodeFeatures.get(t);
    
    if (fs && ft) {
      fs.degree++;
      fs.nx += ft.x;
      fs.ny += ft.y;
      
      ft.degree++;
      ft.nx += fs.x;
      ft.ny += fs.y;
    }
  });
  
  // Finalize neighbor averages and normalize features
  let maxDegree = 0;
  nodes.forEach(node => {
    const f = nodeFeatures.get(node.id)!;
    if (f.degree > 0) {
      f.nx /= f.degree;
      f.ny /= f.degree;
    } else {
      f.nx = f.x;
      f.ny = f.y;
    }
    maxDegree = Math.max(maxDegree, f.degree);
  });

  // 2. Initialize centroids in the augmented feature space
  // Feature vector: [x, y, degree_normalized, nx, ny]
  // We'll weight spatial vs structural. Let's use 0.5 each.
  const getFeature = (nodeId: string) => {
    const f = nodeFeatures.get(nodeId)!;
    return [
      f.x, f.y, 
      (f.degree / (maxDegree || 1)) * 500, // Scale degree to match spatial range roughly
      f.nx, f.ny
    ];
  };

  let centroids = Array.from({ length: k }, () => {
    const randomNode = nodes[Math.floor(Math.random() * nodes.length)];
    return getFeature(randomNode.id);
  });

  const maxIterations = 25;
  let iterations = 0;
  let changed = true;

  while (changed && iterations < maxIterations) {
    changed = false;
    iterations++;

    // Assign nodes to nearest centroid in 5D space
    nodes.forEach(node => {
      const f = getFeature(node.id);
      let minDist = Infinity;
      let closestCluster = 0;
      
      centroids.forEach((centroid, i) => {
        let dist = 0;
        for (let j = 0; j < f.length; j++) {
          const diff = f[j] - centroid[j];
          dist += diff * diff;
        }
        
        if (dist < minDist) {
          minDist = dist;
          closestCluster = i;
        }
      });

      if (node.group !== closestCluster) {
        node.group = closestCluster;
        node.color = COLORS[closestCluster % COLORS.length];
        changed = true;
      }
    });

    // Update centroids
    const newCentroidSums = Array.from({ length: k }, () => Array(5).fill(0).concat([0])); // [x, y, d, nx, ny, count]
    nodes.forEach(node => {
      const f = getFeature(node.id);
      const sum = newCentroidSums[node.group];
      for (let j = 0; j < 5; j++) sum[j] += f[j];
      sum[5]++; // count
    });

    centroids = newCentroidSums.map(sum => {
      const count = sum[5];
      if (count === 0) return Array.from({ length: 5 }, () => Math.random() * 1000 - 500);
      return sum.slice(0, 5).map(v => v / count);
    });
  }

  stats.value.groups = k;
  loading.value = false;
  
  if (simulationRef.value) {
    simulationRef.value.alpha(0.1).restart();
  }
};

const autoCluster = () => {
  const suggestedK = Math.max(2, Math.min(30, Math.floor(Math.sqrt(data.value.nodes.length / 2))));
  kValue.value = suggestedK;
  runKMeans(suggestedK);
};

const handleSearch = (query: string) => {
  searchQuery.value = query;
  if (!query.trim()) {
    selectedNode.value = null;
    return;
  }

  const found = data.value.nodes.find(n => 
    n.label.toLowerCase().includes(query.toLowerCase()) || 
    n.id.toLowerCase().includes(query.toLowerCase())
  );

  if (found) {
    selectedNode.value = found;
    if (zoomRef.value && canvasRef.value) {
      const width = dimensions.value.width;
      const height = dimensions.value.height;
      const k = 4;
      const x = width / 2 - found.x! * k;
      const y = height / 2 - found.y! * k;
      
      d3.select(canvasRef.value)
        .transition()
        .duration(750)
        .call(zoomRef.value.transform, d3.zoomIdentity.translate(x, y).scale(k));
    }
  }
};

// Render function
const render = (ctx: CanvasRenderingContext2D, width: number, height: number, k: number, tx: number, ty: number) => {
  const currentData = data.value;
  const currentSelectedNode = selectedNode.value;
  const currentHoveredNode = hoveredNode.value;
  
  ctx.save();
  ctx.clearRect(0, 0, width, height);
  ctx.translate(tx, ty);
  ctx.scale(k, k);

  const neighborIds = new Set<string>();
  if (currentSelectedNode) {
    neighborIds.add(currentSelectedNode.id);
    currentData.links.forEach(link => {
      const s = link.source as Node;
      const t = link.target as Node;
      if (s.id === currentSelectedNode.id) neighborIds.add(t.id);
      if (t.id === currentSelectedNode.id) neighborIds.add(s.id);
    });
  }

  ctx.beginPath();
  ctx.strokeStyle = currentSelectedNode ? 'rgba(200, 200, 200, 0.05)' : 'rgba(200, 200, 200, 0.15)';
  ctx.lineWidth = 0.5 / k;
  
  if (k > 0.4) {
    currentData.links.forEach(link => {
      const s = link.source as Node;
      const t = link.target as Node;
      if (currentSelectedNode && (s.id === currentSelectedNode.id || t.id === currentSelectedNode.id)) return;
      ctx.moveTo(s.x!, s.y!);
      ctx.lineTo(t.x!, t.y!);
    });
    ctx.stroke();
  }

  if (currentSelectedNode) {
    currentData.links.forEach(link => {
      const s = link.source as Node;
      const t = link.target as Node;
      if (s.id === currentSelectedNode.id || t.id === currentSelectedNode.id) {
        ctx.beginPath();
        ctx.lineWidth = 2 / k;
        if (s.id === currentSelectedNode.id) {
          ctx.strokeStyle = '#f43f5e';
          ctx.setLineDash([]);
        } else {
          ctx.strokeStyle = '#10b981';
          ctx.setLineDash([5 / k, 5 / k]);
        }
        ctx.moveTo(s.x!, s.y!);
        ctx.lineTo(t.x!, t.y!);
        ctx.stroke();
        ctx.setLineDash([]);
      }
    });
  }

  currentData.nodes.forEach(node => {
    const isActive = !currentSelectedNode || neighborIds.has(node.id);
    ctx.beginPath();
    if (isActive) {
      ctx.fillStyle = node.color;
      ctx.globalAlpha = 1;
    } else {
      ctx.fillStyle = '#e2e8f0';
      ctx.globalAlpha = 0.3;
    }
    const r = node.radius / Math.sqrt(k);
    ctx.arc(node.x!, node.y!, r, 0, 2 * Math.PI);
    ctx.fill();
    if (currentHoveredNode?.id === node.id || currentSelectedNode?.id === node.id) {
      ctx.globalAlpha = 1;
      ctx.strokeStyle = currentSelectedNode?.id === node.id ? '#3b82f6' : '#fff';
      ctx.lineWidth = (currentSelectedNode?.id === node.id ? 3 : 2) / k;
      ctx.stroke();
    }
    ctx.globalAlpha = 1;
  });

  if (k > 1.5) {
    ctx.font = `${12 / k}px Inter, sans-serif`;
    ctx.fillStyle = '#334155';
    ctx.textAlign = 'center';
    currentData.nodes.forEach(node => {
      const isActive = !currentSelectedNode || neighborIds.has(node.id);
      if (isActive) {
        ctx.fillText(node.label, node.x!, node.y! + (node.radius + 10) / k);
      }
    });
  }
  ctx.restore();
};

// Handle resizing
let resizeObserver: ResizeObserver | null = null;
onMounted(() => {
  if (containerRef.value) {
    resizeObserver = new ResizeObserver((entries) => {
      if (entries[0]) {
        const { width, height } = entries[0].contentRect;
        dimensions.value = { width, height };
      }
    });
    resizeObserver.observe(containerRef.value);
  }
});

onUnmounted(() => {
  resizeObserver?.disconnect();
});

// Setup simulation and zoom
watch([data, dimensions], () => {
  if (!canvasRef.value || data.value.nodes.length === 0 || dimensions.value.width === 0) return;

  const canvas = canvasRef.value;
  const context = canvas.getContext('2d');
  if (!context) return;

  const { width, height } = dimensions.value;
  canvas.width = width * window.devicePixelRatio;
  canvas.height = height * window.devicePixelRatio;
  context.scale(window.devicePixelRatio, window.devicePixelRatio);

  const groupCenters = new Map<number, { x: number, y: number }>();
  if (stats.value.groups > 1) {
    const groupSums = new Map<number, { x: number, y: number, count: number }>();
    data.value.nodes.forEach(node => {
      const current = groupSums.get(node.group) || { x: 0, y: 0, count: 0 };
      groupSums.set(node.group, {
        x: current.x + (node.x || 0),
        y: current.y + (node.y || 0),
        count: current.count + 1
      });
    });
    groupSums.forEach((val, key) => {
      groupCenters.set(key, { x: val.x / val.count, y: val.y / val.count });
    });
  }

  if (simulationRef.value) simulationRef.value.stop();

  const simulation = markRaw(d3.forceSimulation<Node>(data.value.nodes)
    .force('link', d3.forceLink<Node, Link>(data.value.links).id(d => d.id).distance(40).strength(0.05))
    .force('charge', d3.forceManyBody().strength(-40).distanceMax(300))
    .force('center', d3.forceCenter(width / 2, height / 2))
    .force('collide', d3.forceCollide().radius(d => (d as Node).radius + 3))
    .force('x', d3.forceX(node => {
      const n = node as Node;
      return groupCenters.get(n.group)?.x || width / 2;
    }).strength(() => stats.value.groups > 1 ? 0.05 : 0))
    .force('y', d3.forceY(node => {
      const n = node as Node;
      return groupCenters.get(n.group)?.y || height / 2;
    }).strength(() => stats.value.groups > 1 ? 0.05 : 0))
    .alphaDecay(0.03));

  simulationRef.value = simulation;
  isSimulating.value = true;

  const zoom = markRaw(d3.zoom<HTMLCanvasElement, unknown>()
    .scaleExtent([0.02, 40])
    .on('zoom', (event) => {
      transform.value = event.transform;
    }));

  zoomRef.value = zoom;
  d3.select(canvas).call(zoom);

  simulation.on('tick', () => {
    render(context, width, height, transform.value.k, transform.value.x, transform.value.y);
  });

  simulation.on('end', () => {
    isSimulating.value = false;
    render(context, width, height, transform.value.k, transform.value.x, transform.value.y);
  });
}, { immediate: true });

// Mouse events
const handleMouseMove = (event: MouseEvent) => {
  if (!canvasRef.value) return;
  const canvas = canvasRef.value;
  const rect = canvas.getBoundingClientRect();
  const x = (event.clientX - rect.left - transform.value.x) / transform.value.k;
  const y = (event.clientY - rect.top - transform.value.y) / transform.value.k;

  let found: Node | null = null;
  const searchRadius = 10 / transform.value.k;
  
  for (const node of data.value.nodes) {
    const dx = node.x! - x;
    const dy = node.y! - y;
    if (dx * dx + dy * dy < (node.radius + searchRadius) ** 2) {
      found = node;
      break;
    }
  }
  
  if (found?.id !== hoveredNode.value?.id) {
    hoveredNode.value = found;
  }
};

const handleClick = (event: MouseEvent) => {
  if (!canvasRef.value) return;
  const canvas = canvasRef.value;
  const rect = canvas.getBoundingClientRect();
  const x = (event.clientX - rect.left - transform.value.x) / transform.value.k;
  const y = (event.clientY - rect.top - transform.value.y) / transform.value.k;

  let found: Node | null = null;
  const searchRadius = 10 / transform.value.k;
  
  for (const node of data.value.nodes) {
    const dx = node.x! - x;
    const dy = node.y! - y;
    if (dx * dx + dy * dy < (node.radius + searchRadius) ** 2) {
      found = node;
      break;
    }
  }
  
  selectedNode.value = found;
};

// Re-render when transform, hoveredNode, or selectedNode changes
watch([transform, hoveredNode, selectedNode, dimensions], () => {
  if (!canvasRef.value || dimensions.value.width === 0) return;
  const canvas = canvasRef.value;
  const context = canvas.getContext('2d');
  if (!context) return;
  render(context, dimensions.value.width, dimensions.value.height, transform.value.k, transform.value.x, transform.value.y);
});

const handleZoomIn = () => {
  if (zoomRef.value && canvasRef.value) {
    d3.select(canvasRef.value).transition().duration(300).call(zoomRef.value.scaleBy, 1.5);
  }
};

const handleZoomOut = () => {
  if (zoomRef.value && canvasRef.value) {
    d3.select(canvasRef.value).transition().duration(300).call(zoomRef.value.scaleBy, 0.7);
  }
};

const handleReset = () => {
  if (zoomRef.value && canvasRef.value) {
    d3.select(canvasRef.value).transition().duration(500).call(zoomRef.value.transform, d3.zoomIdentity);
  }
};

const toggleSimulation = () => {
  if (simulationRef.value) {
    if (isSimulating.value) {
      simulationRef.value.stop();
      isSimulating.value = false;
    } else {
      simulationRef.value.alpha(0.3).restart();
      isSimulating.value = true;
    }
  }
};
</script>

<template>
  <div class="relative w-full h-full bg-slate-50 overflow-hidden font-sans" ref="containerRef">
    <!-- Canvas -->
    <canvas 
      ref="canvasRef" 
      class="w-full h-full cursor-grab active:cursor-grabbing"
      @mousemove="handleMouseMove"
      @click="handleClick"
    />

    <!-- Empty State -->
    <div v-if="!loading && data.nodes.length === 0" class="absolute inset-0 flex flex-col items-center justify-center text-slate-400 pointer-events-none">
      <Upload :size="48" class="mb-4 opacity-20" />
      <p class="text-lg font-medium">No graph data loaded</p>
      <p class="text-sm">Upload a file to start exploring</p>
    </div>

    <!-- UI Overlay -->
    <div class="absolute top-6 left-6 flex flex-col gap-4 pointer-events-none">
      <div class="bg-white/90 backdrop-blur-md p-6 rounded-2xl shadow-xl border border-slate-200 pointer-events-auto w-80">
        <h1 class="text-2xl font-bold text-slate-900 mb-1">Huge Graph</h1>
        <p class="text-sm text-slate-500 mb-6">Visualizing complex relationships at scale</p>
        
        <div class="space-y-4">
          <div class="relative">
            <Search class="absolute left-3 top-1/2 -translate-y-1/2 text-slate-400" :size="16" />
            <input 
              type="text"
              placeholder="Search node by name..."
              :value="searchQuery"
              @input="(e) => handleSearch((e.target as HTMLInputElement).value)"
              class="w-full pl-10 pr-4 py-2 bg-slate-50 border border-slate-200 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 transition-all pointer-events-auto"
            />
          </div>

          <div class="flex justify-between items-center text-sm">
            <span class="text-slate-600 font-medium">Nodes</span>
            <span class="bg-blue-100 text-blue-700 px-2 py-0.5 rounded-full font-mono font-bold">
              {{ stats.nodes.toLocaleString() }}
            </span>
          </div>
          <div class="flex justify-between items-center text-sm">
            <span class="text-slate-600 font-medium">Links</span>
            <span class="bg-emerald-100 text-emerald-700 px-2 py-0.5 rounded-full font-mono font-bold">
              {{ stats.links.toLocaleString() }}
            </span>
          </div>
          <div class="flex justify-between items-center text-sm">
            <span class="text-slate-600 font-medium">Groups</span>
            <span class="bg-purple-100 text-purple-700 px-2 py-0.5 rounded-full font-mono font-bold">
              {{ stats.groups.toLocaleString() }}
            </span>
          </div>
          
          <div class="pt-4 border-t border-slate-100">
            <div class="flex justify-between items-center mb-2">
              <label class="text-xs font-semibold text-slate-400 uppercase tracking-wider">
                Clustering (K-Means)
              </label>
              <button 
                @click="autoCluster"
                class="text-[10px] bg-slate-100 hover:bg-slate-200 text-slate-600 px-2 py-0.5 rounded transition-colors"
              >
                Auto
              </button>
            </div>
            <div class="flex gap-3 items-center">
              <input 
                type="range" 
                min="2" 
                max="30" 
                step="1"
                v-model.number="kValue"
                class="flex-1 h-2 bg-slate-200 rounded-lg appearance-none cursor-pointer accent-purple-600"
              />
              <span class="text-xs font-mono font-bold text-purple-600 w-4">{{ kValue }}</span>
              <button 
                @click="runKMeans(kValue)"
                :disabled="data.nodes.length === 0"
                class="p-1.5 bg-purple-600 hover:bg-purple-700 disabled:bg-slate-300 text-white rounded-lg transition-colors"
                title="Run Clustering"
              >
                <Layers :size="14" />
              </button>
            </div>
          </div>

          <div class="pt-4 border-t border-slate-100">
            <label class="text-xs font-semibold text-slate-400 uppercase tracking-wider mb-2 block">
              Load Graph from URL
            </label>
            <div class="flex gap-2">
              <input 
                type="text"
                placeholder="https://example.com/graph.txt"
                v-model="graphUrl"
                @keyup.enter="handleUrlLoad"
                class="flex-1 px-3 py-2 bg-slate-50 border border-slate-200 rounded-xl text-xs focus:outline-none focus:ring-2 focus:ring-blue-500 transition-all pointer-events-auto"
              />
              <button 
                @click="handleUrlLoad"
                class="p-2 bg-blue-600 hover:bg-blue-700 text-white rounded-xl transition-colors"
                title="Load from URL"
              >
                <RefreshCw :size="14" :class="{ 'animate-spin': loading }" />
              </button>
            </div>
          </div>

          <div class="pt-4 border-t border-slate-100">
            <label class="text-xs font-semibold text-slate-400 uppercase tracking-wider mb-2 block">
              Upload Graph File
            </label>
            <div class="relative group">
              <input 
                type="file" 
                accept=".txt,.csv,.graph"
                @change="handleFileUpload"
                class="absolute inset-0 w-full h-full opacity-0 cursor-pointer z-10"
              />
              <div class="flex items-center justify-center gap-2 w-full py-3 px-4 bg-blue-50 text-blue-600 rounded-xl border-2 border-dashed border-blue-200 group-hover:bg-blue-100 group-hover:border-blue-300 transition-all">
                <Upload :size="18" />
                <span class="text-sm font-medium">Choose File</span>
              </div>
            </div>
            <p class="text-[10px] text-slate-400 mt-2">
              Format: TARGET &lt;-f- SOURCE (one per line)
            </p>
          </div>
        </div>
      </div>

      <Transition name="fade">
        <div v-if="hoveredNode || selectedNode" class="bg-slate-900 text-white p-4 rounded-xl shadow-2xl pointer-events-auto border border-slate-700">
          <div class="flex items-center gap-3">
            <div class="w-3 h-3 rounded-full" :style="{ backgroundColor: (hoveredNode || selectedNode)!.color }" />
            <div>
              <div class="font-bold">{{ (hoveredNode || selectedNode)!.label }}</div>
              <div class="text-xs text-slate-400">Group {{ (hoveredNode || selectedNode)!.group }} • ID: {{ (hoveredNode || selectedNode)!.id }}</div>
            </div>
          </div>
          
          <div v-if="selectedNode" class="mt-4 pt-4 border-t border-slate-700 space-y-2">
            <div class="flex items-center justify-between text-xs">
              <span class="flex items-center gap-2">
                <div class="w-2 h-0.5 bg-[#f43f5e]" /> OUT Edges
              </span>
              <span class="font-mono text-rose-400">
                {{ data.links.filter(l => (l.source as Node).id === selectedNode?.id).length }}
              </span>
            </div>
            <div class="flex items-center justify-between text-xs">
              <span class="flex items-center gap-2">
                <div class="w-2 h-0.5 bg-[#10b981] border-b border-dashed" /> IN Edges
              </span>
              <span class="font-mono text-emerald-400">
                {{ data.links.filter(l => (l.target as Node).id === selectedNode?.id).length }}
              </span>
            </div>
            <button 
              @click="selectedNode = null"
              class="w-full mt-2 py-1.5 bg-slate-800 hover:bg-slate-700 rounded-lg text-[10px] transition-colors"
            >
              Clear Selection
            </button>
          </div>
        </div>
      </Transition>
    </div>

    <!-- Controls -->
    <div class="absolute bottom-8 right-8 flex flex-col gap-2 pointer-events-auto">
      <div class="flex flex-col bg-white/90 backdrop-blur-md rounded-2xl shadow-lg border border-slate-200 overflow-hidden">
        <button @click="handleZoomIn" title="Zoom In" class="p-3 hover:bg-slate-100 transition-colors border-b border-slate-100 text-slate-600">
          <ZoomIn :size="20" />
        </button>
        <button @click="handleZoomOut" title="Zoom Out" class="p-3 hover:bg-slate-100 transition-colors border-b border-slate-100 text-slate-600">
          <ZoomOut :size="20" />
        </button>
        <button @click="handleReset" title="Reset View" class="p-3 hover:bg-slate-100 transition-colors text-slate-600">
          <Maximize :size="20" />
        </button>
      </div>
      
      <div class="bg-white/90 backdrop-blur-md rounded-2xl shadow-lg border border-slate-200 overflow-hidden">
        <button 
          @click="toggleSimulation" 
          :title="isSimulating ? 'Stop Simulation' : 'Start Simulation'"
          class="p-3 hover:bg-slate-100 transition-colors text-slate-600"
          :class="{ 'text-blue-600 bg-blue-50': isSimulating }"
        >
          <RefreshCw :size="20" :class="{ 'animate-spin': isSimulating }" />
        </button>
      </div>
    </div>

    <!-- Loading Overlay -->
    <Transition name="fade">
      <div v-if="loading" class="absolute inset-0 bg-slate-50/80 backdrop-blur-sm flex flex-col items-center justify-center z-50">
        <div class="w-16 h-16 border-4 border-blue-600 border-t-transparent rounded-full animate-spin mb-4" />
        <p class="text-slate-600 font-medium">Processing graph data...</p>
      </div>
    </Transition>

    <!-- Help Info -->
    <div class="absolute bottom-8 left-8">
      <div class="group relative">
        <div class="bg-white/90 backdrop-blur-md p-2 rounded-full shadow-md border border-slate-200 cursor-help">
          <Info :size="18" class="text-slate-400" />
        </div>
        <div class="absolute bottom-full left-0 mb-2 w-64 bg-slate-900 text-white p-4 rounded-xl shadow-2xl opacity-0 group-hover:opacity-100 transition-opacity pointer-events-none text-xs leading-relaxed">
          <p class="font-bold mb-2">Navigation Guide</p>
          <ul class="space-y-1 text-slate-300">
            <li>• Drag to pan the graph</li>
            <li>• Scroll to zoom in/out</li>
            <li>• Hover nodes for details</li>
            <li>• Zoom in to see connections and labels</li>
            <li>• Use simulation to auto-layout</li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease, transform 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
  transform: scale(0.95);
}
</style>
