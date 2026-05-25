<template>
  <div class="relative w-full h-full absolute inset-0">
    <vc-viewer 
      @ready="onViewerReady" 
      :camera="camera" 
      @left-click="onMapClick"
      @mouse-move="onViewerMouseMove"
      @left-up="endWaypointDrag"
      :animation="false"
      :timeline="false"
      :base-layer-picker="false"
    >
      <vc-navigation></vc-navigation>
      
      <!-- Waypoints (User Clicks) -->
      <vc-entity
        v-for="(wp, index) in waypoints"
        :key="'wp-' + index"
        :id="'wp-' + index"
        :position="{ lng: wp.lng, lat: wp.lat, height: wp.height }"
        :point="pointGraphics"
        :enable-mouse-event="isWaypointEditable"
        :label="{
          text: (index + 1).toString(),
          font: '14px sans-serif',
          pixelOffset: [0, -20],
          horizontalOrigin: 0,
          verticalOrigin: 1,
          fillColor: 'white',
          outlineColor: 'black',
          outlineWidth: 2
        }"
        @mouseover="onWaypointMouseOver(index, $event)"
        @mouseout="onWaypointMouseOut(index)"
        @mousedown="onWaypointMouseDown(index, $event)"
      >
      </vc-entity>

      <!-- Polygon Area (Patrol Mode) -->
      <vc-entity v-if="isPatrolMode && waypoints.length > 2">
        <vc-graphics-polygon
          :hierarchy="waypointPositions"
          :material="'rgba(52, 152, 219, 0.3)'"
          :outline="true"
          :outlineColor="'#3498db'"
        ></vc-graphics-polygon>
      </vc-entity>

      <!-- Connection Lines (Normal Mode) -->
      <vc-entity v-else-if="!isPatrolMode && waypointPositions.length > 1" ref="routeLineEntityRef">
        <vc-graphics-polyline
          :positions="routeLinePositions"
          :material="'#3498db'"
          :width="3"
        ></vc-graphics-polyline>
      </vc-entity>

      <!-- Generated Scan Path (S-Shape) -->
      <vc-entity v-if="isPatrolMode && scanPath.length > 1">
        <vc-graphics-polyline
          :positions="scanPathPositions"
          :material="'#f1c40f'"
          :width="4"
        ></vc-graphics-polyline>
      </vc-entity>
    </vc-viewer>

    <div
      v-if="showWaypointHint"
      class="waypoint-drag-hint"
      :style="waypointHintStyle"
    >
      <div class="hint-row">
        <span class="mouse-icon" aria-hidden="true"></span>
        <span>拖拽移动水平位置</span>
      </div>
      <div class="hint-row">
        <span class="keycap">Alt</span>
        <span>+</span>
        <span class="mouse-icon" aria-hidden="true"></span>
        <span>上下移动高度</span>
      </div>
    </div>
    
    <!-- Coordinate System Info -->
    <div class="absolute bottom-5 right-5 bg-white/95 p-3.5 rounded-lg shadow-md z-10 backdrop-blur-sm">
      <div class="flex items-center gap-2 mb-1">
        <span class="text-xs text-gray-500 font-medium">坐标系统</span>
        <span class="text-sm font-bold text-green-500 bg-green-50 py-1 px-2.5 rounded font-mono">WGS84</span>
      </div>
      <div class="text-[11px] text-green-600 font-medium">✓ 大疆航线标准坐标</div>
    </div>
  </div>
</template>

<script setup>
import { ref, shallowRef, computed, watch, onMounted, onBeforeUnmount, markRaw } from 'vue';

const props = defineProps({
  waypoints: {
    type: Array,
    default: () => []
  },
  isClosedLoop: {
    type: Boolean,
    default: false
  },
  isPatrolMode: {
    type: Boolean,
    default: false
  },
  isWaypointEditable: {
    type: Boolean,
    default: false
  },
  scanPath: {
    type: Array,
    default: () => []
  }
});

watch(() => props.scanPath, (val) => {
  console.log('MapViewer received scanPath:', val.length);
}, { deep: true });

watch(() => props.isPatrolMode, (val) => {
  console.log('MapViewer isPatrolMode:', val);
});

const emit = defineEmits(['map-click', 'update:waypoints']);

