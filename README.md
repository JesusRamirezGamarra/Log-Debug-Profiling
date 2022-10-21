<p align="center">
  <p align="center">    
    <img src="https://github.com/JesusRamirezGamarra/signature/blob/main/public/img/Logo_Negro.png" alt="BFFs" height="250">    
  </p>
  <p align="center">
       CoderHouse - Backend
  </p>
</p>


# Log-Debug-Profiling
Loggers, gzip y analaisis de Perfomance

> # Consigna:
- Luego implementar loggueo (con alguna librería vista en clase) que registre lo siguiente:
  * Ruta y método de todas las peticiones recibidas por el servidor (info) 

- Fecha y hora - Método - URL 
  * Ruta y método de las peticiones a rutas inexistentes en el servidor (warning)
  * Errores lanzados por las apis de mensajes y productos, únicamente (error)
  
- Considerar el siguiente criterio:
  * Loggear todos los niveles a consola (info, warning y error)
  * Registrar sólo los logs de warning a un archivo llamada warn.log
  * Enviar sólo los logs de error a un archivo llamada error.log

> # Consigna: Luego, realizar el análisis completo de performance del servidor con el que venimos trabajando.
Vamos a trabajar sobre la ruta '/info', en modo fork, agregando ó extrayendo un console.log de la información colectada antes de devolverla al cliente. Además desactivaremos el child_process de la ruta '/randoms'
Para ambas condiciones (con o sin console.log) en la ruta '/info' OBTENER:
1) El perfilamiento del servidor, realizando el test con --prof de node.js. Analizar los resultados obtenidos luego de procesarlos con --prof-process. 
2) Utilizaremos como test de carga Artillery en línea de comandos, emulando 50 conexiones concurrentes con 20 request por cada una. Extraer un reporte con los resultados en archivo de texto.


# Solucion : Gzip
a. Utlizamos la libreria Compression como middleware, unicamnete para el method : `/info`

```javascript
import { logger } from '../utils/logger.js';
router.get('/info',compression(), async(req, res,) => {
   
    const data = infodelProceso
    res.render('info', {data})
    
})

```


# Solucion : Loggers
a. para utilizar el logger sobre el path `\info` utilizamos :
```javascript
import { logger } from '../utils/logger.js';
router.get('/info',compression(), async(req, res,) => {
    
    logger.info(`RUTA: ${req.path} || Mehotd: ${req.method}`)
    const data = infodelProceso
    res.render('info', {data})
    
})

```

creando un `logger.js`  que utilizaremos de utils
```javascript
import winston from 'winston'

    const logConfig = {
        level: 'info',
        format: winston.format.json(),
        //format: winston.format.simple(),
    }

    
export const logger = winston.createLogger(logConfig);
```


b. para invocar al logger  para toda solicitud ( path y/o method) implementamos :

```javascript
app.use( (req,res) => {
  
  let fyh = new moment().format('DD/MM/YYYY HH:mm:ss')
  let url =  req.protocol + '://' + req.get('host') + req.originalUrl;
  logger.warn(`${fyh}  || PATH: ${req.path} || Mehotd: ${req.method} || status: Page Not Fount || URL: ${url }`)

  res.status(404).render('404')

})

```

c. Para invocar al logger para toda excepcion implementamos : 

```javascript
const isLoggedIn = (req, res, next) => {
    try{
        if(!req.isAuthenticated())  res.redirect('/login') 
        else    next();    
    }
    catch(err){
        logger.error(`${new moment().format('DD/MM/YYYY HH:mm:ss')} || PATH: ${req.path} || METHOD: ${req.method} || ERROR: ${err.message}`);
    }        
}
```
logica `try/catch` que implementamos en todo method existente .


agregamos sobre  `logger.js`  los transports que nos permitiran grabar en archivos la informacion solicitadda.
```javascript
import winston from 'winston'

    const logConfig = {
        level: 'info',
        format: winston.format.json(),
        //format: winston.format.simple(),
        transports: [
            new winston.transports.Console({ 
                level: 'info' 
            }),
            new winston.transports.File({
                            filename: './logs/warnings.log',
                            level: 'warn',
            }),
            new winston.transports.File({
                            filename: './logs/errors.log',
                            level: 'error',
            }),
        ],
    }

    
export const logger = winston.createLogger(logConfig);
```

# Solucion : Perfommance / Profiling

a. Realizamos la configuracion para utilizar FORK

Agregamops sobre `package.json`

```json
"dev": "nodemon ./src/index.js -e=DEV -p=8080 -m=FORK"
```

Agregamos la logica para determinar el MODO de ejecucion `FORK`
```javascript
import config from './config/config.js';
import Server from './services/server.js';
import { initDB_CallBack } from './services/db.js'


import cluster from 'cluster'
import minimist from 'minimist';
import os from 'os'

const numCPUs = os.cpus().length;

if(config.init.MODO === 'CLUSTER' && cluster.isPrimary){

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork()
  }
  cluster.on('exit', (worker) => {
    console.log(`Worker ${worker.process.pid} died at ${Date()}`);
    cluster.fork();
  });
}
else {
  const init = async () => {
    initDB_CallBack()
    const server = Server.listen(config.init.PORT, () => {
      console.log(`Servidor escuchando en el puerto ${config.init.PORT} - worker process with ${process.pid} started , ready : http://localhost:${config.init.PORT}/`);
    });
    server.on('error', (error) => console.log(`Error en servidor: ${error}`));
  }
  init()

}

```

Ejecutamos :
```bash
npm run dev --MODE=DEV --PORT=8080 --MODO=FORK
```

a .  considerando el uso de `artillery` y sus parametros , La herramienta tiene el comando «quick» que nos permite lanzar un test rápido contra la URL que definamos, pudiendo establecer los siguientes modificadores::  [ver mas](https://www.adictosaltrabajo.com/2018/02/22/tests-de-rendimiento-con-artillery/)

```config
-d, –duration <número de segundos>
-r, –rate <número de segundos para el ratio de llegada de peticiones>
-p, –payload <cadena con el Payload de una petición POST>
-t, –content-type <cadena con el content type>
-o, –output <cadena con el path del archivo de salida>
-k, –insecure <deshabilita la verificación del certificado TLS>
-q, –quiet <establece el modo «quiet» con lo que no se muestra ningún log>
De esta forma si queremos simular 10 usuarios enviando 20 peticiones cada uno, simplemente tenemos que ejecutar:
```

CORRER COMANDO NPM RUN start:profilling


```bash
artillery quick --count 10 -n 50 "http://localhost:8081/info" > result_nobloq.txt
```

```bash
artillery quick --count 10 -n 50 "http://localhost:8081/info" > result_bloq.txt
```

b. 

autocannon

Correr el comando: npm run start:autocannon
 y el script:  autocannon -c 100 -d 15 'http://localhost:8081/info'

 los resultados de las pruebas estan en la carpeta /docs




Pre requisitos : 

Crear : `.env.development` o `.env.production` segun se requiera.

```
MODE=[PRODUCTION|DEVELOPMENT]
PORT=3000
DEBUG=false
MONGO_URL=[URL de MONGODB]
```
NOTA : 
- logger winston : [ver mas](https://github.com/winstonjs/winston)
- momentsJS ( manejador de Fechas) : [ver mas](https://momentjs.com/)