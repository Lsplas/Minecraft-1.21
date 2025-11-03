# Minecraft-1.21
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Mini Minecraft 1.21 Browser</title>
<style>
  body { margin:0; overflow:hidden; font-family:sans-serif; background:#87CEEB;}
  #overlay { position:absolute; top:0; left:0; width:100%; height:100%; display:flex;
             flex-direction:column; justify-content:center; align-items:center; background:rgba(135,206,235,0.9);}
  button { margin:10px; padding:15px 30px; font-size:20px; }
  #hud { position:absolute; top:10px; left:10px; color:white; font-size:18px; }
  #toolbar { position:absolute; bottom:10px; left:50%; transform:translateX(-50%); display:flex; }
  .block-btn { width:50px; height:50px; margin:0 5px; border:2px solid white; cursor:pointer; }
</style>
</head>
<body>

<div id="overlay">
  <h1>Mini Minecraft 1.21</h1>
  <button id="singleplayerBtn">Singleplayer</button>
  <button id="downloadBtn">Download HTML</button>
</div>

<div id="hud"></div>
<div id="toolbar"></div>

<script src="https://cdn.jsdelivr.net/npm/three@0.154.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.154.0/examples/js/controls/PointerLockControls.js"></script>

<script>
let scene, camera, renderer, controls;
let worldBlocks = [];
let blockSize = 1;
let creativeMode = false;
let hud = document.getElementById('hud');
let selectedBlock = 1; // 0=black,1=green,2=white
const colors = [0x000000,0x00ff00,0xffffff];

// Download button
document.getElementById('downloadBtn').onclick = () => {
    const blob = new Blob([document.documentElement.outerHTML], {type:"text/html"});
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = "MiniMinecraft.html";
    a.click();
    URL.revokeObjectURL(url);
};

// Loading screen
document.getElementById('singleplayerBtn').onclick = () => showGameModeMenu();

function showGameModeMenu() {
    const overlay = document.getElementById('overlay');
    overlay.innerHTML = '<h2>Choose Game Mode</h2>';
    const survivalBtn = document.createElement('button');
    survivalBtn.innerText = 'Survival';
    survivalBtn.onclick = () => { creativeMode=false; startGame(); };
    const creativeBtn = document.createElement('button');
    creativeBtn.innerText = 'Creative';
    creativeBtn.onclick = () => { creativeMode=true; startGame(); };
    overlay.appendChild(survivalBtn);
    overlay.appendChild(creativeBtn);
}

function startGame() {
    document.getElementById('overlay').style.display='none';
    initToolbar();
    initThreeJS();
    animate();
}

function initToolbar() {
    const toolbar = document.getElementById('toolbar');
    colors.forEach((color,index)=>{
        const btn = document.createElement('div');
        btn.className='block-btn';
        btn.style.background = '#' + color.toString(16).padStart(6,'0');
        btn.onclick = ()=> selectedBlock=index;
        toolbar.appendChild(btn);
    });
}

function initThreeJS() {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x87CEEB);

    camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);

    renderer = new THREE.WebGLRenderer();
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    controls = new THREE.PointerLockControls(camera, document.body);
    document.body.onclick = ()=> controls.lock();
    camera.position.set(8,5,8);

    const light = new THREE.DirectionalLight(0xffffff,1);
    light.position.set(10,20,10);
    scene.add(light);
    scene.add(new THREE.AmbientLight(0xffffff,0.4));

    const geometry = new THREE.BoxGeometry(blockSize, blockSize, blockSize);

    // Create flat terrain
    for(let x=0;x<16;x++){
        for(let z=0;z<16;z++){
            const material = new THREE.MeshLambertMaterial({color: colors[1]});
            const cube = new THREE.Mesh(geometry, material);
            cube.position.set(x,0,z);
            cube.userData.type=1;
            scene.add(cube);
            worldBlocks.push(cube);
        }
    }

    document.addEventListener('keydown', keyDown);
    document.addEventListener('keyup', keyUp);
    document.addEventListener('mousedown', handleMouse);
}

let moveForward=false, moveBackward=false, moveLeft=false, moveRight=false, flyUp=false, flyDown=false;
function keyDown(e){
    switch(e.code){
        case 'KeyW': moveForward=true; break;
        case 'KeyS': moveBackward=true; break;
        case 'KeyA': moveLeft=true; break;
        case 'KeyD': moveRight=true; break;
        case 'Space': flyUp=true; break;
        case 'ShiftLeft': flyDown=true; break;
    }
}
function keyUp(e){
    switch(e.code){
        case 'KeyW': moveForward=false; break;
        case 'KeyS': moveBackward=false; break;
        case 'KeyA': moveLeft=false; break;
        case 'KeyD': moveRight=false; break;
        case 'Space': flyUp=false; break;
        case 'ShiftLeft': flyDown=false; break;
    }
}

// Place and break blocks
function handleMouse(e){
    const raycaster = new THREE.Raycaster();
    raycaster.setFromCamera(new THREE.Vector2(0,0), camera);
    const intersects = raycaster.intersectObjects(worldBlocks);
    if(intersects.length>0){
        const intersect = intersects[0];
        if(e.button===0){ // left click = break
            scene.remove(intersect.object);
            worldBlocks = worldBlocks.filter(b=>b!==intersect.object);
        } else if(e.button===2){ // right click = place
            const normal = intersect.face.normal;
            const pos = intersect.object.position.clone().add(normal);
            const geometry = new THREE.BoxGeometry(blockSize,blockSize,blockSize);
            const material = new THREE.MeshLambertMaterial({color:colors[selectedBlock]});
            const cube = new THREE.Mesh(geometry,material);
            cube.position.copy(pos);
            cube.userData.type=selectedBlock;
            scene.add(cube);
            worldBlocks.push(cube);
        }
    }
}

// Prevent context menu on right click
document.addEventListener('contextmenu', e=>e.preventDefault());

let velocityY=0;
function animate(){
    requestAnimationFrame(animate);
    const speed = creativeMode?0.2:0.1;
    const direction = new THREE.Vector3();
    if(moveForward) direction.z-=speed;
    if(moveBackward) direction.z+=speed;
    if(moveLeft) direction.x-=speed;
    if(moveRight) direction.x+=speed;
    controls.moveRight(direction.x);
    controls.moveForward(direction.z);

    if(creativeMode){
        if(flyUp) camera.position.y+=speed;
        if(flyDown) camera.position.y-=speed;
    }else{
        velocityY-=0.01;
        camera.position.y+=velocityY;
        if(camera.position.y<1.5){camera.position.y=1.5; velocityY=0;}
    }

    if(!creativeMode){
        hud.innerHTML="Health: ❤️❤️❤️❤️❤️<br>Hunger: 🍗🍗🍗🍗🍗";
    }else{
        hud.innerHTML="Creative Mode: Fly freely!";
    }

    renderer.render(scene,camera);
}
</script>
</body>
</html>