const camera = ref({
  position: {
    lng: 104.39, 
    lat: 31.09,
    height: 1000
  },
  heading: 360,
  pitch: -90,
  roll: 0
});

const pointGraphics = {
  pixelSize: 10,
  color: 'red',
  outlineColor: 'white',
  outlineWidth: 2
};

let cesiumInstance = null;
let viewerInstance = null;
let previousCameraControls = null;
let pendingDragWindowPosition = null;
let dragAnimationFrame = null;
let suppressNextMapClick = false;
let suppressMapClickTimer = null;

const routeLineEntityRef = ref(null);
const routeLinePositionsProperty = shallowRef(null);
const hoveredWaypointIndex = ref(null);
const waypointHintPosition = ref({ x: 0, y: 0 });
const dragState = ref(null);
const isAltPressed = ref(false);

const showWaypointHint = computed(() => {
  return props.isWaypointEditable && hoveredWaypointIndex.value !== null && !dragState.value;
});

const waypointHintStyle = computed(() => ({
  left: `${waypointHintPosition.value.x}px`,
  top: `${waypointHintPosition.value.y}px`
}));

const waypointPositions = computed(() => {
  // Waypoints are stored in GCJ-02, display directly
  const positions = props.waypoints.map(wp => ({ lng: wp.lng, lat: wp.lat, height: wp.height }));
  // Normal mode closed loop logic
  if (!props.isPatrolMode && props.isClosedLoop && positions.length >= 2) {
    positions.push(positions[0]);
  }
  return positions;
});

const scanPathPositions = computed(() => {
  // Scan path is in GCJ-02, display directly
  return props.scanPath.map(wp => ({ lng: wp.lng, lat: wp.lat, height: wp.height }));
});

const routeLinePositions = computed(() => routeLinePositionsProperty.value || waypointPositions.value);

const getRouteLineCartesianPositions = () => {
  if (!cesiumInstance) return [];

  const positions = props.waypoints.map(wp => (
    cesiumInstance.Cartesian3.fromDegrees(wp.lng, wp.lat, wp.height || 0)
  ));

  if (!props.isPatrolMode && props.isClosedLoop && positions.length >= 2) {
    positions.push(positions[0]);
  }

  return positions;
};

const getWindowPosition = (event) => {
  const position = event?.windowPosition || event?.position || event?.endPosition;
  if (!position) return null;
  return {
    x: position.x,
    y: position.y
  };
};

const toCartesian2 = (windowPosition) => {
  if (!cesiumInstance || !windowPosition) return windowPosition;
  return new cesiumInstance.Cartesian2(windowPosition.x, windowPosition.y);
};

const getNativeWindowPosition = (event) => {
  const canvas = viewerInstance?.canvas;
  if (!canvas) return null;

  const rect = canvas.getBoundingClientRect();
  return {
    x: event.clientX - rect.left,
    y: event.clientY - rect.top
  };
};

const setCanvasCursor = (cursor = '') => {
  if (viewerInstance?.canvas) {
    viewerInstance.canvas.style.cursor = cursor;
  }
};

const updateHintPosition = (windowPosition) => {
  const canvas = viewerInstance?.canvas;
  const hintWidth = 180;
  const hintHeight = 56;
  const padding = 12;
  let x = windowPosition.x + 14;
  let y = windowPosition.y + 14;

  if (canvas) {
    x = Math.min(x, canvas.clientWidth - hintWidth - padding);
    y = Math.min(y, canvas.clientHeight - hintHeight - padding);
    x = Math.max(padding, x);
    y = Math.max(padding, y);
  }

  waypointHintPosition.value = { x, y };
};

const setCameraControlsEnabled = (enabled) => {
  const controller = viewerInstance?.scene?.screenSpaceCameraController;
  if (!controller) return;

  if (!enabled) {
    if (!previousCameraControls) {
      previousCameraControls = {
        enableRotate: controller.enableRotate,
        enableTranslate: controller.enableTranslate,
        enableZoom: controller.enableZoom,
        enableTilt: controller.enableTilt,
        enableLook: controller.enableLook
      };
    }

    controller.enableRotate = false;
    controller.enableTranslate = false;
    controller.enableZoom = false;
    controller.enableTilt = false;
    controller.enableLook = false;
    return;
  }

  if (previousCameraControls) {
    controller.enableRotate = previousCameraControls.enableRotate;
    controller.enableTranslate = previousCameraControls.enableTranslate;
    controller.enableZoom = previousCameraControls.enableZoom;
    controller.enableTilt = previousCameraControls.enableTilt;
    controller.enableLook = previousCameraControls.enableLook;
    previousCameraControls = null;
  }
};

