<template>
  <div class="wrap">
    <div class="overlay" ref="overlay" v-show="loaded">

    </div>
    <div v-show="!loaded">
      <div class="progress-bar">
        <div class="progress-inner">

        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { Application, Assets, Texture } from "pixi.js";
import { CompositeTilemap } from "@pixi/tilemap";
import { computed, onBeforeUnmount, onMounted, ref } from "vue";

const overlay = ref<HTMLElement | null>(null);
const loadingProgress = ref(0.0);
const loadingPercent = computed(() => (loadingProgress.value * 100) + '%');
const loaded = computed(() => loadingProgress.value >= 1.0);

let active = true;
let frame = 0;

const loadAssets = async () => {
  Assets.add('atlas', 'assets/atlas.json');
  Assets.add('button', 'assets/button.png');

  await Assets.load(['atlas', 'button'], progress => loadingProgress.value = progress);
}

const buildTilemap = (tilemap: CompositeTilemap) => {
  // Clear everything, like a PIXI.Graphics
  tilemap.clear();

  const size = 32;

  // if you are too lazy, just specify filename and pixi will find it in cache
  for (let i = 0; i < 7; i++) {
    for (let j = 0; j < 5; j++) {
      tilemap.tile('grass.png', i * size, j * size);

      if (i % 2 === 1 && j % 2 === 1) {
        tilemap.tile('tough.png', i * size, j * size);
      }
    }
  }

  // if you are lawful citizen, please use textures from
  const textures = Assets.get("atlas").textures;

  tilemap.tile(textures['brick.png'], 2 * size, 2 * size);
  tilemap.tile(textures['brick_wall.png'], 2 * size, 3 * size, { alpha: 0.6 });

  // chest will be animated!
  // old way: animate on rebuild
  // tilemap.addFrame(textures[frame % 2 == 0 ? "chest.png" : "red_chest.png"], 4 * size, 4 * size);

  // new way: animate on shader: 2 frames , X offset is 32 , "red_chest" is exactly 34 pixels right in the atlas
  // Frame changes every 100ms because of `setInterval(animShader, 100)`, so the first animation
  // will change to the next frame every 100ms
  tilemap.tile(textures['chest.png'], 4 * size, 4 * size).tileAnimX(34, 2);

  // You can also set independent time for each tile.
  // In this second chest, we pass 3 to tileAnimDivisor
  // 3 multiplies the 100ms we have in setInterval by 3, making the duration 300ms
  tilemap.tile(textures['chest.png'], 5 * size, 4 * size).tileAnimX(34, 2).tileAnimDivisor(3);
  // You can alternatively set it by passing animDivisor option when creating the tile. Below a frame duration is 600ms
  tilemap.tile(textures['chest.png'], 8 * size, 4 * size, { animX: 34, animCountX: 2, animDivisor: 6 });

  // button does not appear in the atlas, but tilemap wont surrender, it will create second layer for special for buttons
  // buttons will appear above everything
  tilemap.tile(Assets.get("button"), 6 * size, 2 * size);

  // if you want rotations:
  // https://pixijs.io/examples-v4/#/textures/texture-rotate.js
  // textures should have frame, orig and trim to do that
  // canvas in pixi-tilemap does not work with rotate!!!!
  const origTex = textures['chest.png'];

  for (let i = 0; i < 8; i++) {
    const frame = origTex.frame.clone();
    const orig = origTex.orig.clone();
    const trim = origTex.orig.clone();
    const rotate = i * 2;

    if (rotate % 4 === 2) {
      orig.width = frame.height;
      orig.height = frame.width;
    }

    const tmpTex = new Texture(origTex.baseTexture, frame, orig, trim, rotate);

    // Swap W and H in orig if you rotate%4 is not 0
    tilemap.tile(tmpTex, i % 4 * size, ((i >> 2) * size) + (5 * size));
    // rotate is also last parameter in addFrame
  }
}

const sleep = (ms: number) => new Promise(resolve => setTimeout(resolve, ms));
const runAnimations = async (tilemap: CompositeTilemap) => {
  while (active) {
    await sleep(100);
    tilemap.tileAnim = [frame, frame];
    frame++;
  }
}

onBeforeUnmount(() => active = false);
onMounted(async () => {
  if (overlay.value == null) {
    throw new Error("Overlay is null!");
  }

  const app = new Application({
    resolution: window.devicePixelRatio || 1,
    autoDensity: true,
    forceCanvas: false,
    width: overlay.value.clientWidth,
    height: overlay.value.clientHeight,
    antialias: false,
    premultipliedAlpha: true,
    backgroundAlpha: 0,
    backgroundColor: 0x000,
    resizeTo: overlay.value,
    clearBeforeRender: true,
    powerPreference: "high-performance",
  });

  await loadAssets();

  const tilemap = new CompositeTilemap();
  app.stage.addChild(tilemap);

  buildTilemap(tilemap);


  overlay.value.appendChild(app.view as any as Node);
  app.resize();

  runAnimations(tilemap);
});
</script>

<style scoped>
.progress-bar {
  width: 100%;
  height: 10px;
  background: gray;
}

.progress-inner {
  width: v-bind(loadingPercent);
  height: 100%;
  background: red;
  transition: width 0.2s;
}

.wrap {
  max-width: 500px;
  margin: 0 auto;
}

.overlay {
  width: 100%;
  height: 500px;
}
</style>