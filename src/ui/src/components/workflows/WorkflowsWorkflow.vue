<template>
	<div
		ref="rootEl"
		class="WorkflowsWorkflow"
		@click="handleClick"
		@dragover="handleDragover"
		@drop="handleDrop"
		@drag="handleDrag"
		@dragstart="handleDragstart"
	>
		<div ref="nodeContainerEl" class="nodeContainer">
			<svg>
				<WorkflowArrow
					v-for="(arrow, arrowId) in arrows"
					:key="arrowId"
					:arrow="arrow"
					:is-selected="selectedArrow == arrowId"
					:is-engaged="
						selectedArrow == arrowId ||
						wfbm.getSelectedId() == arrow.fromNodeId ||
						wfbm.getSelectedId() == arrow.out.toNodeId
					"
					@click="handleArrowClick($event, arrowId)"
					@delete="handleDeleteClick($event, arrow)"
				></WorkflowArrow>
			</svg>
			<template v-for="node in nodes" :key="node.id">
				<component
					:is="renderProxiedComponent(node.id)"
					:data-writer-unselectable="isUnselectable"
					:style="{
						top: `${node.y - renderOffset.y}px`,
						left: `${node.x - renderOffset.x}px`,
					}"
					@drag="(ev: DragEvent) => handleNodeDrag(ev, node.id)"
					@dragstart="(ev: DragEvent) => handleNodeDragStart(ev)"
					@dragend="handleNodeDragend"
					@click="(ev: MouseEvent) => handleNodeClick(ev, node.id)"
					@out-select="
						(outId: string) => handleNodeOutSelect(node.id, outId)
					"
				></component>
			</template>
		</div>
		<WdsButton class="runButton" variant="secondary" @click="handleRun">
			<i class="material-symbols-outlined">play_arrow</i>
			{{ isRunning ? "Running..." : "Run" }}</WdsButton
		>
		<WorkflowMiniMap
			v-if="nodeContainerEl"
			:node-container-el="nodeContainerEl"
			:render-offset="renderOffset"
			class="miniMap"
			@change-render-offset="handleChangeRenderOffset"
		></WorkflowMiniMap>
	</div>
</template>

<script lang="ts">
import { type Component, FieldType } from "@/writerTypes";
import WorkflowArrow from "./base/WorkflowArrow.vue";
import { watch } from "vue";
import WdsButton from "@/wds/WdsButton.vue";
import WorkflowMiniMap from "./base/WorkflowMiniMap.vue";

const description =
	"A container component representing a single workflow within the application.";

export default {
	writer: {
		name: "Workflow",
		toolkit: "workflows",
		category: "Root",
		description,
		allowedChildrenTypes: ["*"],
		allowedParentTypes: ["workflows_root"],
		fields: {
			key: {
				name: "Workflow key",
				desc: "Unique identifier. It's needed to enable navigation to this Workflow.",
				type: FieldType.IdKey,
			},
		},
		previewField: "key",
	},
};

export type WorkflowArrowData = {
	x1: number;
	y1: number;
	x2: number;
	y2: number;
	color: string;
	fromNodeId: Component["id"];
	out: Component["outs"][number];
	isEngaged: boolean;
};
</script>
<script setup lang="ts">
import { Ref, computed, inject, nextTick, onMounted, ref } from "vue";
import { useComponentActions } from "@/builder/useComponentActions";
import { useDragDropComponent } from "@/builder/useDragDropComponent";
import injectionKeys from "@/injectionKeys";
const renderProxiedComponent = inject(injectionKeys.renderProxiedComponent);

const rootEl: Ref<HTMLElement | null> = ref(null);
const nodeContainerEl: Ref<HTMLElement | null> = ref(null);
const fields = inject(injectionKeys.evaluatedFields);
const wf = inject(injectionKeys.core);
const wfbm = inject(injectionKeys.builderManager);
const arrows: Ref<WorkflowArrowData[]> = ref([]);
const renderOffset = ref({ x: 0, y: 0 });
const isRunning = ref(false);
let clickOffset = { x: 0, y: 0 };
const selectedArrow = ref(null);
const instancePath = inject(injectionKeys.instancePath);
const workflowComponentId = inject(injectionKeys.componentId);