const queueMapClickSuppress = (duration = 350) => {
  suppressNextMapClick = true;
  if (suppressMapClickTimer) {
    clearTimeout(suppressMapClickTimer);
    suppressMapClickTimer = null;
  }

  if (duration <= 0) return;

  suppressMapClickTimer = window.setTimeout(() => {
    suppressNextMapClick = false;
    suppressMapClickTimer = null;
  }, duration);
};

const clearMapClickSuppress = () => {
  suppressNextMapClick = false;
  if (suppressMapClickTimer) {
    clearTimeout(suppressMapClickTimer);
    suppressMapClickTimer = null;
  }
};

const consumeMapClickSuppress = () => {
  if (!suppressNextMapClick) return false;

  clearMapClickSuppress();
  return true;
};

const isPickedWaypoint = (windowPosition) => {
  if (!props.isWaypointEditable || !viewerInstance || !cesiumInstance) return false;

  const pickedFeature = viewerInstance.scene.pick(toCartesian2(windowPosition));
  return cesiumInstance.defined(pickedFeature?.id?.point);
};

const getWaypointCanvasPosition = (waypoint) => {
  if (!viewerInstance || !cesiumInstance || !waypoint) return null;

  const position = cesiumInstance.Cartesian3.fromDegrees(
    waypoint.lng,
    waypoint.lat,
    waypoint.height || 0
  );
  const canvasPosition = viewerInstance.scene.cartesianToCanvasCoordinates(position);
  if (!canvasPosition) return null;

  return {
    x: canvasPosition.x,
    y: canvasPosition.y
  };
};

const getDragTargetWindowPosition = (windowPosition, state) => {
  if (!state?.grabOffset) return windowPosition;

  return {
    x: windowPosition.x - state.grabOffset.x,
    y: windowPosition.y - state.grabOffset.y
  };
};

const pickHeightSurfaceCoordinates = (windowPosition, height = 0) => {
  if (!viewerInstance || !cesiumInstance) return null;

  const ray = viewerInstance.camera.getPickRay(toCartesian2(windowPosition));
  if (!ray) return null;

  const ellipsoid = cesiumInstance.Ellipsoid.WGS84;
  const safeHeight = Number.isFinite(Number(height)) ? Math.max(0, Number(height)) : 0;
  const pickEllipsoid = safeHeight > 0
    ? new cesiumInstance.Ellipsoid(
      ellipsoid.radii.x + safeHeight,
      ellipsoid.radii.y + safeHeight,
      ellipsoid.radii.z + safeHeight
    )
    : ellipsoid;

  const intersection = cesiumInstance.IntersectionTests.rayEllipsoid(ray, pickEllipsoid);
  const distance = intersection
    ? (intersection.start >= 0 ? intersection.start : intersection.stop)
    : undefined;
  const cartesian3 = distance !== undefined && distance >= 0
    ? cesiumInstance.Ray.getPoint(ray, distance)
    : viewerInstance.scene.globe.pick(ray, viewerInstance.scene);
  if (!cartesian3) return null;

  const cartographic = cesiumInstance.Cartographic.fromCartesian(cartesian3);
  if (!cartographic) return null;

  return {
    lng: Number(cesiumInstance.Math.toDegrees(cartographic.longitude).toFixed(7)),
    lat: Number(cesiumInstance.Math.toDegrees(cartographic.latitude).toFixed(7))
  };
};

const emitWaypointUpdate = (index, nextWaypoint) => {
  if (!props.waypoints[index]) return;

  const newWaypoints = [...props.waypoints];
  newWaypoints[index] = {
    ...newWaypoints[index],
    ...nextWaypoint
  };
  emit('update:waypoints', newWaypoints);
  requestRender();
};

const requestRender = () => {
  viewerInstance?.scene?.requestRender?.();
};

