# framework-study-group

Iremos criar NOSSO framework do 0 para entender seu funcionamento parte a parte.


## Rotas

Iremos nos basear no USO do Express, dessa forma:

```js
const user = require('./modules/User/routes')
app.use('/api/user', user)
```


```js
router.get('/', (req, res) => {
  res.json([1,2,3,4])
})
```


## HTTP - createServer

Para criarmos um servidor simples de rotas com o módulo `http` podemos fazer assim:

```js
const http = require('http');
const querystring = require('querystring');


const teste = {
    Name: 'Adelmo Junior',
    Age: 17
}

const server = http.createServer(function(req, res){
    res.writeHead(200, {"Content-Type": "application/json"});

    var url = req.url;
    var method = req.method;

    if(method === 'GET' && url === '/' ){
        res.writeHead(200, {"Content-Type": "application/json"});
        res.write(JSON.stringify(teste));
        res.end()
    }else{
        res.writeHead(404);
        res.write('<h1>ERROR</h1>');
    }

    if(method === 'POST' && url === '/post'){

        var data = '';

        req.on('data', function(body){

            data += body;

        });
        req.on('end', function(){
            res.writeHead(200, {"Content-Type": "application/json"});

            console.log(data)
            var value = {
                "sucess": true
            }
            res.end('OK');
        });
    }else{
        res.writeHead(404);
        res.end('<html><body>404</body></html>');
    }
});
server.listen(3000, function(){
    console.log('Server rodando de boas :D');
});
```

## Conceitos

Precisamos enteder como é o fluxo de uma rota nesse framework:

1) Chega uma requisição;
2) Testa qual método da requisição;
3) Testa qual a url da requisição;
4) Executa a ação definida pela rota.

Porém você precisa se questionar:

> Como eu defino essas rotas para serem testadas?

Ótima pergunta!


### Rotas - Definição

Sabemos que precisamos criar uma definição de rotas igual ao do Express:

```js
router.get('/', (req, res) => {
  res.json([1,2,3,4])
})
```

Logo precisamos criar um Objeto `router` que possui a função `get`, a qual recebe 2 parâmetros:

- path: url da rota;
- action: a função que precisa ser executada nessa rota.

Agora sabendo disso podemos criar o seguinte módulo:

```js
// lib/router.js
const router = {
  routes: [],
  get: (path, action) => {
    router.routes.push({
      method: 'GET',
      path,
      action
    })
  }
}

module.exports = router
```

Dessa forma nós já criamos um mecanismo de definição de rotas IGUAL ao do Express.

Olhe como iremos utilizar no nosso arquivo `routes.js`:

```js
// routes.js
const router = require('./lib/router')
const Controller = require('./controller')

const teste = [{
  Name: 'Adelmo Junior',
  Age: 17
}]

router.get('/', (req, res) => {
  Controller.find(req, res)(teste)
})
router.get('/123', (req, res) => {
  Controller.findOne(req, res)(teste)
})
console.log('router: ', router)
/**
router:  { routes: 
   [ { method: 'GET', path: '/', action: [Function] },
     { method: 'GET', path: '/123', action: [Function] } ],
  get: [Function: get] }
*/
module.exports = route
```

Logo mais chegarei nessa parte do Controller, por hora vamos nos focar nas rotas e para isso agora precisamos fazer o mecanismo que executa essas rotas.

Percebeu que essa definição das rotas apenas adiciona seus dados no *Array* `router.routes`, fiz dessa forma para facilitar o mecanismo de execução da rota desejada, então vamos entender como fazer isso.

### Rotas - Execução da Requisição

Primeiramente você precisa entender em que momento uma requisição nova chega no server, para isso vamos montar o seguinte `index.js` com a criação do *server HTTP* e vamos usar o evento `request` assim:

```js
// index.js
const http = require('http')
const server = http.createServer()

const app = require('./lib')
const routes = require('./routes')

// Essa é a forma explícita
// server.on('request', (req, res) => app.use(req, res)(routes))

// Essa é a forma implícita
server.on('request', app.use(routes))

server.listen( 3000, () => {
  console.log( 'Server rodando de boas :D' )
} )
```

Como sabemos que a chamada é `app.use(req, res)(routes)` logo inferimos que precisamos criar um módulo que seja um Objeto com a chave `use`, onde a qual é uma **CLOSURE** pois é uma função que retorna outra, facilmente notada pela sua execução: `(req, res)(routes)`.

