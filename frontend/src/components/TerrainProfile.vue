<script setup lang="ts">
import { ref, onMounted, watch, nextTick, computed } from 'vue';
import { useDroneStore } from '../store/drone';

const store = useDroneStore();
const canvas = ref<HTMLCanvasElement>();

const W = 800;
const H = 200;
const padding = { top: 20, right: 20, bottom: 30, left: 50 };
const plotW = W - padding.left - padding.right;
const plotH = H - padding.top - padding.bottom;

const distances = computed(() => {
  const profile = store.terrainProfile;
  if (profile.length < 2) return [];
  const dists: number[] = [0];
  for (let i = 1; i < profile.length; i++) {
    const dx = (profile[i].lng - profile[i - 1].lng) * 111000;
    const dy = (profile[i].lat - profile[i - 1].lat) * 111000;
    dists.push(dists[i - 1] + Math.sqrt(dx * dx + dy * dy));
  }
  return dists;
});

const maxDist = computed(() => distances.value[distances.value.length - 1] || 1);

const altRange = computed(() => {
  const profile = store.terrainProfile;
  if (profile.length < 2) return { min: 0, max: 1, range: 1 };
  let minAlt = Infinity;
  let maxAlt = -Infinity;
  for (const p of profile) {
    minAlt = Math.min(minAlt, p.terrainElevation);
    maxAlt = Math.max(maxAlt, p.altitude);
  }
  minAlt = Math.max(0, minAlt - 20);
  maxAlt = maxAlt + 20;
  return { min: minAlt, max: maxAlt, range: maxAlt - minAlt || 1 };
});

function toX(d: number) {
  return padding.left + (d / maxDist.value) * plotW;
}

function toY(a: number) {
  return padding.top + plotH - ((a - altRange.value.min) / altRange.value.range) * plotH;
}

function xToDistance(x: number): number {
  const d = ((x - padding.left) / plotW) * maxDist.value;
  return Math.max(0, Math.min(maxDist.value, d));
}

function findNearestIndex(distance: number): number {
  const dists = distances.value;
  if (dists.length === 0) return 0;
  let nearest = 0;
  let minDiff = Infinity;
  for (let i = 0; i < dists.length; i++) {
    const diff = Math.abs(dists[i] - distance);
    if (diff < minDiff) {
      minDiff = diff;
      nearest = i;
    }
  }
  return nearest;
}

