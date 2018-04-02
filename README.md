# framework-study-group
Iremos criar NOSSO framework do 0 para entender seu funcionamento


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