Basicamente esse esqueleto:

```js
const teste = [
  {
    id: 0,
    name: 'Adelmo Junior',
    age: 17
  },{
    id: 1,
    name: 'Suisseba da Periferia',
    age: 33
  }
]

module.exports = {
  use: (router) => (req, res) => {

    const url = req.url
    const method = req.method.toLowerCase()

    switch (method) {
      case 'get': {
        switch (url) {
          case '/': {
            res.writeHead(200, { 'Content-Type': 'application/json' })
            res.write(JSON.stringify(teste))
            res.end()
            break;
          }
          case '/get': {
            res.writeHead(200, { 'Content-Type': 'application/json' })
            res.write(JSON.stringify(teste[0]))
            res.end()
            break;
          } 
          default: {
            res.writeHead(404, { 'Content-Type': 'application/json' })
            res.write(JSON.stringify({status: 'error', message: 'Rota não encontrada'}))
            res.end()
            break;
          }
        }
      break;
      }
      default: {
        res.writeHead(404, { 'Content-Type': 'application/json' })
        res.write(JSON.stringify({status: 'error', message: 'Rota não encontrada'}))
        res.end()
        break;
      }
    }
  }
}
```

Já entendemos como iremos testar e executar nossas rotas.

No primeiro `switch` nós testamos qual o método HTTP da requisição para cair no seu `case` correto para depois testar sua rota, porém como já possuímos o Objeto `router` que possui internamente um *Array* com as nossas rotas, logo nós devemos criar um mecanismo para essa validação e execução de uma forma mais genérica sem esse segundo switch que seria para cada rota.

Para solucionar esse problema iremos criar uma função genérica que será executada em qualquer requisição `GET` e com isso irá buscar o Objeto da rota correta e executar sua ação, essa foi minha solução simplista:

```js
const byPath = (url) => ({ path }) => path === url

const getRoutes = (router, method = 'get') => 
  router
    .routes
    .filter(route => route.method.toLowerCase() === method.toLowerCase())

switch (method) {
  case 'get': {
    getRoutes(router, 'get')
      .find(byPath(url))
      .action(req, res)
    break;
  }
  default:
    break;
}
```

Agora olhe como é o retorno de cada uma dessas funções:

```js
const byPath = (url) => ({ path }) => path === url
/**
{ method: 'GET', path: '/', action: [Function] }
*/

const getRoutes = (router, method = 'get') => 
  router
    .routes
    .filter(route => route.method.toLowerCase() === method.toLowerCase())
/**
[ 
  { method: 'GET', path: '/', action: [Function] },
  { method: 'GET', path: '/123', action: [Function] } 
]
*/

switch (method) {
  case 'get': {
    getRoutes(router, 'get')
      .find(byPath(url))
      .action(req, res)
    break;
  }
  default:
    break;
}
```

