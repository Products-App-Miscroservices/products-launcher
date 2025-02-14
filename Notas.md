# Creación de organización para agrupar repositorios
https://www.udemy.com/course/nestjs-microservicios/learn/lecture/42560210#questions
1. Click sobre el perfir (ubicado en la parte superior derecha).
2. En el menú desplegable que aparece seleccionar __Your Organizations__.
3. Hacer click sobre botón __New organization__
4. Seleccionar plan gratuito.
	1. Nombrar organización. (Products-App-Miscroservices)
	2. Colocar email.
	3. Establecer que la organización pertenece a la cuenta propia.
5. En repositorios se crea uno nuevo para empezar a subir los repositorio

# Git SubModules, launcher
- [Gist](https://gist.github.com/Klerith/efdc4a8975c37643ebe68934bc28f426) dado en el curso.
1. Se deben tener los repositorios ya en GitHub.
  - En este caso se tienen en una organización.
2. Crear nuevo repositorio __products-launcher__ en la organización.
3. En máquna local colocar los archivos del launcher en una carpeta en donde se continuará trabajando.
  - En este caso los archivos son los docker y .env.
4. Hacer `git init`.
5. Realizar primer commit para poder subir el repositorio.
6. Subir repositorio.
7. Ejecutar siguiente comando para empezar a añadir referencias de otros repositorios.
  - El repository_url se puede obtener del boton de __Code__, en la sección __HTTPS__.
  - Este comando crea el archivo __.gitmodules__, el cual contiene las referencias al módulo y en general. Se le tiene que hacer commit también.
```bash
git submodule add <repository_url> <directory_name>

git submodule add https://github.com/Products-App-Miscroservices/gateway.git gateway
git submodule add https://github.com/Products-App-Miscroservices/products-microservice.git products-microservice
```

8. Al cargar todos los submodulos se hace commit y push al repositorio.



# Creación de proyecto (para gateway los recursos son REST API)
1. Crear carpeta en donde se colocaran todos los repositorios.
2. Crear nuevo proyecto con:
```bash
nest new products-ms
```
3. Borrar archivos que no se ocupan:
    1. app.controller.ts
    2. app.service.ts
    3. app.controller.spec.ts

4. Creación de recurso.
    - Crear recurso como microservicio (si el recurso credao es para el gateway entonces debe ser API).
    - No cread métodos CRUD.
```bash
nest g resource <nombre> --no-spec # no spec para que no genere archivos de test
ó
nest g res <nombre>
```

5. Instalar paquetes de microservicio y nats.
```bash
npm i --save @nestjs/microservices nats
```

6. Ajustar __main.ts__ para crear microservicio.
```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { Logger } from '@nestjs/common';
import { envs } from './products/config/envs';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';

async function bootstrap() {
  const logger = new Logger('Products Microservice');
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.NATS,
      options: {
        servers: envs.natsServers
      }
    }
  );

  await app.listen();
  logger.log(`Products Microservice running on port ${envs.port}`);
}
bootstrap();

```
6. Crear __src/config/services.ts__.
```ts
export const NATS_SERVICE = 'NATS_SERVICE'
```
7. Crear módulo de NATS __src/transports/nats.module.ts__.
```ts
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { envs } from 'src/config/envs';
import { NATS_SERVICE } from 'src/config/services';

@Module({
    imports: [
        ClientsModule.register([
            { name: NATS_SERVICE, 
              transport: Transport.NATS,
              options: {
                servers: envs.natsServers
              }
            },
        ])
    ],
    exports: [
        ClientsModule.register([
            { name: NATS_SERVICE, 
              transport: Transport.NATS,
              options: {
                servers: envs.natsServers
              }
            },
        ])
    ]
})
export class NatsModule {}

```

8. Ocupar módulo de Nats en módulo del recurso.
__products.module.ts__
```ts
import { Module } from '@nestjs/common';
import { ProductsService } from './products.service';
import { ProductsController } from './products.controller';
import { NatsModule } from 'src/transports/nats.module';

@Module({
  controllers: [ProductsController],
  providers: [ProductsService],
  imports: [NatsModule]
})
export class ProductsModule {}

```

## Variables de entorno
1. Instalar paquetes
```bash
npm i dotenv joi
```

2. Crear __src/config/envs.ts__.
3. Crear archivos __.env__ y __.env.template__

## Prisma
1. Instalar Prisma.
```bash
npm i prisma -D
```
### MongoDB
- [Documentación](https://www.prisma.io/docs/orm/overview/databases/mongodb) de prisma con mongoDB.
1. En cadena de conexión conseguida colocar al final la base de datos. (Se crea la base de datos en MongoDB Atlas).
2. Iniciar Prisma.
```bash
npx prisma init
```
3. Crear modelos en __prisma/schema.prisma__.
4. Generar cliente
    - Con MongoDB no se requieren hacer migraciones, ya que las bases de datos no relacionales son más volátiles.
```bash
npx prisma generate
```

5. Extender servicio que va a interactuar con la base de datos.
```ts
export class ProductsService extends PrismaClient implements OnModuleInit {

    private readonly logger = new Logger('Products Service')

    onModuleInit() {
        this.$connect();
        this.logger.log('MongoDb connected');
    }
}
```

## Paquetes para validaciones
1. Instalar paquetes.
```bash
npm i class-validator class-transformer
```

2. Establecer global pipes en __main.ts__.
```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { Logger, ValidationPipe } from '@nestjs/common';
import { envs } from './products/config/envs';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';

async function bootstrap() {
  const logger = new Logger('Products Microservice');
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.NATS,
      options: {
        servers: envs.natsServers
      }
    }
  );

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true
    })
  )

  await app.listen();
  logger.log(`Products Microservice running on port ${envs.port}`);
}
bootstrap();

```

# Creación de gateway
- Los pasos que se siguen son los mismos que para un proyecto normal, con la diferencia que los recursos son REST API, y en main no se define un microservicio, sino REST API.

- En los recursos solo se dejan los siguientes archivos, ya que no se requiere del servicio debido a que solo se llama al microservicio.
  - *.module.ts
  - *controller.ts

1. Ajustar __main.ts__
  1. Establecer prefijo global de __api__.
  2. Establecer global Pipes.
  3. Establecer global filters (para este caso se debe tener ya __common/exceptions/rpc-custom-exception.filters.ts__)

__main.ts__
```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { Logger, ValidationPipe } from '@nestjs/common';
import { envs } from './config/envs';
import { RpcCustomExceptionFilter } from './common/exceptions/rpc-custom-exception.filter';


async function bootstrap() {
  const logger = new Logger('Gateway');
  const app = await NestFactory.create(AppModule);

  app.setGlobalPrefix('api');

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
    })
  )

  app.useGlobalFilters(
    new RpcCustomExceptionFilter()
  )

  await app.listen(envs.port);

  logger.log(`Gateway runnin on port ${envs.port}`)
}
bootstrap();

```

__common/exceptions/rpc-custom-exception.filters.ts__
```ts
import { Catch, ArgumentsHost, ExceptionFilter } from '@nestjs/common';
import { RpcException } from '@nestjs/microservices';

@Catch(RpcException)
export class RpcCustomExceptionFilter implements ExceptionFilter {
  catch(exception: RpcException, host: ArgumentsHost) {
    // Se toma el contexto de ejecución
    const ctx = host.switchToHttp();

    // Se toma getResponse() del contexto de ejecución.
    const response = ctx.getResponse();

    const rpcError = exception.getError();

    if (rpcError.toString().includes('Empty response')) {
      return response.status(500).json({
        status: 500,
        message: rpcError.toString().substring(0, rpcError.toString().indexOf('(') - 1)
      })
    }

    if (
      typeof rpcError === 'object' &&
      'status' in rpcError &&
      'message' in rpcError
    ) {
      const status = isNaN(+rpcError.status!) ? 400 : +rpcError.status!;
      return response.status(status).json(rpcError);
    }

    response.status(400).json({
      status: 400,
      message: rpcError
    })

  }
}

```

2. Crear módulo de Nats como en sección anterior.
3. En cada recurso se debe colocar el módulo de Nats (esto si se trata de un microservicio al que se desea comunicarse).
__src/products/products.module.ts__
```ts
import { Module } from '@nestjs/common';
import { ProductsController } from './products.controller';
import { NatsModule } from 'src/transports/nats.module';

@Module({
  controllers: [ProductsController],
  providers: [],
  imports: [NatsModule]
})
export class ProductsModule { }


```

4. Coloca token de inyección ya sea en controlador o servicio.
  1. El token de inyección se debe definir en __src/config/services.ts__.

__products.controller.ts__
```ts
import { Controller, Inject } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';
import { NATS_SERVICE } from 'src/config/services';


@Controller('products')
export class ProductsController {
  constructor(
    @Inject(NATS_SERVICE) private readonly client: ClientProxy
  ) { }

}
```

__src/config/services.ts__
```ts
export const NATS_SERVICE = 'NATS_SERVICE';
```

# Nats
- Se debe tener el servidor de NATS corriendo, el cual se puede correr con docker compose o con el siguiente comando:
```bash
docker run -d --name nats-main -p 4222:4222 -p 6222:6222 -p 8222:8222 nats
```

- Puertos
  - 4222: Donde la comunicación de NATS sucede. Por acá van a hablar los microservicios.
  - 6222: Puerto que se recomienda sea usado para __clustering__. En el curso se omitió este puerto.
  - 8222: Puerto para gestión HTTP para información de reportaje.

# Dcokerización
1. Colocar docker-compose en root que contiene a todos los proyectos.
2. Colocar __.dockerignore__ y __dockerfile__ en cada repositorio.

## Repoisotrio que usa MongoDB y prisma
1. Colocar comando para generar cliente de prisma en __package.json__
2. Usar script cuando se ejecuta script de __npm run start:dev__.