function draw() {
  const ctx = canvas.value?.getContext('2d');
  if (!ctx) return;

  const profile = store.terrainProfile;
  const dists = distances.value;

  ctx.clearRect(0, 0, W, H);

  ctx.fillStyle = '#1e293b';
  ctx.fillRect(0, 0, W, H);

  if (profile.length < 2) {
    ctx.fillStyle = '#94a3b8';
    ctx.font = '14px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('暂无航线数据 — 规划航线后显示地形剖面', W / 2, H / 2);
    return;
  }

  ctx.beginPath();
  ctx.moveTo(toX(dists[0]), toY(profile[0].terrainElevation));
  for (let i = 1; i < profile.length; i++) {
    ctx.lineTo(toX(dists[i]), toY(profile[i].terrainElevation));
  }
  ctx.lineTo(toX(dists[dists.length - 1]), padding.top + plotH);
  ctx.lineTo(toX(0), padding.top + plotH);
  ctx.closePath();
  ctx.fillStyle = 'rgba(139,92,46,0.6)';
  ctx.fill();
  ctx.strokeStyle = '#92613a';
  ctx.lineWidth = 1;
  ctx.stroke();

  ctx.beginPath();
  ctx.moveTo(toX(dists[0]), toY(profile[0].terrainElevation));
  for (let i = 1; i < profile.length; i++) {
    ctx.lineTo(toX(dists[i]), toY(profile[i].terrainElevation));
  }
  for (let i = profile.length - 1; i >= 0; i--) {
    ctx.lineTo(toX(dists[i]), toY(profile[i].altitude));
  }
  ctx.closePath();
  ctx.fillStyle = 'rgba(34,197,94,0.1)';
  ctx.fill();

  ctx.beginPath();
  ctx.moveTo(toX(dists[0]), toY(profile[0].altitude));
  for (let i = 1; i < profile.length; i++) {
    ctx.lineTo(toX(dists[i]), toY(profile[i].altitude));
  }
  ctx.strokeStyle = '#3b82f6';
  ctx.lineWidth = 2;
  ctx.stroke();

  for (let i = 0; i < profile.length; i++) {
    const isHovered = store.hoveredProfileIndex === i;
    ctx.beginPath();
    ctx.arc(toX(dists[i]), toY(profile[i].altitude), isHovered ? 7 : 4, 0, Math.PI * 2);
    ctx.fillStyle = isHovered ? '#fbbf24' : '#60a5fa';
    ctx.fill();
    ctx.strokeStyle = isHovered ? '#f59e0b' : '#1d4ed8';
    ctx.lineWidth = isHovered ? 2 : 1;
    ctx.stroke();
  }

  ctx.strokeStyle = '#475569';
  ctx.lineWidth = 1;
  ctx.beginPath();
  ctx.moveTo(padding.left, padding.top);
  ctx.lineTo(padding.left, padding.top + plotH);
  ctx.lineTo(padding.left + plotW, padding.top + plotH);
  ctx.stroke();

  ctx.fillStyle = '#94a3b8';
  ctx.font = '11px sans-serif';
  ctx.textAlign = 'center';
  const xTicks = 5;
  for (let i = 0; i <= xTicks; i++) {
    const d = (maxDist.value / xTicks) * i;
    const x = toX(d);
    ctx.fillText(`${(d / 1000).toFixed(1)} km`, x, padding.top + plotH + 16);
    ctx.beginPath();
    ctx.moveTo(x, padding.top + plotH);
    ctx.lineTo(x, padding.top + plotH + 4);
    ctx.strokeStyle = '#475569';
    ctx.stroke();
  }

  ctx.textAlign = 'right';
  const yTicks = 4;
  for (let i = 0; i <= yTicks; i++) {
    const a = altRange.value.min + (altRange.value.range / yTicks) * i;
    const y = toY(a);
    ctx.fillText(`${Math.round(a)}m`, padding.left - 6, y + 4);
    ctx.beginPath();
    ctx.moveTo(padding.left - 3, y);
    ctx.lineTo(padding.left, y);
    ctx.strokeStyle = '#475569';
    ctx.stroke();
  }

  if (store.hoveredProfileIndex !== null && profile[store.hoveredProfileIndex]) {
    const idx = store.hoveredProfileIndex;
    const x = toX(dists[idx]);
    const yAlt = toY(profile[idx].altitude);

    ctx.beginPath();
    ctx.moveTo(x, padding.top);
    ctx.lineTo(x, padding.top + plotH);
    ctx.strokeStyle = 'rgba(251, 191, 36, 0.8)';
    ctx.lineWidth = 1.5;
    ctx.setLineDash([4, 4]);
    ctx.stroke();
    ctx.setLineDash([]);

    ctx.beginPath();
    ctx.arc(x, yAlt, 7, 0, Math.PI * 2);
    ctx.fillStyle = '#fbbf24';
    ctx.fill();
    ctx.strokeStyle = '#f59e0b';
    ctx.lineWidth = 2;
    ctx.stroke();

    ctx.textAlign = 'left';
    ctx.font = '11px sans-serif';
    const infoX = x + 10;
    const infoY = padding.top + 14;
    const infoText = [
      `飞行高度: ${profile[idx].altitude}m`,
      `地形高程: ${profile[idx].terrainElevation}m`,
      `距离: ${(dists[idx] / 1000).toFixed(2)} km`,
    ];

    const maxTextWidth = Math.max(...infoText.map(t => ctx.measureText(t).width));
    ctx.fillStyle = 'rgba(30, 41, 59, 0.9)';
    ctx.fillRect(infoX - 4, infoY - 12, maxTextWidth + 12, infoText.length * 14 + 8);
    ctx.strokeStyle = '#475569';
    ctx.lineWidth = 1;
    ctx.strokeRect(infoX - 4, infoY - 12, maxTextWidth + 12, infoText.length * 14 + 8);

    ctx.fillStyle = '#e2e8f0';
    infoText.forEach((text, i) => {
      ctx.fillText(text, infoX, infoY + i * 14);
    });
  }

  ctx.textAlign = 'left';
  ctx.fillStyle = '#3b82f6';
  ctx.fillRect(padding.left + 10, padding.top + 4, 12, 3);
  ctx.fillStyle = '#94a3b8';
  ctx.fillText('飞行高度', padding.left + 26, padding.top + 10);
  ctx.fillStyle = 'rgba(139,92,46,0.8)';
  ctx.fillRect(padding.left + 90, padding.top + 2, 12, 8);
  ctx.fillStyle = '#94a3b8';
  ctx.fillText('地形', padding.left + 106, padding.top + 10);
}

function handleMouseMove(e: MouseEvent) {
  const canvasEl = canvas.value;
  if (!canvasEl) return;

  const rect = canvasEl.getBoundingClientRect();
  const scaleX = W / rect.width;
  const x = (e.clientX - rect.left) * scaleX;

  if (x < padding.left || x > padding.left + plotW) {
    store.setHoveredProfileIndex(null);
    return;
  }

  const distance = xToDistance(x);
  const nearestIdx = findNearestIndex(distance);
  store.setHoveredProfileIndex(nearestIdx);
}

function handleMouseLeave() {
  store.setHoveredProfileIndex(null);
}

onMounted(() => nextTick(draw));
watch(() => store.terrainProfile, draw, { deep: true });
watch(() => store.hoveredProfileIndex, draw);
</script>

<template>
  <canvas
    ref="canvas"
    width="800"
    height="200"
    class="w-full rounded-lg border border-slate-700 cursor-crosshair"
    @mousemove="handleMouseMove"
    @mouseleave="handleMouseLeave"
  />
</template>
