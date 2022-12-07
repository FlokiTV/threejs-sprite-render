<script lang="ts">
  import * as THREE from 'three';
  import JSZip from "jszip"
  import { saveAs } from 'file-saver';
  import { onMount } from 'svelte';
  import Field from './lib/Field.svelte';
  import { GLTFLoader } from 'three-stdlib';
  /*
    TODO
      rename folder on export
      change preview size
      tooltip on buttons
  */
  let editor;
  let gallery;
  let width = 128;
  let height = 128;
  let fps = 8;// preview FPS
  let frames = 8;// total angles to render
  let rendering = false
  let play = false
  let last = Date.now()
  let camX = 0
  let camY = 0
  let model;
  let mainModel = new THREE.Object3D();
  let modelRotOffset = 0;// offset rotation to render
  let modelAngle = 45;
  let fileName = "duck";
  /*
    Setup THREE.js
  */
  let scene = new THREE.Scene();
  const camera = new THREE.OrthographicCamera( 5 / - 2, 5 / 2, 5 / 2, 5 / - 2, 1, 200 );
  camera.position.z = 3;
  /*🎦 Renderer */
  let renderer = new THREE.WebGLRenderer({alpha: true,preserveDrawingBuffer: true });
  renderer.setSize( window.innerWidth, window.innerHeight ); //used only on init
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  /*💡 Lights */
  const ambient = new THREE.AmbientLight( 0x111111 );
  scene.add(ambient)

  let hemiLightForce = 1
  let hemiLight;
  const setHemiLight = () =>{
    /* we need reset hemiLight to apply force */
    if(hemiLight) scene.remove(hemiLight)
    hemiLight = new THREE.HemisphereLight(0xffeeb1, 0x080820, hemiLightForce);
    scene.add(hemiLight);
  }
  setHemiLight(); //init hemiLight

  const light = new THREE.SpotLight(0xffffff)
  light.position.set( 0, 0, 200 )
  // light.shadow.bias = -0.0001;
  // light.shadow.mapSize.width = 1024*4;
  // light.shadow.mapSize.height = 1024*4;
  scene.add(light)
  scene.add(mainModel)
  /*
    Utils Functions
  */
  let toRad = angle => 2 * Math.PI * (angle / 360)
  const loadGLTFFromURL = url =>{
    let c, size; // model center and size
    if(model) mainModel.remove(model)
    let loader = new GLTFLoader()
    loader.load(url,
      (gltf) => {
            gltf.scene.traverse(function (child) {
                if ((child as THREE.Mesh).isMesh) {
                    const m = (child as THREE.Mesh)
                    m.receiveShadow = true
                    m.castShadow = true
                    m.material.metalness = 0;
                    if(m.material.map) m.material.map.anisotropy = 16;
                }
                if (((child as THREE.Light)).isLight) {
                    const l = (child as THREE.Light)
                    l.castShadow = true
                    l.shadow.bias = -.003
                    l.shadow.mapSize.width = 2048
                    l.shadow.mapSize.height = 2048
                }
            })
            const box = new THREE.Box3().setFromObject( gltf.scene );		 
            c = box.getCenter( new THREE.Vector3() );
            size = box.getSize( new THREE.Vector3() );
            gltf.scene.position.set( -c.x, size.y / 2 - c.y, -c.z );
            model = gltf.scene;
            setModelAngle(modelAngle);
            mainModel.add(model)
        },
        (xhr) => {
            console.log((xhr.loaded / xhr.total) * 100 + '% loaded')
        },
        (error) => {
            console.log(error)
        })
  }
  function resizeCanvasToDisplaySize() {
    const canvas = renderer.domElement;
    // look up the size the canvas is being displayed
    const width = canvas.clientWidth;
    const height = canvas.clientHeight;
    // adjust displayBuffer size to match
    if (canvas.width !== width || canvas.height !== height) {
      // you must pass false here or three.js sadly fights the browser
      renderer.setSize(width, height, false);
      camera.aspect = width / height;
      camera.updateProjectionMatrix();
      // update any render target sizes here
    }
  }
  function saveImage() {
    const canvas =  document.getElementsByTagName("canvas")[0]
    const image = canvas.toDataURL("image/png");
    if(false){
      const a = document.createElement("a");
      a.href = image.replace(/^data:image\/[^;]/, 'data:application/octet-stream');
      a.download="image.png"
      a.click();
    }else{
      let i = document.createElement("img")
      i.src = image
      i.classList.add("float-left")
      gallery.appendChild(i)
    }
  }
  const setModelAngle = (angle) =>{
    modelAngle = angle
    mainModel.rotation.x = toRad(modelAngle);
  }
  const setModelRotOffset = () =>{
    mainModel.rotation.y = toRad(modelRotOffset);
    console.log(modelRotOffset)
  }
  async function uploadArchive(e) {
    const file = e.target.files[0];
    fileName = file.name
    if (file == null) return; // If user cancels file selection
    const url = URL.createObjectURL(file);
    loadGLTFFromURL(url)
  }
  function renderSprite(step=0, total=7){
    if(step){
      mainModel.rotation.y -= toRad(360/frames);
    }else{
      /* init */
      gallery.innerHTML = "";
      rendering = true
      mainModel.rotation.y = 0 + toRad(modelRotOffset);
    }
    renderer.render( scene, camera );
    saveImage()
    let nxt = step +1
    if(step < total) renderSprite(nxt, total)
    else rendering = false
  }
  const saveZip = () =>{
    var zip = new JSZip();
    var img = zip.folder(fileName);
    let imgs = gallery.querySelectorAll("img")
    if(!imgs.length) return
    for (let i = 0; i < imgs.length; i++) {
      let s = imgs[i].src.split(";")[1]
      img.file(i+".png", imgs[i].src.split(";base64,")[1], {base64: true});
    }
    zip.generateAsync({type:"blob"})
    .then(function(content) {
        saveAs(content, "example.zip");
    });
  }
  const setCamX = (x) =>{
    camera.position.x += x
    camX = camera.position.x.toFixed(2)
  }
  const setCamY = (y) =>{
    camera.position.y += y
    camY = camera.position.y.toFixed(2)
  }
  $: isBtnActive = (f,t) => f == t ? "btn-success":"";
  // 
  let animate = function () {
    if(!rendering) requestAnimationFrame( animate );
    if(Date.now() - last < 1000/fps) return
    else last = Date.now()
    if(model && play){
      mainModel.rotation.y -= toRad(360/frames);
    }
    renderer.render( scene, camera );
  };
  animate()
  onMount(()=>{
     let duck = "https://raw.githubusercontent.com/KhronosGroup/glTF-Sample-Models/master/2.0/Duck/glTF-Embedded/Duck.gltf";
    loadGLTFFromURL(duck);
    editor.appendChild( renderer.domElement );
    renderer.domElement.classList.add("border");
    renderer.domElement.classList.add("bg-base-300");
    renderer.domElement.style.width = width+"px";
    renderer.domElement.style.height = height+"px";
    resizeCanvasToDisplaySize()
  })