const applyDragUpdate = (windowPosition) => {
  const state = dragState.value;
  if (!state || !props.waypoints[state.index]) {
    endWaypointDrag();
    return;
  }

  const dx = windowPosition.x - state.startWindowPosition.x;
  const dy = windowPosition.y - state.startWindowPosition.y;
  if (Math.abs(dx) > 2 || Math.abs(dy) > 2) {
    state.hasMoved = true;
  }

  if (state.mode === 'height') {
    const startHeight = Number.isFinite(Number(state.startWaypoint.height))
      ? Number(state.startWaypoint.height)
      : 0;
    const nextHeight = Math.max(0, Number((startHeight - dy).toFixed(1)));

    emitWaypointUpdate(state.index, {
      lat: state.startWaypoint.lat,
      lng: state.startWaypoint.lng,
      height: nextHeight
    });
    return;
  }

  const targetWindowPosition = getDragTargetWindowPosition(windowPosition, state);
  const coords = pickHeightSurfaceCoordinates(targetWindowPosition, state.startWaypoint.height);
  if (!coords) return;

  emitWaypointUpdate(state.index, {
    lat: coords.lat,
    lng: coords.lng,
    height: state.startWaypoint.height
  });
};

const flushPendingDragUpdate = () => {
  if (dragAnimationFrame) {
    cancelAnimationFrame(dragAnimationFrame);
    dragAnimationFrame = null;
  }

  if (pendingDragWindowPosition) {
    const nextPosition = pendingDragWindowPosition;
    pendingDragWindowPosition = null;
    applyDragUpdate(nextPosition);
  }
};

const queueDragUpdate = (windowPosition) => {
  pendingDragWindowPosition = windowPosition;

  if (dragAnimationFrame) return;

  dragAnimationFrame = requestAnimationFrame(() => {
    dragAnimationFrame = null;
    if (!pendingDragWindowPosition) return;

    const nextPosition = pendingDragWindowPosition;
    pendingDragWindowPosition = null;
    applyDragUpdate(nextPosition);
  });
};

const clearHoverState = () => {
  hoveredWaypointIndex.value = null;
  if (!dragState.value) {
    setCanvasCursor('');
  }
};

const onViewerReady = ({ Cesium, viewer }) => {
  console.log('Viewer ready');
  cesiumInstance = Cesium;
  viewerInstance = viewer;
  routeLinePositionsProperty.value = markRaw(new Cesium.CallbackProperty(getRouteLineCartesianPositions, false));
  bindNativeDragEvents();
  
  viewer.imageryLayers.removeAll();
  
  const amapProvider = new Cesium.UrlTemplateImageryProvider({
    url: 'https://webst02.is.autonavi.com/appmaptile?style=6&x={x}&y={y}&z={z}',
    minimumLevel: 3,
    maximumLevel: 18
  });
  
  viewer.imageryLayers.addImageryProvider(amapProvider);
};

const onWaypointMouseOver = (index, event) => {
  if (!props.isWaypointEditable || dragState.value) return;

  hoveredWaypointIndex.value = index;
  setCanvasCursor('move');

  const windowPosition = getWindowPosition(event);
  if (windowPosition) {
    updateHintPosition(windowPosition);
  }
};

const onWaypointMouseOut = (index) => {
  if (dragState.value || hoveredWaypointIndex.value !== index) return;
  clearHoverState();
};

const onWaypointMouseDown = (index, event) => {
  if (dragState.value || !props.isWaypointEditable || (event?.button !== undefined && event.button !== 0)) return;

  const waypoint = props.waypoints[index];
  const windowPosition = getWindowPosition(event);
  if (!waypoint || !windowPosition) return;

  startWaypointDrag(index, windowPosition, isAltPressed.value);
};

const startWaypointDrag = (index, windowPosition, altPressed) => {
  const waypoint = props.waypoints[index];
  if (!props.isWaypointEditable || !waypoint || !windowPosition) return;

  const waypointCanvasPosition = getWaypointCanvasPosition(waypoint);
  const grabOffset = waypointCanvasPosition
    ? {
      x: windowPosition.x - waypointCanvasPosition.x,
      y: windowPosition.y - waypointCanvasPosition.y
    }
    : { x: 0, y: 0 };

  dragState.value = {
    index,
    mode: altPressed ? 'height' : 'position',
    startWindowPosition: windowPosition,
    startWaypoint: { ...waypoint },
    grabOffset,
    hasMoved: false
  };

  hoveredWaypointIndex.value = index;
  updateHintPosition(windowPosition);
  setCanvasCursor('move');
  setCameraControlsEnabled(false);
  queueMapClickSuppress(0);
};

