# framework-study-group

Iremos criar NOSSO framework do 0 para entender seu funcionamento parte a parte.


## Rotas

Iremos nos basear no USO do Express, dessa forma:

```js
const user = require('./modules/User/routes')
app.use('/api/user', user)
```


```js
router.get('/', (req, res, next) => {
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
router.get('/', (req, res, next) => {
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

router.get('/', (req, res, next) => {
  Controller.find(req, res)(teste)
})
router.get('/123', (req, res, next) => {
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


```js

const server = http.createServer()

server.on('request', (req, res) => {
  router.run(req, res)
})

server.listen( 3000, () => {
  console.log( 'Server rodando de boas :D' )
} )

```
