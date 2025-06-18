   <meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Sistema Orgânico Funcional</title>

<!-- ===== CSS ===== -->
<style>
  body { margin: 0; font-family: Arial, Helvetica, sans-serif; background: #fffafa }
  header { background: #f28b82; color: #fff; padding: 1rem; text-align: center }

  /* Toolbar */
  .toolbar { display: flex; flex-wrap: wrap; justify-content: center; gap: .5rem; padding: 1rem; background: #ffecec }
  .toolbar button, .toolbar select {
    padding: .5rem 1rem;
    font-size: 1rem;
    border: none;
    border-radius: 5px;
    background: #f28b82;
    color: #fff;
    cursor: pointer;
  }
  .toolbar select { background: #fff; color: #333 }

  /* Opções globais */
  .toggle-options { text-align: center; margin: .5rem 0 }
  .toggle-options label { margin: 0 .5rem }

  /* Contêiner com scroll */
  .org-container {
    display: block;
    overflow-x: auto;
    overflow-y: hidden;
    white-space: nowrap;
    padding: 2rem 0;
    text-align: center;
    -webkit-overflow-scrolling: touch;
  }

  /* Estrutura em árvore */
  .tree { display: inline-block; text-align: center; white-space: normal }

  /* Nó */
  .node {
    background: #fff;
    padding: 1rem;
    margin: 1rem auto;
    border-radius: 10px;
    border: 2px solid #f28b82;
    box-shadow: 0 0 10px rgba(242, 139, 130, .2);
    min-width: 150px;
    transition: background .3s;
  }
  .node:hover { background: #fff0f0 }
  .node input {
    font-size: 1rem;
    border: none;
    border-bottom: 1px solid #ccc;
    width: 100%;
    text-align: center;
    background: transparent;
  }

  /* Botões de ação */
  .actions { margin-top: .5rem }
  .actions button {
    background: #f8a4a4;
    border: none;
    border-radius: 5px;
    padding: .3rem .6rem;
    margin: .15rem;
    color: #fff;
    cursor: pointer;
    font-size: .85rem;
  }

  /* Conexão de filhos */
  .children {
    display: flex;
    justify-content: center;
    margin-top: 2rem;
  }
</style>

<!-- dom‑to‑image -->
<script src="https://cdn.jsdelivr.net/npm/dom-to-image-more@3.3.0/dist/dom-to-image-more.min.js"></script>

<header>
  <h1>Sistema Orgânico Funcional</h1>
</header>

<!-- ===== Toolbar ===== -->
<div class="toolbar">
  <button onclick="toggleEdit()">Modo Edição</button>
  <button onclick="downloadSVG()">Exportar SVG</button>

  <select onchange="filterLevel(this.value)">
    <option value="">Todos os níveis</option>
    <option value="0">Sistema</option>
    <option value="1">Subsistema</option>
    <option value="2">Secção</option>
    <option value="3">Subsecção</option>
    <option value="4">Série</option>
    <option value="5">Subsérie</option>
    <option value="6">Documento Composto</option>
    <option value="7">Documento Simples</option>
  </select>
</div>

<!-- Opções globais -->
<div class="toggle-options">
  <label><input type="checkbox" id="enableSubsistema"> Permitir Subsistema</label>
  <label><input type="checkbox" id="enableSubserie"> Permitir Subsérie</label>
</div>

<!-- Contêiner do organograma -->
<div id="organograma" class="org-container"></div>

<!-- ===== JavaScript ===== -->
<script>
/* ---------- Estado ---------- */
let editing = false;
const data = { name: 'Sistema', children: [] };

/* ---------- Renderização ---------- */
function renderTree(node, level = 0){
  const container=document.createElement('div');
  container.className='tree';

  /* Nó visual */
  const nodeDiv=document.createElement('div');
  nodeDiv.className='node';
  nodeDiv.dataset.level=level;

  const nameInput=document.createElement('input');
  nameInput.value=node.name;
  nameInput.disabled=!editing;
  nameInput.oninput=()=>node.name=nameInput.value;
  nodeDiv.appendChild(nameInput);

  const actions=document.createElement('div');
  actions.className='actions';

  if(editing){
    /* Botão genérico de sub‑nível */
    const generic=document.createElement('button');
    generic.textContent='+ Subnível';
    generic.onclick=()=>{
      node.children.push({name:getDefaultName(node.name),children:[]});
      render();
    };
    actions.appendChild(generic);

    /* Sistema */
    if(node.name==='Sistema'){
      addButton(actions,node,'Secção');
      addButton(actions,node,'Subsistema',()=>document.getElementById('enableSubsistema').checked);
    }
    /* Secção */
    if(node.name==='Secção'){
      addButton(actions,node,'Subsecção');
      addButton(actions,node,'Série');
      addButton(actions,node,'Documento Composto');
      addButton(actions,node,'Documento Simples');
    }
    /* Subsecção */
    if(node.name==='Subsecção'){
      addButton(actions,node,'Série');
      addButton(actions,node,'Documento Composto');
      addButton(actions,node,'Documento Simples');
    }
    /* Série */
    if(node.name==='Série'){
      addButton(actions,node,'Subsérie',()=>document.getElementById('enableSubserie').checked);
      addButton(actions,node,'Documento Composto');
      addButton(actions,node,'Documento Simples');
    }
    /* Subsérie */
    if(node.name==='Subsérie'){
      addButton(actions,node,'Documento Composto');
      addButton(actions,node,'Documento Simples');
    }

    /* Remover nó */
    const del=document.createElement('button');
    del.textContent='🗑️';
    del.onclick=()=>{
      const parent=findParent(data,node);
      if(parent){
        parent.children=parent.children.filter(c=>c!==node);
        render();
      }
    };
    actions.appendChild(del);
  }

  nodeDiv.appendChild(actions);
  container.appendChild(nodeDiv);

  /* Filhos */
  if(node.children?.length){
    const kids=document.createElement('div');
    kids.className='children';
    node.children.forEach(c=>kids.appendChild(renderTree(c,level+1)));
    container.appendChild(kids);
  }
  return container;
}

/* ---------- Helpers ---------- */
function addButton(actions,node,type,condition=()=>true){
  const btn=document.createElement('button');
  btn.textContent='+ '+type;
  btn.onclick=()=>{
    if(condition()){
      node.children.push({name:type,children:[]});
      render();
    }
  };
  actions.appendChild(btn);
}
function getDefaultName(parent){
  switch(parent){
    case 'Sistema':
    case 'Subsistema': return 'Secção';
    case 'Secção': return 'Subsecção';
    case 'Subsecção': return 'Série';
    case 'Série': return 'Subsérie';
    case 'Subsérie': return 'Documento Composto';
    case 'Documento Composto': return 'Documento Simples';
    default: return 'Novo';
  }
}
function findParent(root,target,parent=null){
  if(root===target) return parent;
  if(root.children){
    for(const child of root.children){
      const res=findParent(child,target,root);
      if(res) return res;
    }
  }
  return null;
}

/* ---------- UI ---------- */
function render(){
  const wrap=document.getElementById('organograma');
  wrap.innerHTML='';
  wrap.appendChild(renderTree(data));
}
function toggleEdit(){ editing=!editing; render(); }
function filterLevel(level){
  document.querySelectorAll('.node').forEach(n=>{
    n.style.display = (!level || n.dataset.level===level) ? 'block' : 'none';
  });
}

/* ---------- Exportar SVG ---------- */
function downloadSVG(){
  const tree=document.querySelector('#organograma .tree');
  if(!tree){ alert('Nada para exportar'); return; }

  const w=tree.scrollWidth;
  const h=tree.scrollHeight;

  domtoimage.toSvg(tree,{width:w,height:h})
    .then(dataUrl=>{
      const link=document.createElement('a');
      link.href=dataUrl;
      link.download='organograma.svg';
      link.click();
    })
    .catch(err=>console.error('Erro ao gerar SVG:',err));
}

/* ---------- Inicialização ---------- */
render();
</script>
