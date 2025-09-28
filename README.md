# index-muatan.github.io
<!doctype html>
<html lang="id">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Preview WebAR Gaya Listrik</title>
  <style>
    body { margin:0; overflow:hidden; font-family:sans-serif; }
    #ui {
      position:absolute;top:10px;left:10px;z-index:10;
      background:rgba(255,255,255,0.85);padding:8px;border-radius:8px;
      width:260px;font-size:14px;
    }
  </style>
</head>
<body>
  <div id="ui">
    <b>Preview AR Gaya Listrik</b><br>
    <small>(Jika AR tidak tersedia, tampil dalam 3D biasa)</small><br><br>
    Muatan terpilih: <select id="chargeSelect"></select><br>
    Nilai q: <input id="qRange" type="range" min="-5" max="5" step="0.1" value="1">
    <span id="qVal">1</span> C<br>
    <button id="addPos">Tambah +</button>
    <button id="addNeg">Tambah -</button>
    <button id="remove">Hapus</button>
  </div>

  <script type="module">
  import * as THREE from "https://cdn.jsdelivr.net/npm/three@0.153.0/build/three.module.js";
  import { OrbitControls } from "https://cdn.jsdelivr.net/npm/three@0.153.0/examples/jsm/controls/OrbitControls.js";

  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(60, innerWidth/innerHeight, 0.01, 100);
  camera.position.set(0,0.6,2);

  const renderer = new THREE.WebGLRenderer({antialias:true});
  renderer.setSize(innerWidth, innerHeight);
  document.body.appendChild(renderer.domElement);

  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;

  const light = new THREE.HemisphereLight(0xffffff,0x444444,1);
  scene.add(light);

  const grid = new THREE.GridHelper(5,20);
  scene.add(grid);

  // charges
  const charges=[];
  const K=8.99e9;
  function addCharge(q,pos){
    const color = q>=0?0xff4444:0x4444ff;
    const mesh = new THREE.Mesh(
      new THREE.SphereGeometry(0.07,32,24),
      new THREE.MeshStandardMaterial({color})
    );
    mesh.position.copy(pos);
    scene.add(mesh);
    const arrow = new THREE.ArrowHelper(new THREE.Vector3(0,0,1),pos,0.1,0x00ff00);
    scene.add(arrow);
    const id="c"+Date.now()+Math.floor(Math.random()*100);
    charges.push({id,q,mesh,arrow});
    refreshSelect();
  }
  function refreshSelect(){
    const sel=document.getElementById("chargeSelect");
    const prev=sel.value;
    sel.innerHTML="";
    charges.forEach(c=>{
      const o=document.createElement("option");
      o.value=c.id;o.text=`${c.id} (q=${c.q})`;
      sel.appendChild(o);
    });
    sel.value=prev||charges[0]?.id;
    updateUI();
  }
  function getSelected(){
    const id=document.getElementById("chargeSelect").value;
    return charges.find(c=>c.id===id);
  }
  function updateUI(){
    const c=getSelected();if(!c)return;
    document.getElementById("qRange").value=c.q;
    document.getElementById("qVal").textContent=c.q;
  }

  // force calc
  function updateForces(){
    charges.forEach(a=>{
      let net=new THREE.Vector3();
      charges.forEach(b=>{
        if(a===b)return;
        const rvec=new THREE.Vector3().subVectors(b.mesh.position,a.mesh.position);
        const r=rvec.length(); if(r<0.01)return;
        const mag=(a.q*b.q)/(r*r);
        const dir=rvec.normalize().multiplyScalar(mag*1e-2);
        net.add(dir);
      });
      const len=net.length();
      if(len<0.001){a.arrow.setLength(0.001);}
      else{
        a.arrow.position.copy(a.mesh.position);
        a.arrow.setDirection(net.clone().normalize());
        a.arrow.setLength(Math.min(len,1));
        a.arrow.setColor(new THREE.Color(net.dot(new THREE.Vector3(1,0,0))<0?0x00ff00:0xff0000));
      }
    });
  }

  // init
  addCharge(1,new THREE.Vector3(-0.3,0,0));
  addCharge(-1,new THREE.Vector3(0.3,0,0));

  // UI events
  document.getElementById("addPos").onclick=()=>addCharge(1,new THREE.Vector3(0,0,0));
  document.getElementById("addNeg").onclick=()=>addCharge(-1,new THREE.Vector3(0,0,0));
  document.getElementById("remove").onclick=()=>{
    const c=getSelected();if(!c)return;
    scene.remove(c.mesh);scene.remove(c.arrow);
    charges.splice(charges.indexOf(c),1);
    refreshSelect();
  };
  document.getElementById("qRange").oninput=e=>{
    const c=getSelected();if(!c)return;
    c.q=parseFloat(e.target.value);
    c.mesh.material.color.setHex(c.q>=0?0xff4444:0x4444ff);
    document.getElementById("qVal").textContent=c.q;
    refreshSelect();
  };

  window.addEventListener("resize",()=>{
    camera.aspect=innerWidth/innerHeight;camera.updateProjectionMatrix();
    renderer.setSize(innerWidth,innerHeight);
  });

  function animate(){
    requestAnimationFrame(animate);
    controls.update();
    updateForces();
    renderer.render(scene,camera);
  }
  animate();
  </script>
</body>
</html>