const onViewerMouseMove = (event) => {
  const windowPosition = getWindowPosition(event);
  if (!windowPosition) return;

  if (dragState.value) {
    queueDragUpdate(windowPosition);
    return;
  }

  if (showWaypointHint.value) {
    updateHintPosition(windowPosition);
  }
};

const findWaypointIndexAtWindowPosition = (windowPosition) => {
  if (!props.isWaypointEditable || !viewerInstance || !cesiumInstance) return -1;

  const pickedFeature = viewerInstance.scene.pick(toCartesian2(windowPosition));
  const pickedEntity = pickedFeature?.id;
  if (!cesiumInstance.defined(pickedEntity?.point)) return -1;

  const indexFromEntity = props.waypoints.findIndex((_, index) => {
    return pickedEntity.id === `wp-${index}`;
  });
  if (indexFromEntity !== -1) return indexFromEntity;

  return props.waypoints.findIndex((wp) => {
    const position = pickedEntity.position?.getValue?.(viewerInstance.clock.currentTime);
    if (!position) return false;

    const cartographic = cesiumInstance.Cartographic.fromCartesian(position);
    const lng = Number(cesiumInstance.Math.toDegrees(cartographic.longitude).toFixed(7));
    const lat = Number(cesiumInstance.Math.toDegrees(cartographic.latitude).toFixed(7));
    return Math.abs(wp.lng - lng) < 0.0000001 && Math.abs(wp.lat - lat) < 0.0000001;
  });
};

const onNativePointerDown = (event) => {
  if (!props.isWaypointEditable || event.button !== 0 || dragState.value) return;

  const windowPosition = getNativeWindowPosition(event);
  if (!windowPosition) return;

  const index = findWaypointIndexAtWindowPosition(windowPosition);
  if (index === -1) return;

  event.preventDefault();
  event.stopPropagation();
  viewerInstance.canvas.setPointerCapture?.(event.pointerId);
  isAltPressed.value = event.altKey;
  startWaypointDrag(index, windowPosition, event.altKey);
};

const onNativePointerMove = (event) => {
  const windowPosition = getNativeWindowPosition(event);
  if (!windowPosition) return;

  if (dragState.value) {
    event.preventDefault();
    event.stopPropagation();
    queueDragUpdate(windowPosition);
    return;
  }

  const index = findWaypointIndexAtWindowPosition(windowPosition);
  if (index !== -1) {
    hoveredWaypointIndex.value = index;
    setCanvasCursor('move');
    updateHintPosition(windowPosition);
  } else if (hoveredWaypointIndex.value !== null) {
    clearHoverState();
  }
};

const onNativePointerUp = (event) => {
  if (dragState.value) {
    event.preventDefault();
    event.stopPropagation();
  }

  if (viewerInstance?.canvas?.hasPointerCapture?.(event.pointerId)) {
    viewerInstance.canvas.releasePointerCapture(event.pointerId);
  }

  endWaypointDrag({ windowPosition: getNativeWindowPosition(event) });
};

const endWaypointDrag = (event) => {
  if (!dragState.value) return;

  const windowPosition = getWindowPosition(event);
  if (windowPosition) {
    pendingDragWindowPosition = windowPosition;
  }

  flushPendingDragUpdate();
  dragState.value = null;
  pendingDragWindowPosition = null;
  setCameraControlsEnabled(true);
  clearHoverState();
  queueMapClickSuppress();
};

const onMapClick = (e) => {
  if (!viewerInstance || !cesiumInstance) return;
  
  try {
    const windowPosition = e.windowPosition || e.position;
    if (!windowPosition) return;

    if (consumeMapClickSuppress() || isPickedWaypoint(windowPosition)) return;
    
    const ray = viewerInstance.camera.getPickRay(windowPosition);
    if (!ray) return;
    
    const cartesian3 = viewerInstance.scene.globe.pick(ray, viewerInstance.scene);
    if (!cartesian3) return;
    
    const cartographic = cesiumInstance.Cartographic.fromCartesian(cartesian3);
    if (!cartographic) return;
    
    const lng = cesiumInstance.Math.toDegrees(cartographic.longitude);
    const lat = cesiumInstance.Math.toDegrees(cartographic.latitude);
    
    emit('map-click', { lat, lng });
  } catch (error) {
    console.error('Map click error:', error);
  }
};