const nodes = computed(() =>
	wf.getComponents(workflowComponentId, { sortedByPosition: true }),
);

const { createAndInsertComponent, addOut, removeOut, changeCoordinates } =
	useComponentActions(wf, wfbm);
const { getComponentInfoFromDrag } = useDragDropComponent(wf);

const activeNodeOut: Ref<{
	fromComponentId: string;
	outId: string;
} | null> = ref(null);

function refreshArrows() {
	const a = [];
	const canvasCBR = rootEl.value?.getBoundingClientRect();
	if (!canvasCBR) {
		return;
	}

	const arrowSlot: Record<Component["id"], number> = {};

	nodes.value
		.filter((node) => node.outs?.length > 0)
		.forEach((node) => {
			const fromNodeId = node.id;
			node.outs.forEach((out) => {
				const fromEl = document.querySelector(
					`[data-writer-id="${fromNodeId}"] [data-writer-socket-id="${out.outId}"]`,
				);
				const toEl = document.querySelector(
					`[data-writer-id="${out.toNodeId}"]`,
				);
				if (!fromEl || !toEl) return;
				const fromCBR = fromEl.getBoundingClientRect();
				const toCBR = toEl.getBoundingClientRect();

				a.push({
					x1: fromCBR.x - canvasCBR.x + fromCBR.width / 2,
					y1: fromCBR.y - canvasCBR.y + fromCBR.height / 2,
					x2: toCBR.x - canvasCBR.x,
					y2: toCBR.y - canvasCBR.y + toCBR.height / 2,
					color: getComputedStyle(fromEl).backgroundColor,
					fromNodeId,
					out,
				});
			});
		});
	arrows.value = a;
}

const isUnselectable = computed(() => {
	if (activeNodeOut.value === null) return null;
	return true;
});

function handleClick() {
	selectedArrow.value = null;
}

async function handleRun() {
	if (isRunning.value) return;
	isRunning.value = true;
	await wf.forwardEvent(
		new CustomEvent("wf-builtin-run", {
			detail: {
				callback: () => {
					isRunning.value = false;
				},
			},
		}),
		instancePath,
		false,
	);
}

function handleNodeOutSelect(componentId: Component["id"], outId: string) {
	activeNodeOut.value = {
		fromComponentId: componentId,
		outId,
	};
}

function getEmptyDragImage() {
	var img = new Image();
	img.src =
		"data:image/gif;base64,R0lGODlhAQABAIAAAAUEBAAAACwAAAAAAQABAAACAkQBADs=";
	return img;
}

function handleDragstart(ev: DragEvent) {
	ev.dataTransfer.setDragImage(getEmptyDragImage(), 0, 0);
	clickOffset = getAdjustedCoordinates(ev);
}

function handleDrag(ev: DragEvent) {
	ev.preventDefault();
	const { x, y } = getAdjustedCoordinates(ev);
	if (x < 0 || y < 0) return;
	renderOffset.value.x = Math.max(
		0,
		renderOffset.value.x - (x - clickOffset.x),
	);
	renderOffset.value.y = Math.max(
		0,
		renderOffset.value.y - (y - clickOffset.y),
	);
	refreshArrows();
}

function handleDragover(ev: DragEvent) {
	ev.preventDefault();
	ev.stopPropagation();
}

function getAdjustedCoordinates(ev: DragEvent) {
	const canvasCBR = rootEl.value.getBoundingClientRect();
	const x = ev.pageX - canvasCBR.x + renderOffset.value.x;
	const y = ev.pageY - canvasCBR.y + renderOffset.value.y;
	return { x, y };
}

function handleDrop(ev: DragEvent) {
	ev.preventDefault();
	ev.stopPropagation();
	const dropInfo = getComponentInfoFromDrag(ev);

	if (!dropInfo) return;
	const { draggedType, draggedId } = dropInfo;
	if (draggedId) return;

	const { x, y } = getAdjustedCoordinates(ev);
	if (x < 0 || y < 0) return;

	createNode(draggedType, x, y);
}