</script>

<div class="min-h-screen p-2">
  <div class="flex w-full justify-center flex-col">
    <div class="flex space-x-2">
      <!-- Fields -->
      <div class="w-[250px] bg-base-300 p-2">
        <Field title="preview" value={fileName}>
          <div class="flex justify-center">
            <div class="w-[{width}px] h-[{height}px]" bind:this={editor} on:click={()=> play = !play}>
              <!-- canvas -->
            </div>
          </div>
          <input class="file-input file-input-bordered file-input-xs w-full max-w-xs mt-2"
            type="file"
            accept=".gltf,.glb"
            on:change={uploadArchive}
          >
        </Field>
        <Field title="controls">
          <div class="flex justify-center">
            <div class="tooltip" data-tip="{!play ? 'play' : 'pause'}">
              <button class="btn btn-sm text-[20px] btn-ghost" on:click={()=> play = !play}>{ !play ? '▶' : '⏸'}</button>
            </div>
            <div class="tooltip" data-tip="render">
              <button class="btn btn-sm text-[20px] btn-ghost" on:click={()=>renderSprite(0,frames-1)}>🎦</button>
            </div>
            <div class="tooltip" data-tip="download as zip">
              <button class="btn btn-sm text-[20px] btn-ghost" on:click={()=>saveZip()}>⏬</button>
            </div>
          </div>
        </Field>
        <Field title="camera angle">
          <div class="flex space-x-2">
            <button class="btn btn-xs {isBtnActive(26.5, modelAngle)}" on:click={()=> setModelAngle(26.5)}>26.5</button>
            <button class="btn btn-xs {isBtnActive(30, modelAngle)}" on:click={()=> setModelAngle(30)}>30</button>
            <button class="btn btn-xs {isBtnActive(45, modelAngle)}" on:click={()=> setModelAngle(45)}>45</button>
            <button class="btn btn-xs {isBtnActive(60, modelAngle)}" on:click={()=> setModelAngle(60)}>60</button>
            <input type="number" bind:value={modelAngle} on:change={()=>setModelAngle(modelAngle)} class="input input-xs input-bordered w-[60px]">
          </div>
        </Field>
        <Field title="render sides" value={frames}>
          <div class="flex space-x-2">
            <button class="btn btn-xs {isBtnActive(8,frames)}" on:click={()=> frames = 8}>8</button>
            <button class="btn btn-xs {isBtnActive(16,frames)}" on:click={()=> frames = 16}>16</button>
            <button class="btn btn-xs {isBtnActive(32,frames)}" on:click={()=> frames = 32}>32</button>
            <button class="btn btn-xs {isBtnActive(64,frames)}" on:click={()=> frames = 64}>64</button>
            <button class="btn btn-xs {isBtnActive(128,frames)}" on:click={()=> frames = 128}>128</button>
            <button class="btn btn-xs {isBtnActive(256,frames)}" on:click={()=> frames = 256}>256</button>
          </div>
        </Field>
        <Field title="camera" value="x: {camX} y: {camY}">
          <div class="flex justify-center">
            <button class="btn btn-sm btn-ghost" on:click={()=>setCamX(+0.05)}>⬅</button>
            <button class="btn btn-sm btn-ghost" on:click={()=>setCamX(-0.05)}>➡</button>
            <button class="btn btn-sm btn-ghost" on:click={()=>setCamY(-0.05)}>⬆</button>
            <button class="btn btn-sm btn-ghost" on:click={()=>setCamY(+0.05)} >⬇</button>
          </div>
        </Field>
        <Field title="zoom" value={camera.zoom}>
          <input type="range" step=".1" min="0" max="10" on:change={()=>{
            camera.updateProjectionMatrix ()
          }} bind:value={camera.zoom} class="range range-xs" />
        </Field>
        <Field title="fps" value={fps}>
          <input type="range" step="1" min="0" max="30" bind:value={fps} class="range range-xs" />
        </Field>
        <Field title="rotation offset" value={modelRotOffset}>
          <input type="range" step={360/frames} min="0" max="360" on:change={setModelRotOffset} bind:value={modelRotOffset} class="range range-xs" />
        </Field>
        <Field title="hemisphere light force" value={hemiLightForce}>
          <input type="range" step=".5" min="0" max="4" on:change={setHemiLight} bind:value={hemiLightForce} class="range range-xs" />
        </Field>
      </div>
      <div bind:this={gallery}></div>
    </div>
  </div>
</div>