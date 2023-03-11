# Tutorial de JSON Web Token con JavaScript y Node.js 

## JSON Web Token

#### Estructura
Los JSON Web Tokens generalmente están formados por tres partes: un encabezado o header, un contenido o payload, y una firma o signature. El encabezado identifica qué algoritmo fue usado para generar la firma y normalmente se ve de la siguiente forma:
```JSON
header = '{"alg":"HS256","typ":"JWT"}'
```
`HS256` indica que este token está firmado utilizando HMAC-SHA256.

El contenido contiene la información de los privilegios o claims del token:
```JSON
payload = '{"loggedInAs":"admin","iat":1422779638}'
```

El estándar sugiere incluir una marca temporal o timestamp en inglés, llamado `iat` para indicar el momento en el que el token fue creado. 

La firma está calculada codificando el encabezamiento y el contenido en base64url,  concatenándose ambas partes con un punto como separador: 
```JSON
key           = 'secretkey'
unsignedToken = encodeBase64Url(header) + '.' + encodeBase64Url(payload)
signature     = HMAC-SHA256(key, unsignedToken) 
```

En el token,  las tres partes -encabezado, contenido y firma- están concatenadas utilizando puntos de la siguiente forma: 
```JSON
token = encodeBase64Url(header) + '.' + encodeBase64Url(payload) + '.' + encodeBase64Url(signature) # token es: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJsb2dnZWRJbkFzIjoiYWRtaW4iLCJpYXQiOjE0MjI3Nzk2Mzh9.gzSraSYS8EXBxLN_oWnFSRgCzcmJmMjLiuyu5CSpyHI 
```

El token puede ser fácilmente transmitido en entornos HTML y HTTP, siendo similar a estándares basados en XML como SAML. Generalmente los algoritmos de cifrado utilizados son HMAC con SHA-256 (HS256) y firma digital con SHA-256 (RS256).

## Implementacion de autenticacion con JWT en un servidor con Node.js
### 1) Montar el servidor con express y las dependencias necesarias
`npm install express --save`
`npm install cors --save`
`npm install jsonwebtoken --save`
`npm install bcrypt --save`
### 2) Setup 
#### config
`mkdir config`
Creamos la variable que guarda la llave que genera los tokens en `jwt.js`
```javascript
module.exports = {
    key: 'UnaLlaveSecretaXD'
}
```

#### helpers
`mkdir helpers`
Helper que guarda mensajes para la API `messages.js`
```javascript
module.exports = {
    loginSuccessfully: {
        "MesageType": "Success",
        "Message": "Sucessfull login",
        "Component": "Token/Authentication",
        "Authenticated": true
    },
    tokenNotFound: {
        "MesageType": "Error",
        "Message": "Token is missing",
        "Component": "Token/Authentication",
        "Authenticated": false
    },
    invalidToken: {
        "MesageType": "Error",
        "Message": "The given token is invalid",
        "Component": "Token/Authentication",
        "Authenticated": false
    },
}
```

#### frontend
`mkdir frontend`
Carpeta con contenido de frontend `index.html`
```HTML
<!DOCTYPE html>
<html>
    <body>

        <h2>Index</h2>
        <p>This is frontend</p>

    </body>
</html>
```

### Middleware
El middleware actua como intermediario para decidir si un usuario tiene acceso a cierto contenido en base a la sesion determinada por un token.
```javascript
const express = require('express');
const jwt = require('jsonwebtoken');
const middleware = express.Router();

const config = require('../config/jwt')
var msg = require('../helpers/messages')

middleware.use((req, res, next) => {
    //Authorization
    const token = req.headers['x-access-token']
    if (token) {
        // We check if the token is legit
        const decode = jwt.verify(token, config.key, (err, decoded) => {
            if(err)
                return res.status(401).json(msg.invalidToken)
            else   
                next() // Executes code when properly authenticated
        })
        
    }else{
        return  res.status(401).send(msg.tokenNotFound); // Missing token
    }
})

module.exports = middleware
```
### Rutas
`mkdir routes`