function handleArrowClick(ev: MouseEvent, arrowId: number) {
	if (selectedArrow.value == arrowId) {
		selectedArrow.value = null;
		return;
	}
	ev.stopPropagation();
	selectedArrow.value = arrowId;
	wfbm.setSelection(null);
}

async function handleDeleteClick(ev: MouseEvent, arrow: WorkflowArrowData) {
	removeOut(arrow.fromNodeId, arrow.out);
}

function handleNodeDragStart(ev: DragEvent) {
	ev.stopPropagation();
	ev.dataTransfer.setDragImage(getEmptyDragImage(), 0, 0);
	clickOffset = {
		x: ev.offsetX,
		y: ev.offsetY,
	};
}

function handleNodeDrag(ev: DragEvent, componentId: Component["id"]) {
	ev.preventDefault();
	ev.stopPropagation();
	const { x, y } = getAdjustedCoordinates(ev);
	if (x < 0 || y < 0) return;

	const component = wf.getComponentById(componentId);

	const newX = x - clickOffset.x;
	const newY = y - clickOffset.y;

	if (component.x == newX && component.y == newY) return;

	component.x = newX;
	component.y = newY;

	setTimeout(() => {
		// Debouncing
		if (component.x !== newX) return;
		if (component.y !== newY) return;
		changeCoordinates(componentId, newX, newY);
	}, 200);
}

function handleNodeDragend(ev: DragEvent) {
	ev.preventDefault();
}

async function handleNodeClick(ev: MouseEvent, componentId: Component["id"]) {
	if (!activeNodeOut.value) return;
	if (activeNodeOut.value.fromComponentId == componentId) return;

	addOut(activeNodeOut.value.fromComponentId, {
		toNodeId: componentId,
		outId: activeNodeOut.value.outId,
	});

	activeNodeOut.value = null;
}

async function createNode(type: string, x: number, y: number) {
	createAndInsertComponent(type, workflowComponentId, undefined, {
		x: Math.floor(x),
		y: Math.floor(y),
	});
}

async function findAndCenterBlock(componentId: Component["id"]) {
	const el = rootEl.value.querySelector(`[data-writer-id="${componentId}"]`);
	const canvasCBR = rootEl.value?.getBoundingClientRect();
	if (!el || !canvasCBR) return;
	const { width, height } = el.getBoundingClientRect();
	const component = wf.getComponentById(componentId);
	if (!component) return;
	renderOffset.value = {
		x: Math.max(0, component.x - canvasCBR.width / 2 + width / 2),
		y: Math.max(0, component.y - canvasCBR.height / 2 + height / 2),
	};
	await nextTick();
	refreshArrows();
}

async function handleChangeRenderOffset(payload) {
	renderOffset.value = {
		x: payload.x,
		y: payload.y,
	};
	await nextTick();
	refreshArrows();
}

watch(
	() => wfbm.getSelection(),
	(newSelection) => {
		if (!newSelection) return;
		selectedArrow.value = null;
		if (!wf.isChildOf(workflowComponentId, newSelection.componentId))
			return;
		if (newSelection.source !== "click") {
			findAndCenterBlock(newSelection.componentId);
		}
	},
);

watch(
	nodes,
	async () => {
		// Refresh arrows

		await nextTick();
		refreshArrows();
	},
	{ deep: true },
);

onMounted(async () => {
	await nextTick();
	refreshArrows();
});
</script>

<style scoped>
@import "@/renderer/sharedStyles.css";

.WorkflowsWorkflow {
	display: flex;
	width: 100%;
	min-height: 100%;
	background: var(--builderSubtleSeparatorColor);
	flex: 1 0 auto;
	flex-direction: row;
	align-items: stretch;
	position: relative;
	overflow: hidden;
}

button.runButton {
	position: absolute;
	right: 32px;
	top: 32px;
}

.component.WorkflowsWorkflow.selected {
	background: var(--builderSubtleSeparatorColor);
}

.miniMap {
	position: absolute;
	bottom: 32px;
	left: 32px;
}

.nodeContainer {
	position: absolute;
	top: 0;
	left: 0;
	overflow: hidden;
	width: 100%;
	height: 100%;
}

svg {
	position: absolute;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
}
</style>
