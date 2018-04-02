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