Com a `getRoutes` nós recebemos um *Array* com os Objetos da rotas que são do mesmo método, logo depois utilizo a função [find](http://mdn.io/find), entretando note como eu defini a função `byPath`:


```js
const byPath = (url) => ({ path }) => path === url
```

Porém a forma mais comum de se escrever isso é assim:

```js
const byPath = (url) => (route) => route.path === url
```

Bom como DEFINIMOS que esse é nosso padrão de configuração da rota, nós TEMOS CERTEZA que o Objeto que está contido no *Array* `routes` possui a seguinte estrutura imutável:

- method;
- path;
- action.

Eu usei `({ path }) => path === url` pois queria pegar apenas o valor dessa propriedade, isso foi possível graças à [Atribuição via desestruturação (destructuring assignment)](http://mdn.io/destructuring), já aproveitando o ensejo vamos criar uma validação para a requisição do `favicon.ico` para retornarmos `false`. 


```js
const byPath = (url) => ({ path }) => path === url

const getRoutes = (router, method = 'get') => 
  router
    .routes
    .filter(route => route.method.toLowerCase() === method.toLowerCase())

module.exports = {
  use: (router) => async (req, res) => {
    const url = req.url
                            
    const method = req.method.toLowerCase()
    
    if (url.includes('favicon.ico'))
      return false
    
    switch (method) {
      case 'get': {
        getRoutes(router, 'get')
          .find(byPath(url))
          .action(req, res)
        break;
      }
      default:
        break;
    }
  }
}
```

Além disso vamos trocar `getRoutes(router, 'get')` por `getRoutes(router, method)` para que dessa forma possamos extender esse código facilmente e de maneira genérica, por exemplo:

```js
const byPath = (url) => ({ path }) => path === url

const getRoutes = (router, method = 'get') => 
  router
    .routes
    .filter(route => route.method.toLowerCase() === method.toLowerCase())

module.exports = {
  use: (router) => async (req, res) => {
    const url = req.url
                            
    const method = req.method.toLowerCase()
    
    if (url.includes('favicon.ico'))
      return false
    
    switch (method) {
      case 'get': {
        getRoutes(router, method)
          .find(byPath(url))
          .action(req, res)
        break;
      }
      case 'post': {
        getRoutes(router, method)
          .find(byPath(url))
          .action(req, res)
        break;
      }
      default:
        break;
    }
  }
}
```

Percebeu que para criarmos para os métodos *PUT* e *DELETE* precisamos fazer apenas isso:


```js
const byPath = (url) => ({ path }) => path === url

const getRoutes = (router, method = 'get') => 
  router
    .routes
    .filter(route => route.method.toLowerCase() === method.toLowerCase())

module.exports = {
  use: (router) => async (req, res) => {
    const url = req.url
                            
    const method = req.method.toLowerCase()
    
    if (url.includes('favicon.ico'))
      return false
    
    switch (method) {
      case 'get': {
        getRoutes(router, method)
          .find(byPath(url))
          .action(req, res)
        break;
      }
      case 'post': {
        getRoutes(router, method)
          .find(byPath(url))
          .action(req, res)
        break;
      }
      case 'put': {
        getRoutes(router, method)
          .find(byPath(url))
          .action(req, res)
        break;
      }
      case 'delete': {
        getRoutes(router, method)
          .find(byPath(url))
          .action(req, res)
        break;
      }
      default:
        break;
    }
  }
}
```

<hr>

# ATENÇÃO!!!!


<hr>


> **O que você notou nesse código acima???**


Veja novamente apenas a parte que interessa:

```js
switch (method) {
  case 'get': {
    getRoutes(router, method)
      .find(byPath(url))
      .action(req, res)
    break;
  }
  case 'post': {
    getRoutes(router, method)
      .find(byPath(url))
      .action(req, res)
    break;
  }
  case 'put': {
    getRoutes(router, method)
      .find(byPath(url))
      .action(req, res)
    break;
  }
  case 'delete': {
    getRoutes(router, method)
      .find(byPath(url))
      .action(req, res)
    break;
  }
  default:
    break;
}
```


<br>
<br>
<br>

# SIM!!! Ele é quase TODO IGUAL!!!

<br>
<br>
<br>


Retirando os códigos duplicados ficamos com isso:

```js
const byPath = (url) => ({ path }) => path === url

const getRoutes = (router, method = 'get') => 
  router
    .routes
    .filter(route => route.method.toLowerCase() === method.toLowerCase())

module.exports = {
  use: (router) => async (req, res) => {
    const url = req.url
                            
    const method = req.method.toLowerCase()
    
    if (url.includes('favicon.ico'))
      return false
    
    return getRoutes(router, method)
      .find(byPath(url))
      .action(req, res)
  }
}
```

## Refatoração NERVOSA - Mas simples

Acompanhe comigo o seguinte, vamos retirar a definição das constantes internas e usar seu valor diretamente como visto abaixo:

```js
const byPath = (url) => ({ path }) => path === url

const getRoutes = (router, method = 'get') => 
  router
    .routes
    .filter(route => route.method.toLowerCase() === method.toLowerCase())

module.exports = {
  use: (router) => async (req, res) => {
    
    if (req.url.includes('favicon.ico'))
      return false
    
    return getRoutes(router, req.method.toLowerCase())
      .find(byPath(req.url))
      .action(req, res)
  }
}
```

Perceba que nossa função pode ser separada em 2 partes:

```js
if (req.url.includes('favicon.ico'))
  return false
```

```js
return getRoutes(router, req.method.toLowerCase())
  .find(byPath(req.url))
  .action(req, res)
```

Sabendo disso nós podemos FACILMENTE refatorar para um *IF* ternário, para deixarmos nossa função com **APENAS UMA FUCKING LINHA**:


```js
const byPath = (url) => ({ path }) => path === url

const getRoutes = (router, method = 'get') => 
  router
    .routes
    .filter(route => route.method.toLowerCase() === method.toLowerCase())

module.exports = {
  use: (router) => async (req, res) => 
    (req.url.includes('favicon.ico'))
      ? false
      : getRoutes(router, req.method.toLowerCase())
        .find(byPath(req.url))
        .action(req, res)
  }
}
```

## Router

Com isso podemos definir as outras funções do nosso `router`:

```js
// lib/router.js
const router = {
  routes: [],
  get: (path, action) => {
    router.routes.push({
      method: 'GET',
      path,
      action
    })
  },
  post: (path, action) => {
    router.routes.push({
      method: 'POST',
      path,
      action
    })
  },
  put: (path, action) => {
    router.routes.push({
      method: 'PUT',
      path,
      action
    })
  },
  delete: (path, action) => {
    router.routes.push({
      method: 'DELETE',
      path,
      action
    })
  }
}

module.exports = router
```

<br>

**OBVIAMENTE VOCÊ PERCEBEU QUE O CÓDIGO ESTÁ SE REPETINDO.** Logo nós **DEVEMOS** refatorar ele para encapsular sua lógica em uma função genérica para reusarmos, dessa forma:

```js
const addRoute = (router, method, path, action) => {
  router.routes.push({
    method,
    path,
    action
  })
}

const router = {
  routes: [],
  get: (path, action) => addRoute(router, 'GET', path, action),
  post: (path, action) => addRoute(router, 'POST', path, action),
  put: (path, action) => addRoute(router, 'PUT', path, action),
  delete: (path, action) => addRoute(router, 'DELETE', path, action)
}

module.exports = router
```

<br>
Dessa forma saimos de 33 linhas para 17! 

<br>

> **Quase a metade! Tá bom né?**


<br>
<br>

## Request


### Request - params

No Express quando definimos uma rota que aceita parâmetros precisamos disponibilizar esses parâmetros e seus valores, que vieram no `req.url`, como um Objeto dentro de `req.params`.

```js
router.get('/', (req, res) => {
})

router.get('/:id', (req, res) => {
})

router.get('/:id/:name', (req, res) => {
})
```

Primeiro precisamos pensar em como armazenar essas rotas pois eu não posso salva-las com esas urls, tendo em vista que nosso router iria procurar EXATAMENTE por essa rota e obviamente a url enviada estará com os valores que devem ser colocados nessas variáveis.

Imagine que teremos 3 requisições:

```
GET /
GET /1
GET /1/suissa
```

> Como que iremos tratar essas urls para que possamos buscar a rota correta?

A solução que pensei foi o seguinte:

Analisando as rotas podemos perceber que elas possuem quantidade de parâmetros diferentes, porém todas iniciam na `/`, sabendo disso nós podemos armazenar a url de cada rota apenas com `/`, todavia precisamos criar uma lógica para separar os parâmetros da url e depois colocar corretamente em `req.params` no momento que a requisição chegar no nosso servidor.

Nesse momento precisamos inferir qual a lógica para a separação desses parâmetros, então observe abaixo:

```js

> "/".split("/")
[ '', '' ]
> "/:id".split("/")
[ '', ':id' ]
> "/:id/:name".split("/")
[ '', ':id', ':name' ]

```

Depois dessa quebra da url vamos eliminar os valores vazios:

```js

> "/".split("/").filter(e => e !== '')
[]
> "/:id".split("/").filter(e => e !== '')
[ ':id' ]
> "/:id/:name".split("/").filter(e => e !== '')
[ ':id', ':name' ]

```

Já podemos colocar essa lógica em uma função:

```js

const getValuesFromURL = (path) => path.split('/').filter(e => e !== '')

```

Como queremos apenas o nome dos parâmetros precisamos eliminar o `:` com um `map`:

```js
> "/:id".split("/").filter(e => e !== '').map(p => p.replace(':', ''))
[ 'id' ]
> "/:id/:name".split("/").filter(e => e !== '').map(p => p.replace(':', ''))
[ 'id', 'name' ]
```

Passamos essa lógica para uma função a qual irá receber o RESULTADO da `getValuesFromURL`, por exemplo:


```js

const hasParams = getParams(getValuesFromURL(path))

```

Então criamos a função `getParams` que irá OU receber um *Array* com os parâmetros OU `false`:

```js

const getParams = (arrParams) => 
  (arrParams.length) 
    ? arrParams.map(p => p.replace(':', '')) 
    : false

```


```js
const addRoute = (router, method, path, action) => {
  const hasParams = getParams(getValuesFromURL(path))
  path = (hasParams) ? '/' : path
  
  router.routes.push({
    method,
    path,
    action,
    hasParams
  })
}
```

Note essa definição `path = (hasParams) ? '/' : path` que foi feita para definir a url da rota com `/`, pois caso possua parâmetros sua url será `/`, senão será o próprio `path` definido na rota.

Juntando tudo agora temos:


```js
// lib/router.js
const getParams = (arrParams) => 
  (arrParams.length) 
    ? arrParams.map(p => p.replace(':', '')) 
    : false
    
const getValuesFromURL = (path) => path.split('/').filter(e => e !== '')

const addRoute = (router, method, path, action) => {
  const hasParams = getParams(getValuesFromURL(path))
  path = (hasParams) ? '/' : path
  
  router.routes.push({
    method,
    path,
    action,
    hasParams
  })
}
const router = {
  routes: [],
  get: (path, action) => addRoute(router, 'GET', path, action),
  post: (path, action) => addRoute(router, 'POST', path, action),
  put: (path, action) => addRoute(router, 'PUT', path, action),
  delete: (path, action) => addRoute(router, 'DELETE', path, action)
}

module.exports = router
```

Para entendermos melhor como ficaram nossas rotas observe o log abaixo:

```js
router.get('/', (req, res) => {})
router.get('/:id', (req, res) => {})
router.get('/:id/:name', (req, res) => {})

/**
[ { method: 'GET', path: '/', action: [Function], hasParams: false },
  { method: 'GET',
    path: '/',
    action: [Function],
    hasParams: [ 'id' ] },
  { method: 'GET',
    path: '/',
    action: [Function],
    hasParams: [ 'id', 'name' ] 
  } 
]
*/
```

Depois disso precisamos criar o mecanismo que valida esses parâmetros q adiciona esse objeto no `req`, para isso iremos modificar nossa função do `use` para que depois que acharmos a rota nós criemos o `req.params` para que quando executarmos `route.action(req, res)` esse `req` esteja correto.

E quase sempre que quisermos criar um Objeto de forma dinâmica e usando dados de 2 lugares diferentes nós usaremos o `reduce`, além disso a nossa função do `reduce` precisa ser uma *closure*, pois como iremos iterar em um *Array*, que possui os valores dos parâmetros que vieram na url da requisição, precisamos também injetar o Objeto da rota para que possamos pegar também o nome definido para cada parâmetro:

```js

const params = getValuesFromURL(req.url)

const toParams = (route) => (obj, cur, i) => 
  Object.assign( obj, { [route.hasParams[i]]: cur } )

req.params = params.reduce(toParams(route), {})
// { id: '1', name: 'suissa' }
```


Substituindo nosso antigo `use` por esse novo:

```js
const getValuesFromURL = (path) => path.split('/').filter(e => e !== '')
const byPath = (url) => ({ path }) => path === url
const getRoutes = (router, method = 'get') => 
  router
    .routes
    .filter(route => route.method.toLowerCase() === method.toLowerCase())

module.exports = {
  use: (router) => async (req, res) => {

    if (req.url.includes('favicon.ico'))
      return false

    const route = getRoutes(router, req.method.toLowerCase())
                    .find(byPath(req.url))

    req.params = getValuesFromURL(req.url).reduce(toParams(route), {})
    return  route.action(req, res)
  }
}
```

```js
const getValuesFromURL = (path) => path.split('/').filter(e => e !== '')

const byPath = (url) => (route) => {
  const path = route.path
  const arrParams = getValuesFromURL(url)
  const hasParams = (arrParams.length) ? arrParams : false

  if (!route.hasParams) return (path === url) 
  
  return (hasParams) 
    ? (path === '/' && route.hasParams.length === hasParams.length)
    : (path === url) 
}

const getRoutes = (router, method = 'get') => {
  return router
    .routes
    .filter(route => route.method.toLowerCase() === method.toLowerCase())
}

const toParams = (route) => (obj, cur, i) => {
  return Object.assign( obj, { [route.hasParams[i]]: cur } )
}
module.exports = {
  use: (router) => async (req, res) => {
    if (req.url.includes('favicon.ico'))
      return false

    const route = getRoutes(router, req.method.toLowerCase())
        .find(byPath(req.url))

    const params = getValuesFromURL(req.url)

    req.params = (!route.hasParams) 
      ? { }
      : params.reduce(toParams(route), {})

    return  route.action(req, res)
  }
}
```


### Request - query


### Request - body