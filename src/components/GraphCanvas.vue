<script setup lang="ts">
import { ref, shallowRef, onMounted, onUnmounted, watch, computed, nextTick, markRaw } from 'vue';
import * as d3 from 'd3';
import { 
  Search, ZoomIn, ZoomOut, Maximize, RefreshCw, Info, Upload, Layers,
  ChevronLeft, ChevronRight
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
const graphUrl = ref('https://i.gogingko.net/api/v1/v/exchange/forwards.txt');
const isCollapsed = ref(false);
const selectedGroup = ref<number | null>(null);
const groupViewMode = ref(false);
const posts = ref<any[]>([]);
const postsLoading = ref(false);
const showPostsWidget = ref(false);
const postsNodeName = ref('');

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

const fetchLatestPosts = async (nodeName: string) => {
  postsNodeName.value = nodeName;
  postsLoading.value = true;
  showPostsWidget.value = true;
  posts.value = [];
  
  try {
    const response = await fetch(`https://i.gogingko.net/api/v1/last/${nodeName}?n=10`);
    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
    const data = await response.json();
    posts.value = Array.isArray(data) ? data : (data.posts || []);
  } catch (error) {
    console.error('Failed to fetch posts:', error);
  } finally {
    postsLoading.value = false;
  }
};

// Label Propagation Algorithm
const runLabelPropagation = () => {
  if (data.value.nodes.length === 0) return;
  
  loading.value = true;
  const nodes = data.value.nodes;
  const links = data.value.links;
  
  // 1. Initialize labels
  const labels = new Map<string, number>();
  const adj = new Map<string, string[]>();
  
  nodes.forEach((node, i) => {
    labels.set(node.id, i);
    adj.set(node.id, []);
  });
  
  links.forEach(link => {
    const s = typeof link.source === 'string' ? link.source : (link.source as Node).id;
    const t = typeof link.target === 'string' ? link.target : (link.target as Node).id;
    adj.get(s)!.push(t);
    adj.get(t)!.push(s);
  });

  let changed = true;
  let iterations = 0;
  const maxIterations = 20;

  while (changed && iterations < maxIterations) {
    changed = false;
    iterations++;
    
    // Shuffle nodes
    const shuffledNodes = [...nodes].sort(() => Math.random() - 0.5);
    
    shuffledNodes.forEach(node => {
      const neighbors = adj.get(node.id)!;
      if (neighbors.length === 0) return;
      
      const labelCounts = new Map<number, number>();
      neighbors.forEach(neighborId => {
        const label = labels.get(neighborId)!;
        labelCounts.set(label, (labelCounts.get(label) || 0) + 1);
      });
      
      // Find most frequent label
      let maxCount = -1;
      let bestLabels: number[] = [];
      labelCounts.forEach((count, label) => {
        if (count > maxCount) {
          maxCount = count;
          bestLabels = [label];
        } else if (count === maxCount) {
          bestLabels.push(label);
        }
      });
      
      const chosenLabel = bestLabels[Math.floor(Math.random() * bestLabels.length)];
      if (labels.get(node.id) !== chosenLabel) {
        labels.set(node.id, chosenLabel);
        changed = true;
      }
    });
  }

  // Map labels to groups
  const uniqueLabels = Array.from(new Set(labels.values()));
  const labelToGroup = new Map<number, number>();
  uniqueLabels.forEach((label, i) => labelToGroup.set(label, i));

  nodes.forEach(node => {
    const groupIdx = labelToGroup.get(labels.get(node.id)!)!;
    node.group = groupIdx;
    node.color = COLORS[groupIdx % COLORS.length];
  });

  stats.value.groups = uniqueLabels.length;
  loading.value = false;
  
  if (simulationRef.value) {
    simulationRef.value.alpha(0.1).restart();
  }
};

// Louvain Method for Community Detection
const runLouvain = () => {
  if (data.value.nodes.length === 0) return;
  
  loading.value = true;
  const nodes = data.value.nodes;
  const links = data.value.links;
  const m = links.length;
  if (m === 0) {
    loading.value = false;
    return;
  }

  // 1. Initialize each node to its own community
  const nodeToCommunity = new Map<string, number>();
  const communityToNodes = new Map<number, Set<string>>();
  const nodeDegrees = new Map<string, number>();
  const adj = new Map<string, Map<string, number>>();

  nodes.forEach((node, i) => {
    nodeToCommunity.set(node.id, i);
    communityToNodes.set(i, new Set([node.id]));
    nodeDegrees.set(node.id, 0);
    adj.set(node.id, new Map());
  });

  links.forEach(link => {
    const s = typeof link.source === 'string' ? link.source : (link.source as Node).id;
    const t = typeof link.target === 'string' ? link.target : (link.target as Node).id;
    
    nodeDegrees.set(s, (nodeDegrees.get(s) || 0) + 1);
    nodeDegrees.set(t, (nodeDegrees.get(t) || 0) + 1);
    
    adj.get(s)!.set(t, (adj.get(s)!.get(t) || 0) + 1);
    adj.get(t)!.set(s, (adj.get(t)!.get(s) || 0) + 1);
  });

  const communityWeights = new Map<number, number>(); // sum_tot
  communityToNodes.forEach((nodesInComm, commId) => {
    let weight = 0;
    nodesInComm.forEach(nodeId => {
      weight += nodeDegrees.get(nodeId) || 0;
    });
    communityWeights.set(commId, weight);
  });

  let changed = true;
  let iterations = 0;
  const maxIterations = 10;

  while (changed && iterations < maxIterations) {
    changed = false;
    iterations++;

    // Shuffle nodes for better convergence
    const shuffledNodes = [...nodes].sort(() => Math.random() - 0.5);

    shuffledNodes.forEach(node => {
      const nodeId = node.id;
      const currentCommId = nodeToCommunity.get(nodeId)!;
      const ki = nodeDegrees.get(nodeId)!;

      // Find neighboring communities
      const neighborComms = new Map<number, number>(); // commId -> k_i,in
      const neighbors = adj.get(nodeId)!;
      neighbors.forEach((weight, neighborId) => {
        const neighborCommId = nodeToCommunity.get(neighborId)!;
        neighborComms.set(neighborCommId, (neighborComms.get(neighborCommId) || 0) + weight);
      });

      // Calculate modularity gain for each neighbor community
      let bestCommId = currentCommId;
      let maxDeltaQ = 0;

      // Remove node from current community temporarily for calculation
      const currentSumTot = communityWeights.get(currentCommId)! - ki;

      neighborComms.forEach((kiin, commId) => {
        const sumTot = commId === currentCommId ? currentSumTot : communityWeights.get(commId)!;
        
        // Delta Q formula (simplified for weight=1)
        const deltaQ = kiin - (sumTot * ki) / (2 * m);
        
        if (deltaQ > maxDeltaQ) {
          maxDeltaQ = deltaQ;
          bestCommId = commId;
        }
      });

      if (bestCommId !== currentCommId) {
        // Move node
        communityToNodes.get(currentCommId)!.delete(nodeId);
        if (communityToNodes.get(currentCommId)!.size === 0) {
          communityToNodes.delete(currentCommId);
          communityWeights.delete(currentCommId);
        } else {
          communityWeights.set(currentCommId, communityWeights.get(currentCommId)! - ki);
        }

        if (!communityToNodes.has(bestCommId)) {
          communityToNodes.set(bestCommId, new Set());
          communityWeights.set(bestCommId, 0);
        }
        communityToNodes.get(bestCommId)!.add(nodeId);
        communityWeights.set(bestCommId, communityWeights.get(bestCommId)! + ki);
        nodeToCommunity.set(nodeId, bestCommId);
        
        changed = true;
      }
    });
  }

  // Map final communities to groups and colors
  const finalComms = Array.from(communityToNodes.keys());
  const commToGroupIdx = new Map<number, number>();
  finalComms.forEach((commId, i) => commToGroupIdx.set(commId, i));

  nodes.forEach(node => {
    const groupIdx = commToGroupIdx.get(nodeToCommunity.get(node.id)!)!;
    node.group = groupIdx;
    node.color = COLORS[groupIdx % COLORS.length];
  });

  stats.value.groups = finalComms.length;
  loading.value = false;
  
  if (simulationRef.value) {
    simulationRef.value.alpha(0.1).restart();
  }
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

// Zoom to a specific group
const zoomToGroup = (groupId: number) => {
  const groupNodes = data.value.nodes.filter(n => n.group === groupId);
  if (groupNodes.length === 0) return;

  const xMin = d3.min(groupNodes, d => d.x!)!;
  const xMax = d3.max(groupNodes, d => d.x!)!;
  const yMin = d3.min(groupNodes, d => d.y!)!;
  const yMax = d3.max(groupNodes, d => d.y!)!;

  const width = dimensions.value.width;
  const height = dimensions.value.height;
  const dx = xMax - xMin;
  const dy = yMax - yMin;
  const x = (xMin + xMax) / 2;
  const y = (yMin + yMax) / 2;
  
  // Calculate scale to fit the group with some padding
  const padding = 40;
  const scale = Math.max(0.5, Math.min(8, 0.8 / Math.max(dx / width, dy / height)));
  
  if (zoomRef.value && canvasRef.value) {
    d3.select(canvasRef.value)
      .transition()
      .duration(750)
      .call(zoomRef.value.transform, d3.zoomIdentity.translate(width / 2 - scale * x, height / 2 - scale * y).scale(scale));
  }
};

// Render function
const render = (ctx: CanvasRenderingContext2D, width: number, height: number, k: number, tx: number, ty: number) => {
  const currentData = data.value;
  const currentSelectedNode = selectedNode.value;
  const currentHoveredNode = hoveredNode.value;
  const currentGroupViewMode = groupViewMode.value;
  const currentSelectedGroup = selectedGroup.value;
  
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

  // Helper to determine if a node/link should be dimmed
  const isNodeDimmed = (node: Node) => {
    if (currentGroupViewMode && currentSelectedGroup !== null) {
      return node.group !== currentSelectedGroup;
    }
    if (currentSelectedNode) {
      return !neighborIds.has(node.id);
    }
    return false;
  };

  const isLinkDimmed = (s: Node, t: Node) => {
    if (currentGroupViewMode && currentSelectedGroup !== null) {
      return s.group !== currentSelectedGroup || t.group !== currentSelectedGroup;
    }
    if (currentSelectedNode) {
      return s.id !== currentSelectedNode.id && t.id !== currentSelectedNode.id;
    }
    return false;
  };

  ctx.beginPath();
  ctx.strokeStyle = (currentSelectedNode || (currentGroupViewMode && currentSelectedGroup !== null)) 
    ? 'rgba(200, 200, 200, 0.05)' 
    : 'rgba(200, 200, 200, 0.15)';
  ctx.lineWidth = 0.5 / k;
  
  if (k > 0.4) {
    currentData.links.forEach(link => {
      const s = link.source as Node;
      const t = link.target as Node;
      if (isLinkDimmed(s, t)) return;
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
    const dimmed = isNodeDimmed(node);
    ctx.beginPath();
    if (!dimmed) {
      ctx.fillStyle = node.color;
      ctx.globalAlpha = 1;
    } else {
      ctx.fillStyle = '#e2e8f0';
      ctx.globalAlpha = 0.2;
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
      const dimmed = isNodeDimmed(node);
      if (!dimmed) {
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
  // Load default graph
  handleUrlLoad();
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
    const numGroups = stats.value.groups;
    const radius = Math.min(width, height) * 0.3; // Spread groups in a circle
    for (let i = 0; i < numGroups; i++) {
      const angle = (i / numGroups) * 2 * Math.PI;
      groupCenters.set(i, {
        x: width / 2 + radius * Math.cos(angle),
        y: height / 2 + radius * Math.sin(angle)
      });
    }
  }

  if (simulationRef.value) simulationRef.value.stop();

  const simulation = markRaw(d3.forceSimulation<Node>(data.value.nodes)
    .force('link', d3.forceLink<Node, Link>(data.value.links).id(d => d.id).distance(30).strength(0.1))
    .force('charge', d3.forceManyBody().strength(-60).distanceMax(500))
    .force('center', d3.forceCenter(width / 2, height / 2))
    .force('collide', d3.forceCollide().radius(d => (d as Node).radius + 4))
    .force('x', d3.forceX(node => {
      const n = node as Node;
      return groupCenters.get(n.group)?.x || width / 2;
    }).strength(() => stats.value.groups > 1 ? 0.15 : 0))
    .force('y', d3.forceY(node => {
      const n = node as Node;
      return groupCenters.get(n.group)?.y || height / 2;
    }).strength(() => stats.value.groups > 1 ? 0.15 : 0))
    .alphaDecay(0.02));

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

const handleContextMenu = (event: MouseEvent) => {
  event.preventDefault();
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
  
  if (found) {
    fetchLatestPosts(found.label);
  }
};

// Re-render when transform, hoveredNode, selectedNode, or groupViewMode changes
watch([transform, hoveredNode, selectedNode, dimensions, groupViewMode, selectedGroup], () => {
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
      @contextmenu="handleContextMenu"
    />

    <!-- Empty State -->
    <div v-if="!loading && data.nodes.length === 0" class="absolute inset-0 flex flex-col items-center justify-center text-slate-400 pointer-events-none">
      <Upload :size="48" class="mb-4 opacity-20" />
      <p class="text-lg font-medium">No graph data loaded</p>
      <p class="text-sm">Upload a file to start exploring</p>
    </div>

    <!-- UI Overlay -->
    <div class="absolute top-6 left-6 flex flex-col gap-4 pointer-events-none">
      <div 
        class="bg-white/90 backdrop-blur-md rounded-2xl shadow-xl border border-slate-200 pointer-events-auto transition-all duration-300 ease-in-out overflow-hidden"
        :class="isCollapsed ? 'w-12 h-12 p-2' : 'w-80 p-6'"
      >
        <div class="flex items-center justify-between" :class="{ 'mb-1': !isCollapsed }">
          <h1 v-if="!isCollapsed" class="text-2xl font-bold text-slate-900 truncate">Huge Graph</h1>
          <button 
            @click="isCollapsed = !isCollapsed"
            class="p-1.5 hover:bg-slate-100 rounded-lg text-slate-400 transition-colors"
            :title="isCollapsed ? 'Expand' : 'Collapse'"
          >
            <ChevronRight v-if="isCollapsed" :size="20" />
            <ChevronLeft v-else :size="20" />
          </button>
        </div>
        
        <div v-if="!isCollapsed" class="mt-1">
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
                Community Detection
              </label>
            </div>
            <div class="grid grid-cols-1 gap-2">
              <button 
                @click="runLouvain"
                :disabled="data.nodes.length === 0"
                class="w-full flex items-center justify-center gap-2 py-2 bg-indigo-600 hover:bg-indigo-700 disabled:bg-slate-300 text-white rounded-xl transition-all text-xs font-medium"
                title="Louvain Modularity Optimization"
              >
                <RefreshCw :size="12" :class="{ 'animate-spin': loading }" />
                Louvain Method
              </button>
              <button 
                @click="runLabelPropagation"
                :disabled="data.nodes.length === 0"
                class="w-full flex items-center justify-center gap-2 py-2 bg-blue-600 hover:bg-blue-700 disabled:bg-slate-300 text-white rounded-xl transition-all text-xs font-medium"
                title="Fast Label Propagation"
              >
                <RefreshCw :size="12" :class="{ 'animate-spin': loading }" />
                Label Propagation
              </button>
            </div>
          </div>

          <div class="pt-4 border-t border-slate-100">
            <div class="flex items-center justify-between mb-2">
              <label class="text-xs font-semibold text-slate-400 uppercase tracking-wider">
                Group View Mode
              </label>
              <button 
                @click="groupViewMode = !groupViewMode"
                class="relative inline-flex h-5 w-9 items-center rounded-full transition-colors focus:outline-none"
                :class="groupViewMode ? 'bg-blue-600' : 'bg-slate-200'"
              >
                <span 
                  class="inline-block h-3 w-3 transform rounded-full bg-white transition-transform"
                  :class="groupViewMode ? 'translate-x-5' : 'translate-x-1'"
                />
              </button>
            </div>
            
            <div v-if="groupViewMode" class="space-y-2 animate-in fade-in slide-in-from-top-1 duration-200">
              <select 
                v-model="selectedGroup"
                class="w-full px-3 py-2 bg-slate-50 border border-slate-200 rounded-xl text-xs focus:outline-none focus:ring-2 focus:ring-blue-500 transition-all pointer-events-auto"
              >
                <option :value="null">Select a group...</option>
                <option v-for="i in stats.groups" :key="i-1" :value="i-1">
                  Community {{ i-1 }} ({{ data.nodes.filter(n => n.group === i-1).length }} nodes)
                </option>
              </select>
              
              <button 
                v-if="selectedGroup !== null"
                @click="zoomToGroup(selectedGroup)"
                class="w-full flex items-center justify-center gap-2 py-2 bg-slate-800 hover:bg-slate-900 text-white rounded-xl transition-all text-xs font-medium"
              >
                <Maximize :size="12" />
                Focus on Group
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
            <li>• Right-click node for latest posts</li>
            <li>• Zoom in to see connections and labels</li>
            <li>• Use simulation to auto-layout</li>
          </ul>
        </div>
      </div>
    </div>

    <!-- Latest Posts Widget -->
    <Transition name="slide-right">
      <div v-if="showPostsWidget" class="absolute top-8 right-8 bottom-8 w-96 bg-white/95 backdrop-blur-md rounded-3xl shadow-2xl border border-slate-200 flex flex-col overflow-hidden z-40 pointer-events-auto">
        <div class="p-6 border-b border-slate-100 flex items-center justify-between bg-slate-50/50">
          <div>
            <h3 class="font-bold text-slate-900">Latest Posts</h3>
            <p class="text-xs text-slate-500">Channel: <span class="font-mono text-blue-600 font-semibold">{{ postsNodeName }}</span></p>
          </div>
          <button @click="showPostsWidget = false" class="p-2 hover:bg-slate-200 rounded-full transition-colors text-slate-400 hover:text-slate-600">
            <ChevronRight :size="20" />
          </button>
        </div>

        <div class="flex-1 overflow-y-auto p-4 space-y-4 custom-scrollbar">
          <div v-if="postsLoading" class="flex flex-col items-center justify-center h-64 space-y-4">
            <div class="w-8 h-8 border-3 border-blue-600 border-t-transparent rounded-full animate-spin" />
            <p class="text-xs text-slate-400 animate-pulse">Fetching latest updates...</p>
          </div>

          <div v-else-if="posts.length === 0" class="flex flex-col items-center justify-center h-64 text-center p-8">
            <div class="w-12 h-12 bg-slate-100 rounded-full flex items-center justify-center mb-4">
              <Info :size="24" class="text-slate-300" />
            </div>
            <p class="text-sm text-slate-500 font-medium">No posts found</p>
            <p class="text-xs text-slate-400 mt-1">This channel might be empty or restricted.</p>
          </div>

          <div v-for="(post, idx) in posts" :key="idx" class="bg-white border border-slate-100 rounded-2xl p-4 shadow-sm hover:shadow-md transition-all border-l-4 border-l-blue-500 overflow-hidden">
            <!-- Metadata Badges -->
            <div class="flex flex-wrap gap-2 mb-3">
              <span class="px-2 py-0.5 bg-slate-100 text-slate-600 rounded text-[10px] font-mono" title="Post ID">
                ID: {{ post.key }}
              </span>
              <span v-if="post.data?.date" class="px-2 py-0.5 bg-blue-50 text-blue-600 rounded text-[10px] font-medium">
                {{ post.data.date }}
              </span>
              <span v-if="post.data?.user" class="px-2 py-0.5 bg-emerald-50 text-emerald-600 rounded text-[10px] font-medium">
                @{{ post.data.user }}
              </span>
              <span v-if="post.data?.views" class="px-2 py-0.5 bg-slate-100 text-slate-500 rounded text-[10px] font-medium">
                {{ post.data.views }} views
              </span>
            </div>
            
            <!-- Photo -->
            <div v-if="post.data?.photos" class="mb-3 rounded-xl overflow-hidden border border-slate-100 bg-slate-50">
              <img 
                :src="`https://i.gogingko.net/api/v1/v/telegram-photo/${post.key.split('/').pop()}_0`" 
                class="w-full h-auto max-h-64 object-cover hover:scale-105 transition-transform duration-500"
                referrerpolicy="no-referrer"
                loading="lazy"
              />
            </div>

            <!-- Video -->
            <div v-if="post.data?.videos" class="mb-3 rounded-xl overflow-hidden border border-slate-100 bg-slate-900">
              <video 
                :src="`https://i.gogingko.net/api/v1/v/telegram-video/${post.key.split('/').pop()}-0`" 
                class="w-full h-auto max-h-64"
                controls
                preload="metadata"
              ></video>
            </div>

            <!-- Content -->
            <div class="space-y-3">
              <div v-if="post.data?.author" class="text-[10px] font-bold text-slate-400 uppercase tracking-wider">
                Author: {{ post.data.author }}
              </div>
              
              <div v-if="post.data?.content" class="text-xs text-slate-700 whitespace-pre-wrap break-words leading-relaxed">
                {{ post.data.content }}
              </div>

              <div v-if="post.data?.linkPreview" class="mt-2 p-2 bg-slate-50 rounded-lg border border-slate-100 text-[10px] text-slate-500 italic">
                <span class="font-bold not-italic block mb-1 text-slate-400">Link Preview:</span>
                {{ post.data.linkPreview }}
              </div>
            </div>
          </div>
        </div>
        
        <div class="p-4 bg-slate-50 border-t border-slate-100 text-[10px] text-slate-400 text-center">
          Showing last 10 entries from Gingko API
        </div>
      </div>
    </Transition>
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

.slide-right-enter-active,
.slide-right-leave-active {
  transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
}

.slide-right-enter-from,
.slide-right-leave-to {
  opacity: 0;
  transform: translateX(100px);
}

.custom-scrollbar::-webkit-scrollbar {
  width: 4px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: transparent;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background: #e2e8f0;
  border-radius: 10px;
}
.custom-scrollbar::-webkit-scrollbar-thumb:hover {
  background: #cbd5e1;
}
</style>