const onKeyDown = (event) => {
  if (event.key === 'Alt' || event.altKey) {
    isAltPressed.value = true;
  }
};

const onKeyUp = (event) => {
  if (event.key === 'Alt' || !event.altKey) {
    isAltPressed.value = false;
  }
};

const bindNativeDragEvents = () => {
  const canvas = viewerInstance?.canvas;
  if (!canvas || canvas.__waypointDragEventsBound) return;

  canvas.addEventListener('pointerdown', onNativePointerDown, true);
  canvas.addEventListener('pointermove', onNativePointerMove, true);
  canvas.addEventListener('pointerup', onNativePointerUp, true);
  canvas.addEventListener('pointercancel', onNativePointerUp, true);
  canvas.__waypointDragEventsBound = true;
};

const unbindNativeDragEvents = () => {
  const canvas = viewerInstance?.canvas;
  if (!canvas || !canvas.__waypointDragEventsBound) return;

  canvas.removeEventListener('pointerdown', onNativePointerDown, true);
  canvas.removeEventListener('pointermove', onNativePointerMove, true);
  canvas.removeEventListener('pointerup', onNativePointerUp, true);
  canvas.removeEventListener('pointercancel', onNativePointerUp, true);
  delete canvas.__waypointDragEventsBound;
};

const cleanupDragState = () => {
  flushPendingDragUpdate();
  dragState.value = null;
  pendingDragWindowPosition = null;
  setCameraControlsEnabled(true);
  clearHoverState();
  clearMapClickSuppress();
};

watch(() => props.isWaypointEditable, (isEditable) => {
  if (!isEditable) {
    cleanupDragState();
  }
});

watch(() => props.waypoints.length, () => {
  if (dragState.value && !props.waypoints[dragState.value.index]) {
    cleanupDragState();
  }
});

onMounted(() => {
  window.addEventListener('keydown', onKeyDown);
  window.addEventListener('keyup', onKeyUp);
  window.addEventListener('mouseup', endWaypointDrag);
  window.addEventListener('blur', cleanupDragState);
  bindNativeDragEvents();
});

onBeforeUnmount(() => {
  unbindNativeDragEvents();
  window.removeEventListener('keydown', onKeyDown);
  window.removeEventListener('keyup', onKeyUp);
  window.removeEventListener('mouseup', endWaypointDrag);
  window.removeEventListener('blur', cleanupDragState);

  if (dragAnimationFrame) {
    cancelAnimationFrame(dragAnimationFrame);
    dragAnimationFrame = null;
  }

  clearMapClickSuppress();

  setCameraControlsEnabled(true);
  setCanvasCursor('');
});
</script>

<style scoped>
.waypoint-drag-hint {
  position: absolute;
  z-index: 20;
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 6px 8px;
  color: #4b5563;
  font-size: 12px;
  line-height: 1;
  white-space: nowrap;
  pointer-events: none;
  background: rgba(255, 255, 255, 0.96);
  border: 1px solid #d1d5db;
  border-radius: 4px;
  box-shadow: 0 6px 16px rgba(15, 23, 42, 0.14);
}

.hint-row {
  display: flex;
  align-items: center;
  gap: 5px;
}

.mouse-icon {
  position: relative;
  width: 13px;
  height: 18px;
  background: #f3f4f6;
  border: 2px solid #9ca3af;
  border-radius: 8px;
  box-sizing: border-box;
}

.mouse-icon::before {
  position: absolute;
  top: 2px;
  left: 50%;
  width: 2px;
  height: 5px;
  content: '';
  background: #6b7280;
  border-radius: 2px;
  transform: translateX(-50%);
}

.keycap {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 22px;
  height: 18px;
  padding: 0 4px;
  color: #4b5563;
  font-size: 12px;
  font-weight: 600;
  background: #f9fafb;
  border: 1px solid #9ca3af;
  border-radius: 3px;
  box-shadow: inset 0 -1px 0 rgba(0, 0, 0, 0.08);
}
</style>
