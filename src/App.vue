<script lang="ts" setup>
import { createEventHook } from "@vueuse/core";

import { ref, type ShallowRef, type Ref, onMounted } from "vue";
const canvasRef = ref<HTMLCanvasElement | null>(null);
const canvasInitResult = createEventHook<Ref<HTMLCanvasElement | null>>();
onMounted(() => {
	canvasInitResult.trigger(canvasRef);
});

import { shallowRef } from "vue";
import { Application, HTMLText } from "pixi.js";
import { useWindowSize, useDevicePixelRatio } from "@vueuse/core";
const { width, height } = useWindowSize();
const { pixelRatio } = useDevicePixelRatio();
const appRef = shallowRef<Application | null>(null);
const appInitResult = createEventHook<ShallowRef<Application | null>>();
canvasInitResult.on(async (canvasRef) => {
	if (canvasRef.value === null) {
		return;
	}
	const app = new Application();
	appRef.value = app;

	await app.init({
		width: width.value / pixelRatio.value,
		height: height.value / pixelRatio.value,
		resolution: pixelRatio.value,
		autoDensity: true,

		antialias: true,

		canvas: canvasRef.value,

		preference: "webgpu",

		backgroundColor: "gray",
	});
	appRef.value = app;

	app.renderer.resize(width.value, height.value);
	appRef.value = app;

	appInitResult.trigger(appRef);
});

import { Viewport } from "pixi-viewport";
const viewportRef = shallowRef<Viewport | null>(null);
const viewportInitResult = createEventHook<ShallowRef<Viewport | null>>();
appInitResult.on((appRef) => {
	if (appRef.value === null) {
		return;
	}

	const app = appRef.value;

	const viewport = new Viewport({
		events: app.renderer.events,
	});
	viewportRef.value = viewport;
	appRef.value = app;

	app.stage.addChild(viewport);
	viewportRef.value = viewport;
	appRef.value = app;

	viewport.drag().wheel().pinch().clampZoom({
		maxScale: 4,
		minScale: 0.1,
	});
	viewportRef.value = viewport;

	viewportInitResult.trigger(viewportRef);
});