#### api.js
```javascript
const { request, response, Router } = require('express');
var express = require('express');
var router = express.Router();

var u_controller = require('../controllers/user.controller'); 

// USERS
// Metodo de LOGIN en [login.js] (routes/login.js)
router.get('/user/', u_controller.GET);

module.exports = router;
```

#### index.js
```javascript
const { request, response } = require('express');
var express = require('express');
var router = express.Router();

router.get('/', (request, response) => {
    response.send('index');
});

module.exports = router;
```

#### login.js
```javascript
const { request, response } = require('express');
var express = require('express');
var router = express.Router();

var u_controller = require('../controllers/user.controller'); 

router.post('/', u_controller.LOGIN);

module.exports = router;
```

### Controladores
`mkdir controllers`

#### user.controller.js
Este controlador concentra lo que se comunica con la base de datos.
```javascript
const {
    response
} = require('express');

const config = require('../config/jwt');

const jwt = require('jsonwebtoken');
var msg = require('../helpers/messages')

const bcrypt = require('bcrypt');
const saltRounds = 10;


/**
 * [Function that returns an user from system]
 * @param request.body.username
 * @param response/error 
 */
module.exports.GET = (request, response) => {
    // Buscamos usuario
    let token = request.headers['x-access-token'];

    /* Sample user to avoid db */
    const sample_user = {
        "usertag": "DRAG010803MDFRVNA7",
        "birth": "03-08-2001:4500",
        "parent_usertag": "",
        "password": "$2a$10$EYzb8objNwuUoCHMwgQbQekCN6psiRQiMKKd7V0PmIBnKzl.qBHu2" // coco, 10 rounds
    }

    response.json({
        mensaje: "Get protegido",
        token: token,
        results: sample_user
    })
}

/**
 * [Function that authenticates an user into the system]
 * @param request.body.username
 * @param request.body.password 
 * @param response/error 
 */
module.exports.LOGIN = async (request, response) => {
    let bod = request.body;

    /* Sample user to avoid db */
    const sample_user = {
        "usertag": "DRAG010803MDFRVNA7",
        "birth": "03-08-2001:4500",
        "parent_usertag": "",
        "password": "$2a$10$EYzb8objNwuUoCHMwgQbQekCN6psiRQiMKKd7V0PmIBnKzl.qBHu2" // coco, 10 rounds
    }
    
    let error = false; // Error debe ser el resultado de encontrar o no un usuario
    if(error){
        response.status(500).send(error);
    } else {
        var isMatch = await (bcrypt.compare(bod.password, sample_user.password)) // Comparing bcrypt pwd
        if (isMatch) { // Successfull login
            const payload = {
                usertag: bod.usertag,
                birth: bod.birth,
                parent_usertag: bod.parent_usertag
            }
            let token = jwt.sign(payload, config.key, {
                expiresIn: 14400 // 4 hour token
            })
            response.json({
                mensaje: msg.loginSuccessfully,
                token: token,
                results: sample_user
            })
        } else response.status(403).json("Incorrect password"); // Incorrect password
    }
}
```

### index.js
```javascript
const express = require('express');
const middleware = require('./middleware/jwt-middleware');

const api = require('./routes/api');
const login = require('./routes/login');
const index = require('./routes/index');

const cors = require('cors');
const bodyParser = require('body-parser');

const host = '0.0.0.0';
const port = process.env.PORT || 3000;
const app = express(); 

app.use(cors());
// app.use(bodyParser.json({limit: '900mb'}));

app.use(bodyParser.urlencoded({
    extended: true
}));
app.use(bodyParser.json());

// Linking content from frontent folder
app.use(express.static(__dirname + '/frontend'));

// ROUTES
app.use('/api', middleware, api); // API middleware protected endpoints
app.use('/login', login); // Login page
app.use('', index); // Landing page

app.listen(port, () => {
    console.log('El servidor inicio en el puerto ' + port);
});
```

