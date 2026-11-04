
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Canyon Circuit</title>
<style>
*{box-sizing:border-box}
body{margin:0;overflow:hidden;background:#111;color:white;font-family:Arial}
canvas{display:block}
#hud{position:absolute;top:20px;left:20px;right:20px;display:flex;justify-content:space-between;pointer-events:none}
.logo{font-size:25px;font-weight:900;letter-spacing:4px}
.logo span{color:#ff9e27}
.panel{background:#111b;padding:12px 18px;border:1px solid #ffffff33}
#speed{position:absolute;bottom:25px;right:30px;font-size:55px;font-weight:900}
#speed small{font-size:15px;color:#ff9e27}
#help{position:absolute;bottom:25px;left:25px;line-height:1.8;color:#ddd}
#menu{position:absolute;inset:0;background:#080d12dd;display:flex;align-items:center;justify-content:center}
.box{background:#101820;border:1px solid #ff9e27;padding:40px;max-width:500px;width:90%}
h1{font-size:48px;margin:10px 0}
h1 span{color:#ff9e27}
button{background:#ff9e27;border:0;padding:16px 28px;font-weight:bold;font-size:16px;cursor:pointer}
#message{position:absolute;top:25%;width:100%;text-align:center;font-size:40px;font-weight:bold}
</style>
</head>
<body>
<div id="hud">
 <div class="logo">CANYON <span>CIRCUIT</span></div>
 <div class="panel">LAP <span id="lap">1</span>/3<br>POSITION <span id="pos">1</span>/6</div>
</div>
<div id="speed">000 <small>KM/H</small></div>
<div id="help">WASD / ARROWS — DRIVE<br>SPACE — DRIFT<br>SHIFT — NITRO<br>C — CAMERA<br>R — RESET</div>
<div id="message"></div>

<div id="menu">
 <div class="box">
  <h1>CANYON <span>CIRCUIT</span></h1>
  <p>Six racers. Three laps. One massive canyon.</p>
  <p>Master the corners, use nitro and beat the competition.</p>
  <button id="start">START RACE →</button>
 </div>
</div>

<script type="importmap">
{"imports":{"three":"https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js"}}
</script>

<script type="module">
import * as THREE from 'three';

const scene=new THREE.Scene();
scene.background=new THREE.Color(0x92b8ce);
scene.fog=new THREE.Fog(0x92b8ce,180,600);

const camera=new THREE.PerspectiveCamera(70,innerWidth/innerHeight,.1,1000);
const renderer=new THREE.WebGLRenderer({antialias:true});
renderer.setSize(innerWidth,innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio,2));
renderer.shadowMap.enabled=true;
renderer.shadowMap.type=THREE.PCFSoftShadowMap;
renderer.toneMapping=THREE.ACESFilmicToneMapping;
document.body.appendChild(renderer.domElement);

scene.add(new THREE.HemisphereLight(0xffffff,0x65402a,2));

const sun=new THREE.DirectionalLight(0xffe0ae,3);
sun.position.set(-100,150,80);
sun.castShadow=true;
sun.shadow.mapSize.set(2048,2048);
scene.add(sun);

const material=(color,metalness=0,roughness=.8)=>
 new THREE.MeshStandardMaterial({color,metalness,roughness});

// DESERT
const ground=new THREE.Mesh(
 new THREE.PlaneGeometry(1200,1200),
 material(0x966344)
);
ground.rotation.x=-Math.PI/2;
ground.position.y=-2;
ground.receiveShadow=true;
scene.add(ground);

// CIRCUIT
const RX=175,RZ=115,N=500,WIDTH=17;

function point(t,lane=0){
 const x=RX*Math.cos(t);
 const z=RZ*Math.sin(t);
 const y=4+7*Math.sin(t*2)+2*Math.sin(t*5);

 const dx=-RX*Math.sin(t);
 const dz=RZ*Math.cos(t);
 const len=Math.hypot(dx,dz);

 return new THREE.Vector3(
 x+dz/len*lane,
 y,
 z-dx/len*lane
 );
}

function ribbon(a,b,mat,y=0){
 const vertices=[],indices=[];

 for(let i=0;i<=N;i++){
  const t=i/N*Math.PI*2;
  const p1=point(t,a);
  const p2=point(t,b);

  vertices.push(p1.x,p1.y+y,p1.z);
  vertices.push(p2.x,p2.y+y,p2.z);

  if(i<N){
   let k=i*2;
   indices.push(k,k+1,k+2,k+1,k+3,k+2);
  }
 }

 const geo=new THREE.BufferGeometry();
 geo.setAttribute('position',
  new THREE.Float32BufferAttribute(vertices,3));
 geo.setIndex(indices);
 geo.computeVertexNormals();

 const mesh=new THREE.Mesh(geo,mat);
 mesh.receiveShadow=true;
 scene.add(mesh);
}

ribbon(-WIDTH/2,WIDTH/2,material(0x303438),.2);
ribbon(-WIDTH/2-.6,-WIDTH/2,material(0xf2d39c),.25);
ribbon(WIDTH/2,WIDTH/2+.6,material(0xf2d39c),.25);

// CANYON WALLS
const rocks=[
 material(0x99502f),
 material(0xb36a42),
 material(0x70412f)
];

for(let i=0;i<240;i++){
 const t=Math.random()*Math.PI*2;
 const lane=(Math.random()<.5?-1:1)*(28+Math.random()*25);
 const p=point(t,lane);

 const h=25+Math.random()*65;
 const w=12+Math.random()*25;

 const rock=new THREE.Mesh(
  new THREE.ConeGeometry(w,h,6),
  rocks[i%3]
 );

 rock.position.set(p.x,h/2-1,p.z);
 rock.rotation.y=Math.random()*6;
 rock.scale.z=.8+Math.random();
 rock.castShadow=true;
 scene.add(rock);
}

// CAR FACTORY
function makeCar(color){
 const group=new THREE.Group();

 function part(w,h,d,x,y,z,col,metal=0){
  const mesh=new THREE.Mesh(
   new THREE.BoxGeometry(w,h,d),
   material(col,metal,.35)
  );
  mesh.position.set(x,y,z);
  mesh.castShadow=true;
  group.add(mesh);
  return mesh;
 }

 part(2.3,.7,4.3,0,.8,0,color,.5);
 part(1.7,.8,1.8,0,1.4,.2,0x183344,.6);
 part(1.7,.12,1.5,0,1.85,.2,0x252a30,.4);

 const wheels=[];

 for(const x of [-1.2,1.2]){
  for(const z of [-1.4,1.4]){
   const wheel=new THREE.Mesh(
    new THREE.CylinderGeometry(.47,.47,.32,16),
    material(0x101010)
   );
   wheel.rotation.z=Math.PI/2;
   wheel.position.set(x,.5,z);
   wheel.castShadow=true;
   group.add(wheel);
   wheels.push(wheel);
  }
 }

 part(2.1,.14,.4,0,1.3,1.85,0x171717);
 part(2.3,.2,.2,0,.65,-2.15,0xffc54d);

 scene.add(group);

 return {group,wheels};
}

// PLAYER
const player=makeCar(0xff8b22);

// AI
const colors=[0x168cdd,0xe33c40,0xe5d83d,0x8a57d9,0x2eaf74];

const rivals=colors.map((color,i)=>({
 car:makeCar(color),
 progress:-.04-i*.014,
 speed:37+Math.random()*8,
 lane:(i%2?1:-1)*4
}));

let progress=0;
let speed=0;
let lane=0;
let nitro=100;
let lap=1;
let racing=false;
let raceTime=0;
let cameraMode=0;

const keys={};

addEventListener('keydown',e=>{
 keys[e.key.toLowerCase()]=true;
 if(e.key.toLowerCase()==='c')cameraMode=(cameraMode+1)%3;
 if(e.key.toLowerCase()==='r')reset();
});

addEventListener('keyup',e=>keys[e.key.toLowerCase()]=false);

function reset(){
 progress=0;
 speed=0;
 lane=0;
 nitro=100;
 lap=1;
}

function placeRacers(){
 const p=point(progress,lane);
 player.group.position.set(p.x,p.y+.5,p.z);

 player.group.rotation.y=
 -Math.atan2(-RX*Math.sin(progress),RZ*Math.cos(progress));

 rivals.forEach(r=>{
  const q=point(r.progress,r.lane);
  r.car.group.position.set(q.x,q.y+.5,q.z);
  r.car.group.rotation.y=
   -Math.atan2(-RX*Math.sin(r.progress),RZ*Math.cos(r.progress));
 });
}

placeRacers();

const clock=new THREE.Clock();

function animate(){
 requestAnimationFrame(animate);

 const dt=Math.min(clock.getDelta(),.04);

 if(racing){
  raceTime+=dt;

  const accelerating=keys.w||keys.arrowup;
  const braking=keys.s||keys.arrowdown;
  const left=keys.a||keys.arrowleft;
  const right=keys.d||keys.arrowright;
  const drifting=keys[' '];
  const boosting=keys.shift&&nitro>0;

  if(accelerating)speed+=35*dt;
  else speed-=7*dt;

  if(braking)speed-=45*dt;

  speed=THREE.MathUtils.clamp(
   speed,0,boosting?88:65
  );

  if(boosting&&speed>5){
   speed=Math.min(88,speed+35*dt);
   nitro=Math.max(0,nitro-25*dt);
  }else{
   nitro=Math.min(100,nitro+8*dt);
  }

  const steering=(right?1:0)-(left?1:0);

  lane+=steering*speed/65*dt*
   (drifting?12:7);

  lane=THREE.MathUtils.clamp(
   lane,-WIDTH*.85,WIDTH*.85
  );

  const curveLength=Math.sqrt(
   (RX*Math.sin(progress))**2+
   (RZ*Math.cos(progress))**2
  );

  progress+=speed/curveLength*dt;

  if(progress>=Math.PI*2){
   progress-=Math.PI*2;
   lap++;

   if(lap>3){
    racing=false;
    document.getElementById('message').textContent=
     'RACE COMPLETE!';
   }
  }

  // AI RACING LOGIC
  rivals.forEach(r=>{
   const target=progress-.025;
   const correction=THREE.MathUtils.clamp(
    (target-r.progress)*7,-3,5
   );

   r.speed=THREE.MathUtils.clamp(
    r.speed+correction*dt,32,57
   );

   const length=Math.sqrt(
    (RX*Math.sin(r.progress))**2+
    (RZ*Math.cos(r.progress))**2
   );

   r.progress+=r.speed/length*dt;

   if(r.progress>=Math.PI*2)
    r.progress-=Math.PI*2;
  });

  placeRacers();

  player.wheels.forEach(w=>{
   w.rotation.x-=speed*dt*1.5;
  });

  rivals.forEach(r=>r.car.wheels.forEach(w=>{
   w.rotation.x-=r.speed*dt*1.5;
  }));

  document.getElementById('speed').innerHTML=
   String(Math.round(speed*2.4)).padStart(3,'0')+
   ' <small>KM/H</small>';

  document.getElementById('lap').textContent=
   Math.min(lap,3);

  const positions=[progress,...rivals.map(r=>r.progress)]
   .sort((a,b)=>b-a);

  document.getElementById('pos').textContent=
   positions.indexOf(progress)+1;

  const forward=new THREE.Vector3(0,0,-1)
   .applyQuaternion(player.group.quaternion);

  let desired,look;

  if(cameraMode===2){
   desired=player.group.position.clone()
    .add(new THREE.Vector3(0,1.6,0))
    .add(forward.clone().multiplyScalar(1));

   look=player.group.position.clone()
    .add(forward.clone().multiplyScalar(30));
  }else{
   const distance=cameraMode===1?18:12;

   desired=player.group.position.clone()
    .add(forward.clone().multiplyScalar(-distance))
    .add(new THREE.Vector3(0,6,0));

   look=player.group.position.clone()
    .add(forward.clone().multiplyScalar(15))
    .add(new THREE.Vector3(0,1,0));
  }

  camera.position.lerp(desired,1-Math.pow(.001,dt));
  camera.lookAt(look);
 }

 renderer.render(scene,camera);
}

animate();

// MENU
document.getElementById('start').onclick=()=>{
 document.getElementById('menu').style.display='none';
 document.getElementById('message').textContent='GO!';
 setTimeout(()=>{
  document.getElementById('message').textContent='';
 },1000);

 reset();
 rivals.forEach((r,i)=>{
  r.progress=-.04-i*.014;
 });
 raceTime=0;
 racing=true;
};

// RESIZE
addEventListener('resize',()=>{
 camera.aspect=innerWidth/innerHeight;
 camera.updateProjectionMatrix();
 renderer.setSize(innerWidth,innerHeight);
});
</script>
</body>
</html>
