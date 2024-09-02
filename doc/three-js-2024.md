# Getting Started with Three JS in 2024

This guide assumes you are on a fresh install of Ubuntu 24

## Prerequisites

Install the following:

### NVM and Node

Node version manager - helps you make sure you're running a recent version of Node. Installing Node via other means (eg. apt or dpkg) is not recommended.

Visit: https://github.com/nvm-sh/nvm/blob/master/README.md

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

COMMON GOTCHA: Nvm will not be immediately available in the window you installed it in, you have to restart the terminal window. See 'Troubleshooting on Linux'

Install the latest LTS version of Node. This is the most recent stable release.

```bash
nvm install 20
```

```bash
nvm use 20
```

Now,

```bash
node -v
```

Should yield

```
v20.17.0
```

### PNPM

Pnpm is a drop-in replacement for npm which is faster and more secure. We install it using the global flag on npm.

```bash
npm --global install pnpm
```

or

```bash
npm -g i pnpm
```

for short.

### Vite

Vite is a high quality tool that makes browser dev more pleasant. In any sane ecosystem it would be an added extra. For JS, it's mandatory! https://vitejs.dev/guide/

In your development directory, which we will assume exists...

```bash
cd ~/dev
```

This command will create a project folder:

```bash
pnpm create vite three-noodling
```

Follow the prompt, choose `Vue` and `Typescript`. This will create a new package (Javascript project in a directory) with Vue, Vite and Typescript ready installed and configured.

Follow the instructions issued:

```
Done. Now run:

  cd three-noodling
  pnpm install
  pnpm run dev
```

Visiting localhost, we find the 'Vite + Vue' starter page.

### Installing packages

In the terminal, open a new pane, shift+control+e or shift+control+o. Make sure you're in your project directory.

Install:
 - three.js
 - three.js type information

```bash
pnpm i three @types/three
```

Open the project in VSCode:

```bash
code .
```

Go to `http://localhost:5173/` in your browser (preferably Chrome, it's faster than Firefox for WebGL)

### First Steps with Vite / Vue

Ignore `index.html` for now, the heart of your app is `src/App.vue`

It demonstrates how to encapsulate functionality in custom components. Inspect `App.vue` and `HelloWorld.vue` taking note of how HelloWorld is referenced from App.

We will hijack HelloWorld.vue to be a game. Strip out almost everything from `App.vue` so it looks like this:

```vue
<script setup lang="ts">
import HelloWorld from './components/HelloWorld.vue'
</script>

<template>
  <HelloWorld/>
</template>

<style scoped>
</style>
```

Going back to the browser, no need to refresh, the page will have the branding removed.

Rename `HelloWorld.vue` to `Game.vue` and update all three references to it in `App.vue`:

```vue
<script setup lang="ts">
import Game from './components/Game.vue'
</script>

<template>
  <Game/>
</template>

<style scoped>
</style>
```

Game will complain that we are not passing it a prop. Rather than satisfying the requirement, remove it by similarly gutting `Game.vue`:

```vue
<script setup lang="ts">
</script>

<template>
    Hello this is Game.vue
</template>

<style scoped>
</style>
```

At this point, VSCode might get angry and give you red import errors. Restart VSCode. Annoying right? This mostly happens when you're renaming files or refactoring.

### Three JS

Let's verify that three.js is being correctly imported. Following the guide at:

https://threejs.org/docs/#manual/en/introduction/Creating-a-scene

Our first opportunity to check things are as expected is to `console.log` the THREE module itself to see if it's present.

Insert code into the `<script setup lang="ts">` tag:

```typescript
import * as THREE from 'three';
console.log('Do we have three js?', THREE)
```

Back in the browser, a message appears in the console (F12)

```Do we have three js? 
Object { ACESFilmicToneMapping: 4, AddEquation: 100, AddOperation: 2, AdditiveAnimationBlendMode: 2501, AdditiveBlending: 2, AgXToneMapping: 6, AlphaFormat: 1021, AlwaysCompare: 519, AlwaysDepth: 1, AlwaysStencilFunc: 519, … }```

Continue to follow the three.js tutorial to produce a `Game.vue` like this:

```vue
<script setup lang="ts">
import * as THREE from 'three';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);

const renderer = new THREE.WebGLRenderer();
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setAnimationLoop(animate);
document.body.appendChild(renderer.domElement);

const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
const cube = new THREE.Mesh(geometry, material);
scene.add(cube);

camera.position.z = 5;

function animate() {
    cube.rotation.x += 0.01;
    cube.rotation.y += 0.01;
    renderer.render(scene, camera);
}
</script>

<template>
</template>

<style scoped></style>
```

There is some extraneous styling in `style.css` - delete all of this and replace it with your favourite CSS reset sheet, at the very least removing margin from `<body>`.

```css
body { margin: 0 }
```

### Converting the Three JS example to idiomatic Vue

`Game.vue` works, but is very old-school. It creates a new DOM element and uses the native browser API (`document.body.appendChild`) to insert it. The Vue way is different. Rather than inserting the canvas into the body, put it in a container element.

Also, this code is executed very early! Code at the root level `<script setup lang="ts">` is run *before the DOM for the component is ready*, which means the container ref is not accessible until the Mounting phase of the lifecycle. Vue can be instructed to run code when the component is ready with the `onMounted` hook. This is where the three.js bootstrapping logic will live. Think of the root level as 'setting up variables' and the onMounted as 'starting the game once everything is ready'

Using the usual `const myElement = ref()` results in an untyped ref for the canvas, so provide some explicit typing information:

```vue
const container: Ref<HTMLCanvasElement | undefined> = ref()
```

At root level, `container.value` will be undefined. in `onMounted`, `container.value` will (if the corresponding element exists in the template) contain a full `HTMLElement`, aka a `<div>`.

```vue
<script setup lang="ts">
import * as THREE from 'three';
import { ref, onMounted, Ref } from 'vue';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);

const renderer = new THREE.WebGLRenderer();
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setAnimationLoop(animate);

const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
const cube = new THREE.Mesh(geometry, material);
scene.add(cube);

camera.position.z = 5;

onMounted(() => {
    container.value?.appendChild(renderer.domElement);
})

function animate() {
    cube.rotation.x += 0.01;
    cube.rotation.y += 0.01;
    renderer.render(scene, camera);
}

const container: Ref<HTMLElement | undefined> = ref()

</script>

<template>
    <div ref="container"></div>
</template>

<style scoped></style>
```

Have fun!