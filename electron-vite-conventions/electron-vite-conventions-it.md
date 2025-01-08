# CONVENZIONI ELECTRON + VITE

<br/>

Un progetto electron può diventare davvero complesso con il passare del tempo,
soprattutto quando si utilizzano API o servizi esterni che richiedono molteplici operazioni
diverse, per questo è importante suddividere logicamente un progetto in principio
e renderlo il più modulare possibile, queste sono alcune direttive che mi sono 
dato lavorando nel corso degli anni su progetti electron che diventavano sempre più grandi.

Electron divide il progetto in 3 parti distinte, ma non dice come ottimizzarle, queste parti sono:

<img src="folder-tree.png"/>

<br/>

* **main**: è il cuore del programma, dove gira node 
* **preload**: è il punto in cui definiamo tutte le funzioni di node che possono essere chiamate arbitrariamente dal front-end
* **renderer**: è l'area in cui costruiamo il front-end e consumiamo le funzioni del preload


<br/>
<br/>

# 🛠 Ecco una proposta per ordinare ognuna di queste 3 sezioni 

<br/>
<br/>

# main

Il main deve essere diviso come un MVC, simile alla struttura di un server express, divisa quindi in controllers separati, che gestiscono isolatamente la logica, separando in blocchi a seconda del servizio, ecco un esempio

```javascript
// index.js

import { listUsers } from 'controllers/msql'

ipcMain.handle("msql:list_users:req",listUsers)
```

```javascript
// controllers/msql.js

export function listUsers(){ 
    return msql.findAll();
}
```

### Se dobbiamo prendere i dati mandati dal front?
I dati vengono passati automaticamente alla funzione, il primo è sempre un oggetto "event" aggiunto da electron 

```javascript
// index.js

import { addUser } from 'controllers/msql'

ipcMain.handle("msql:add_user:req",(e,user)=>addUser(user))
```

```javascript
// controllers/msql.js

export function addUser( user ){ 

    const users = msql.addOne(user);

    return {success:true}
}
```

<br/>
<br/>

# preload

Anche il preload va suddiviso in base ai servizi

```javascript
// preload/index.js

import { contextBridge } from 'electron'
import { electronAPI } from '@electron-toolkit/preload'

import msql from 'msql' // la nostra logica per msql
import api from 'api' // la nostra logica per api


contextBridge.exposeInMainWorld('msql', msql)
contextBridge.exposeInMainWorld('api', api)
```

```javascript
// preload/msql.js (valido anche per api.js)

import { electronAPI } from '@electron-toolkit/preload'

const msql = {
    listUsers:() => electronAPI.ipcRenderer.invoke('msql:list_users:req')
}

export default msql;
```

### nel front-end adesso possiamo chiamare i servizi in questo modo

```javascript
// app.js

const msqlResponse = window.msql.listUsers();
```


<br/>

# renderer

Per ottenere rapidamente indietro i dati elaborati da node nella sezione main, possiamo usare invoke, che permette di ritornare un risultato immediatamente, dovrebbe essere il metodo preferito quando ci aspettiamo una risposta, in più bisognerebbe aggiungere un modo per gestire la risposta o l'errore, solo quando ci serve, ad esempio una CALLLBACK PREVENTIVA


### Qui creiamo una callback preventiva <br/> in modo da poter gestire tutto nel front-end
```javascript
// preload/msql.js

import { electronAPI } from '@electron-toolkit/preload'

const msql = {
    listUsers:(cb,errCb) => electronAPI // passiamo le 2 callback
    .ipcRenderer
    .invoke('msql:list_users:req') // usiamo invoke al posto di send

    // gestiamo gli eventi con le callback
    .then(res=>cb(res))
    .catch(res=>erCb(err))
}

export default msql;
```

### adesso finalmente nel front-end possiamo richiamare la funzione così
#### entrambe le callback sono opzionali, quindi decidiamo noi se e quale usare
```javascript
// app.js


function handleSuccess(res){
    // ..code
}

function handleError(err){
    // ..code
}

window.msql.listUsers( handleSuccess, handleError )
```
