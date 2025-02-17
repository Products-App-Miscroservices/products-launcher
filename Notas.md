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
9. En caso de clonar el repositorio del launcher las carpetas van a venir vacías, por lo que se debe correr el siguiente comando:

```bash
git submodule update --init --recursive
```

## Trabajar con Launcher
- Al hacer cambios en los repositorios a los que se tienen referencias se debe hacer lo siguiente:
  1. Hacer commit push de los cambios en cada repositorio.
  2. Hacer commit push en el launcher.
- Si se hace primero lo anterior en el launcher entonces va a generar errores.
  - En VS Code se recomienda gestionar el versionamiento. Si en el versionamiento al hacer click en uno de los archivos modificados y en el hash del commit aparece __dirty__, significa que no se han hecho los push en el repositorio del submodulo al que se hace referencia.
    - Asegurarse de no estar en la rama del hash cuando se trabaje con VS Code.
  - Cuando se hace al revés el paso, entonces se pierden referencias de los sub-módulos en el repositorio del launcher, lo cual provacará tener que resolver conflictos.

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

### Postgres
- Se puede tener ya una base de datos aprovisionada en ... o una imagen de docker. Por el momento, para trabajar en desarrollo se opta por docker.
- A diferencia de mongo que es una base de datos no relacional, en este caso sí se deben hacer migraciones

1. Crear __docker-compose.yaml__ en root de la aplicación.
  - En este caso, se crea en orders-ms

```yaml
version: '3'

services:
  orders-db:
    container_name: orders_database
    image: postgres:16.2
    restart: always
    volumes:
      - ./postgres:/var/lib/postgresql/data
    ports:
      - 5432:5432
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=123456
      - POSTGRES_DB=ordersdb
```

2. Crear contenedor.
```bash
docker compose up -d
```

3. Ignorar carpeta de volumen en __.gitignore__.
```.gitignore
postgres/
```

4. Anotar pasos en __README.md__.
```md


1. Clonar proyecto
2. Crear un archivo .env basado en .env.template
3. Levantar la base de datos con docker compose up -d
4. Levantar proyecto con npm run start:dev
```

5. Conectarse por medio de __Table Plus__.

![alt text](./Images/06-Docker-Postgres.png)

### Prisma - Modelo - conexión
- La documentación de prisma en este [link](https://docs.nestjs.com/recipes/prisma).
  - Son los mismos pasos que se anotaron anteriormente, con el único cambio de la cadena de conexión.
  - Por otro lado, en la creación del modelo se muestra la enumeración para prisma.

1. Instalar prisma.
```bash
npm i prisma --save-dev
```

2. Crear setuo de prisma inicial.
```bash
npx prisma init
```

3. Ajustar cadena de conexión dada en __.env__.
```.env
DATABASE_URL="postgresql://postgres:123456@localhost:5432/ordersdb?schema=public"
```

4. Instalar prisma/client.
```bash
npm i @prisma/client
```
5. Definir modelos en __src/prisma/schema.prisma__.
```prisma
// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema

// Looking for ways to speed up your queries, or scale easily with your serverless or edge functions?
// Try Prisma Accelerate: https://pris.ly/cli/accelerate-init

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum OrderStatus {
  PENDING
  DELIVERED
  CANCELLED
}

model Order {
  id String @id @default(uuid())
  totalAmount Float
  totalItems Int

  status OrderStatus
  paid Boolean @default(false)
  paidAt DateTime? // Se puede tener otra tabla con las ordenes que ya están pagadas para evitar tener que insertar null en db

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

```

- Al hacer la migración, en el cliente se tiene el tipado estricto de cómo se maneja en la db. Por esta razón OrderStatus se puede tomar desde prisma client.
```ts
import { OrderStatus } from "@prisma/client";
import { IsBoolean, IsEnum, IsNumber, IsOptional, IsPositive } from "class-validator";
import { OrderStatusList } from "../enum/order.enum";

export class CreateOrderDto {

    @IsNumber()
    @IsPositive()
    totalAmount: number;

    @IsNumber()
    @IsPositive()
    totalItems: number;

    // Al hacer la migración, en el cliente se tiene el tipado estricto de cómo se maneja en la db. Por esta razón OrderStatus se puede tomar desde prisma client.
    @IsEnum(OrderStatusList, {
        message: `Possible status values are ${OrderStatusList}`
    })
    @IsOptional()
    status: OrderStatus = OrderStatus.PENDING;

    @IsBoolean()
    @IsOptional()
    paid: boolean = false;
}


```

- Sin embargo, al momento de tipar en el DTO se debe crear una enumeración válida para poder usar __@IsEnum__.
  - Se crea en orders-md __src/orders/enum/order.enum.ts__ para centralizar las posibles enumeraciones
```ts
import { OrderStatus } from "@prisma/client";

export const OrderStatusList = [
    OrderStatus.PENDING,
    OrderStatus.DELIVERED,
    OrderStatus.CANCELLED,
];
```

- Por otro lado, al momento de pasar el DTO también al Gateway no se cuenta con prisma ahí. Entonces, también se coloca la carpeta y archivo de enum y se le modifica.
__orders/enum/order.enum.ts__
```ts
export enum OrderStatus {
    PENDING = 'PENDING',
    DELIVERED = 'DELIVERED',
    CANCELLED = 'CANCELLED'
}

export const OrderStatusList = [
    OrderStatus.PENDING,
    OrderStatus.DELIVERED,
    OrderStatus.CANCELLED,
];
```

6. Ejecutar migración.
```bash
npx prisma migrate dev --name init
```

7. Extender PrismaClient en servicio deseado.
```ts
@Injectable()
export class OrdersService extends PrismaClient implements OnModuleInit {

  private readonly logger = new Logger('OrdersService')

  async onModuleInit() {
    await this.$connect();
    this.logger.log('Database connected');
  }
  ...
}
```

### Dockerización
1. En el microservicio se crea un script en package.json
```json

    "start:dev": "npm run docker:start && nest start --watch",
    "start:debug": "nest start --debug --watch",
    "start:prod": "node dist/main",
    "docker:start": "prisma migrate dev && prisma generate",
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


# Auth Microservice
## Paquetes
```bash 
npm i bcrypt
npm i --save-dev @types/bcrypt
npm i --save @nestjs/jwt
```

1. Craer interfaz para definir payload de jwt.
__src\auth\interfaces\jwt-payload.ts__
```ts
export interface JwtPayload {
    id:string;
    email: string;
    username: string;
}
```
2. Definir variables de entorno:
  - JWT_SECRET

3. Importar módulo de jwt en auth.module.ts.
```ts
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { NatsModule } from 'src/transports/nats.module';
import { JwtModule } from '@nestjs/jwt';
import { envs } from 'src/config/envs';

@Module({
  controllers: [AuthController],
  providers: [AuthService],
  imports: [
    NatsModule,
    JwtModule.register({
      global: true,
      secret: envs.jwtSecret,
      signOptions: {expiresIn: '2h'}
    })
  ]
})
export class AuthModule {}

```

- De ahí se implementan los archivos creados en el curso, tales como los decoradores y los guards en __Gateway__.
  - Con el guard ya se verifica el token con el microservicio.
  - Con los decoradores se obtiene el usuario y el token ya que el guard se encargó de verificar el token y poner esa info en el request.
# Pendientes
1. __Products Microservice__. Guardar imágenes en cloudinary al crear un producto y al hacer update. __products.service.ts__.