// need empty import or IDE report error
import { Layout } from "@pixi/layout";
import { Sprite, Assets, Container } from "pixi.js";
import canvasSize from "canvas-size";
viewportInitResult.on((viewportRef) => {
	if (viewportRef.value === null) {
		return;
	}
	const viewport = viewportRef.value;

	// rwd a4 size and dpi
	const maxWidth = width.value - 32 * 2;
	const maxHeight = height.value - 32 * 2;
	const ratio = 210 / 297;
	function calculateMaxDimensions(
		maxWidth: number,
		maxHeight: number,
		ratio: number,
	): { width: number; height: number } {
		// 確保比例為正數
		if (ratio <= 0) {
			throw new Error("Ratio must be greater than 0");
		}

		// 計算最大高度
		const height = Math.min(maxHeight, maxWidth / ratio);

		// 計算最大寬度
		const width = Math.min(maxWidth, ratio * maxHeight);

		return { width, height };
	}
	const size = calculateMaxDimensions(maxWidth, maxHeight, ratio);
	const a4DPI = size.width / 8.27;

	// layout provide order, gap, position, and size for page
	// if you want to change lauout, just to update layout style
	const pageLayout = new Container().initLayout();
	pageLayout.layout?.setStyles({
		// top-down
		// width: "100%",
		// height: "100%",

		// left-right
		// none

		padding: convertPtToPx(32, a4DPI),
	});
	viewport.addChild(pageLayout);
	viewportRef.value = viewport;

	const page1 = new Container().initLayout();
	page1.label = "page1";
	pageLayout.layout?.addContent([
		{
			content: page1,
			styles: {
				background: "white",
				width: size.width,
				height: size.height,

				// left-right
				marginRight: convertPtToPx(32, a4DPI),

				// top-down
				// marginBottom: convertPtToPx(32, a4DPI),
				// marginLeft: (width.value - size.width) / 2 - convertPtToPx(32, a4DPI),
			},
			id: page1.label,
		},
	]);
	const page2 = new Container().initLayout();
	page2.label = "page2";
	pageLayout.layout?.addContent([
		{
			content: page2,
			styles: {
				background: "red",
				width: size.width,
				height: size.height,

				// left-right
				// none

				// top-down
				// marginLeft: (width.value - size.width) / 2 - convertPtToPx(32, a4DPI),
			},
			id: page2.label,
		},
	]);

	// get page's xy, width,height from their layout
	const page1Layout = pageLayout.layout?.getChildByID(page1.label);
	if (page1Layout === undefined) {
		return;
	}
	const page2Layout = pageLayout.layout?.getChildByID(page2.label);
	if (page2Layout === undefined) {
		return;
	}

	const text1 = new HTMLText();
	// const textcontent1 =
	// 	"輝達發布新聞稿說明，Fugatto是Foundational Generative Audio Transformer Opus 1的縮寫，可使用文字與音訊檔案的任何組合，產生或轉換描述的音樂、語音與聲音。\n輝達應用音訊研究部門經理巴耶（Rafael Valle）表示，「我們希望創造一個能夠像人類一樣理解和生成聲音的模型」，Fugatto是邁向未來的第一步。\n輝達指出，音樂製作人可以使用Fugatto快速製作聲音的原型或編輯歌曲構想，嘗試各種風格、聲音和樂器，也能加入效果並提升現有曲目整體音訊品質。";
	const textcontent1 = "這";

	// always write style in text so as to work normaly
	text1.text = `<div style='font-size: ${convertPtToPx(300, a4DPI)}px;font-weight: normal;line-height: normal;text-decoration: underline;'>${textcontent1}</div>`;
	text1.style.fill = "black";
	/**
	 * 將字體大小從 pt 轉換為 px，根據指定的 DPI。
	 * @param fontSizePt 字體大小（以 pt 為單位）
	 * @param dpi 輸出的解析度（DPI，默認為 72）
	 * @returns 字體大小（以 px 為單位）
	 */
	function convertPtToPx(fontSizePt: number, dpi: number): number {
		if (fontSizePt <= 0 || dpi <= 0) {
			throw new Error("fontSizePt and dpi must be greater than 0");
		}
		return (fontSizePt * dpi) / 72;
	}

	text1.style.wordWrap = true;
	text1.style.wordWrapWidth = size.width;
	text1.resolution = 5;

	// check is text too large then recover to normal style
	canvasSize
		.maxArea({
			usePromise: true,
		})
		.then((rs) => {
			if (
				text1.width * (text1.resolution | 1) >= rs.width ||
				text1.height * (text1.resolution | 1) >= rs.height
			) {
				console.log("word break!");
				text1.resolution = 5;
				text1.text = `<div style='font-size: ${convertPtToPx(24, a4DPI)}px;font-weight: normal;line-height: normal;text-decoration: underline;'>${textcontent1}</div>`;
			}
		});

	text1.label = "text1";
	page1Layout.addChild(text1);
	viewportRef.value = viewport;

	const image1 = new Sprite();
	image1.label = "image1";
	page2Layout.addChild(image1);
	image1.position.set(page2Layout.width / 2, page2Layout.height / 2);
	viewportRef.value = viewport;
	Assets.load("https://pixijs.com/assets/bunny.png")
		.then((resolvedTexture) => {
			image1.texture = resolvedTexture;
		})
		.then(() => {
			image1.width = convertPtToPx(image1.width, a4DPI);
			image1.height = convertPtToPx(image1.height, a4DPI);
		});

	// console.log(pageLayout.layout?.isPortrait);
	// need this or no work!
	pageLayout.layout?.refresh();
	// console.log(pageLayout.layout?.isPortrait);

	// // page1
	viewport.setZoom(1);
	viewport.moveCenter(
		page1Layout.x + page1Layout.width / 2,
		page1Layout.y + page1Layout.height / 2,
	);

	// // page2
	viewport.setZoom(1);
	viewport.moveCenter(
		page2Layout.x + page2Layout.width / 2,
		page2Layout.y + page2Layout.height / 2,
	);

	// animate slide
	viewport.decelerate().animate({
		position: {
			x: page1Layout.x + page1Layout.width / 2,
			y: page1Layout.y + page1Layout.height / 2,
		},
		scale: 1,
		ease: "easeInOutSine", // default: linear
		time: 1300,
		callbackOnComplete: () => {
			viewport.decelerate().animate({
				position: {
					x: page2Layout.x + page2Layout.width / 2,
					y: page2Layout.y + page2Layout.height / 2,
				},
				scale: 1,
				ease: "easeInOutSine", // default: linear
				time: 1300,
			});
		},
	});

	// border
	// viewport.bounce();
});
</script>

<template>
  <div class="container">
    <canvas class="canvas" ref="canvasRef"></canvas>
  </div>
</template>

<style lang="scss" scoped>
html,body,.container,.canvas {
	height: 100dvh !important;
	width: 100dvw !important;
	overflow: hidden !important;
}
</style>