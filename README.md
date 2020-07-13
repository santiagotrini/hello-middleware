# Hello Middleware

Un poco de teoría sobre como funciona Express. En esta guía vamos a explorar con un poco más de detalle el funcionamiento de ExpressJS y como crear una cadena de _middlewares_.

## ¿Qué es un middleware?

Tal como está definido en [la guía oficial](https://expressjs.com/es/guide/using-middleware.html).

> Express es un _framework_ para enrutamiento en la web con poca funcionalidad en sí mismo. Una aplicación de Express es esencialmente una serie de llamadas a funciones de _middleware_.

> Las funciones de _middleware_ son funciones que tienen acceso al objeto `req` (_request_), el objeto `res` (_response_) y al próximo _middleware_ en el ciclo de petición-respuesta HTTP de la aplicación. La próxima función de _middleware_ la nombramos con la variable `next`.

## Creamos una app

Lo de siempre.

```
$ mkdir hello-middleware
$ cd hello-middleware
$ npm init -y
$ npm i express morgan
$ npm i -D nodemon
```

Los últimos dos comandos merecen una explicación. `npm i` es la versión corta de `npm install`. En el primero instalamos Express por supuesto y también Morgan que es un _middleware_ que funciona como _logger_, nos permite generar registros (_logs_) del uso de nuestra app. Como Express es un _framework_ minimalista la funcionalidad de _logger_ se encuentra en un paquete de npm aparte, pero mantenido por la misma gente de Express.

En el último comando instalamos un paquete llamado nodemon, pero lo hacemos con `npm i -D` para indicar que se trata de una dependencia de desarrollo (en el `package.json` va a figurar en `devDependencies`). Este paquete me permite reiniciar mi app automáticamente cada vez que hagamos cambios en los archivos como `index.js`. Para usarlo agregamos un script al `package.json`.

```json
"scripts": {
  "start": "node index.js",
  "dev": "nodemon index.js"
}
```

Y vamos a ejecutar nuestra app desde la terminal con `npm run dev`.

## Middleware de aplicación

Los métodos `app.use` y `app.METHOD` (reemplacen METHOD por algún método HTTP) nos permiten usar _middleware_ en toda la app. Por ejemplo si queremos imprimir algo a consola cada vez que la app reciba una petición HTTP hacemos lo siguiente.

```js
const express = require('express');
const app = express();
app.use((req, res) => {
  console.log('Hola');
  res.end();
});
app.listen(3000);
```

Noten que la aplicación tiene que finalizar el ciclo de petición-respuesta. El _middleware_ en el ejemplo de arriba es la función dentro del `app.use()`. El esquema general de una app de Express es como sigue.
El cliente realiza una petición HTTP, la app de Express ejecuta una o más funciones de _middleware_ en el orden definido en el código y alguna de esas funciones, generalmente la última, termina el ciclo enviando una respuesta al cliente. En el ejemplo de arriba usamos `res.end()`, si nuestra aplicación no termina la petición, esta queda colgada bloqueando toda interacción entre cliente y servidor.

Podemos restringir las peticiones a las que respondemos utilizando una ruta como primer argumento de `app.use()`.

```js
const express = require('express');
const app = express();
app.use('/hola', (req, res) => {
  console.log('Hola');
  res.end();
});
app.listen(3000);
```

Noten que ahora no se ejecuta `console.log()` para `localhost:3000/` pero sí para `localhost:3000/hola` o `localhost:3000/hola/1`. Decimos que el _middleware_ está montado en la ruta `/hola`.

Por último, podemos restringir también por método HTTP, con `app.get()` por ejemplo.

```js
const express = require('express');
const app = express();
app.get('/hola', (req, res) => {
  console.log('Hola');
  res.end();
});
app.listen(3000);
```

Noten la diferencia con `app.use()` ahora la única ruta que ejecuta nuestra función es `localhost:3000/hola`, ya no vemos el `console.log()` para rutas debajo de esta como `localhost:3000/hola/1` o `localhost:3000/hola/hola`.

Las funciones de _middleware_ en todos los casos tienen dos argumentos que representan la petición y la respuesta (`req` y `res` tradicionalmente). En el caso de que querramos encadenar funciones tenemos un tercer argumento que en general llamamos `next`.

## Más de un middleware

Veamos en el código como es una cadena de _middleware_ para alguna ruta.

```js
const express = require('express');
const app = express();
const f = (req, res, next) => {
  console.log('voy primero');
  next()
};
const g = (req, res, next) => {
  console.log('voy segundo');
  next();
};
const h = (req, res, next) => {
  res.end();
};
app.get('/hola', f, g, h);
app.listen(3000);
```

Tenemos tres funciones `f()`, `g()` y `h()` que se ejecutan en el orden esperado. Para que esto suceda cada función tiene que o terminar la petición o pasarle el control a la próxima función en la cadena usando `next()`. Les dejo como ejercicio poner las funciones en el orden inverso y tratar de predecir que pasaría.

También podemos pasarle un array de funciones a `app.get()` como segundo argumento.

```js
const express = require('express');
const app = express();
const f = (req, res, next) => {
  console.log('voy primero');
  next()
};
const g = (req, res, next) => {
  console.log('voy segundo');
  next();
};
const h = (req, res, next) => {
  res.end();
};

let middlewares = [f, g, h];

app.get('/hola', middlewares);
app.listen(3000);
```

## Middleware de terceros

Un uso común de esta técnica es agregar algún _middleware_ de terceros que se ejecute para cualquier petición a la app. Por ejemplo podemos usar Morgan para logear las peticiones a la aplicación.

```js
const express = require('express');
const morgan  = require('morgan');
const app = express();
app.use(morgan('dev'));
app.get('/hola', (req, res) => {
  res.send('Hola');
});
app.listen(3000);
```

En la terminal del server vemos algo similar a

```
[nodemon] starting `node index.js`
GET /hola 200 4.427 ms - 4
```

Primero se ejecuta Morgan y después el `res.send()` que escribimos, pero si intercambiamos el orden así

```js
const express = require('express');
const morgan  = require('morgan');
const app = express();
app.get('/hola', (req, res) => {
  res.send('Hola');
});
app.use(morgan('dev'));
app.listen(3000);
```

vemos que no sale el _log_ porque la petición termina antes de llegar al _middleware_ de Morgan.

## Middleware incluído en Express

Express nos provee de funcionalidad mínima, pero hay unos pocos _middlewares_ ya incluídos en el _framework_. Uno de ellos es `express.json()` que sirve para parsear peticiones con cuerpos en formato JSON. Veamos un ejemplo.

```js
const express = require('express');
const app = express();
app.post('/hola', (req, res) => {
  console.log(`Hola, ${req.body.name}`);
  res.end();
});
app.listen(3000);
```

La idea acá es que hacemos una petición de tipo POST a la ruta `/hola` con el siguiente JSON en el cuerpo de la misma.

```json
{
  "name": "Juan"
}
```

Para testear esta ruta hacemos la siguiente petición con `curl` desde la terminal. No podemos hacer esto desde el navegador.

```
$ curl -X POST -H "Content-Type: application/json" -d '{"name":"Juan"}' http://localhost:3000/hola
```

Pero nos devuelve un error, usemos `express.json()` como _middleware_ antes de la función de nuestra ruta.

```js
const express = require('express');
const app = express();
app.use(express.json());
app.post('/hola', (req, res) => {
  console.log(`Hola, ${req.body.name}`);
  res.end();
});
app.listen(3000);
```

Ahora sí vemos `Hola, Juan` en la terminal, porque lo que hace `express.json()` es leer el JSON que viene el cuerpo de la petición y lo agrega dentro de `req.body`. Si no usamos este _middleware_ `req.body` vale `undefined`.

## Middleware de enrutador

Todo lo que vimos arriba aplica a instancias de `express()`, generalmente denotamos a esa instancia con la variable `app`. Lo mismo aplica a instancias de `express.Router()` como bien explica la guía a la que hice referencia al principio.

```js
const express = require('express')
const app = express();
const apiRouter = express.Router();
apiRouter.get('/users', (req, res) => {
  res.json({ msg: 'Todos los usuarios' });
});
app.use('/api', apiRouter);
app.listen(3000);
```

En el ejemplo de arriba la ruta a la que estamos respondiendo con el mensaje en JSON sería `localhost:3000/api/users`.

Agregando un _middleware_ a todas las peticiones que nos diga en que ruta estamos en la consola el código sería algo así.

```js
const express = require('express')
const app = express();
const apiRouter = express.Router();
const logger = (req, res, next) => {
  console.log(`estas en la ruta ${req.path}`);
  next();
};
apiRouter.get('/users', (req, res) => {
  res.json({ msg: 'Todos los usuarios' });
});
app.use(logger);
app.use('/api', apiRouter);
app.listen(3000);
```

## Conclusión

Con estos ejemplos no cubrimos todas las posibilidades pero espero que se den una idea más detallada de como funciona Express. Para más información diríjanse a la guía oficial.

No dejen de ver la próxima guía en [hello-mvc](https://github.com/santiagotrini/hello-mvc).
