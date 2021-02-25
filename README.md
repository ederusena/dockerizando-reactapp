# Dockerizando uma aplicação React JS



### Iniciando o projeto e as configurações de Desenvolvimento

```js
npx create-react-app dockerized-react-app
cd dockerized-react-app
```

Agora, adicione o seguinte arquivo *Dockerfile*, na raíz do projeto:

```dockerfile
# Imagem de Origem
FROM node:14-alpine
# Diretório de trabalho(é onde a aplicação ficará dentro do container).
WORKDIR /app
# Adicionando `/app/node_modules/.bin` para o $PATH
ENV PATH /app/node_modules/.bin:$PATH
# Instalando dependências da aplicação e armazenando em cache.
COPY package.json /app/package.json
RUN npm install --silent
RUN npm install react-scripts@3.3.1 -g --silent
# Inicializa a aplicação
CMD ["npm", "start"]
```



#### Adicione ao projeto o arquivo `.dockerignore` , dentro dele adicione o diretório `node_modules/`

Agora é hora de *buildar* a imagem, vamos fazer isso e adicionar uma tag (-t) para ela:

```bash
docker build -t sample:dev .
```

Assim que o comando acima finalizar a construção da imagem, vamos criar o container a partir dela:

```
docker run -v ${PWD}:/app -v /app/node_modules -p 3001:3000 --rm sample:dev
```

A porta que foi exportada para a máquina local foi a 3001 pelo parâmentro *-p.* Portanto para acessar a aplicação faz-se da seguinte maneira:

```
localhost:3001
```

A página inicial já deverá funcionar:

![Image for post](https://miro.medium.com/max/60/1*5x6c_J1CuGYvfE-oF45nZg.png?q=20)

![Image for post](https://miro.medium.com/max/1126/1*5x6c_J1CuGYvfE-oF45nZg.png)

Página inicial de uma aplicação React JS



O que está acontecendo até aqui?

1. O comando `docker run` criou a instância de um novo container a partir de uma imagem que foi criada através de um arquivo *Dockerfile*.
2. `-v ${PWD}:/app` monta/leva/move o código para dentro do container no diretório “/app”.
   *OBS: {PWD}, pode não funcionar no windows.*
3. O objetivo é utilizar o diretório `node_modules/` de dentro do container, por isso foi criado um segundo volume: `-v /app/node_modules` . Agora é possível remover o diretório local `node_modules/` do diretório do seu projeto local.
4. `-p 3001:3000` expõe a porta 3000 a outros containers do docker na mesma rede(para comunicação entre containers) e porta 3001 ao host.
   *Para mais informações veja esta tread no* [*Stackoverflow*](https://stackoverflow.com/questions/22111060/what-is-the-difference-between-expose-and-publish-in-docker)*.*
5. Por fim, `--rm` remove o container e os volumes, depois que o container for finalizado.



## Docker-compose



Vamos realizar a execução do container, mas agora usando um arquivo de receita do Docker, o `docker-compose.yml`. Crie e adicione este arquivo na raíz do projeto.

```yaml
version: '3.7'
services:
    app:
        container_name: dockerized-react-app
        build:
            context: .
            dockerfile: Dockerfile
        volumes:
            - '.:/app'
            - '/app/node_modules'
        ports:
            - '3001:3000'
        environment:
            - NODE_ENV=development
```

#### Crie a imagem e ative o container:

```
docker-compose up -d --build
```

##### Verifique se o aplicativo está rodando e test o `hot-reload` . Pare os containers antes de prosseguir:

```
docker-compose stop
```



# Produção

Primeiro iremos criar o Dockerfile que se chamará `Dockerfile-prod` . Crie na raíz do projeto:

```dockerfile
# build environment
FROM node:13-alpine as build
WORKDIR /app
ENV PATH /app/node_modules/.bin:$PATH
COPY package.json /app/package.json
RUN npm install — silent
RUN npm install react-scripts@3.0.1 -g — silent
COPY . /app
RUN npm run build

# production environment
FROM nginx:1.16.0-alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Usando o arquivo Dockerfile de produção, agora basta construir e atribuir uma tag a Imagem:

```
docker build -f Dockerfile-prod -t sample:prod .
```

Ative o container:

```
docker run -it -p 8080:80 --rm sample:prod
```

## Usando o docker-compose

Crie o arquivo `docker-compose-prod.yml` para ser usado em produção dentro da raíz do projeto:

```yaml
version: '3.7'
services:app-prod:
    container_name: dockerized-react-app
    build:
      context: .
      dockerfile: Dockerfile-prod
    ports:
      - '8080:80'
```

Ative o container usando o docker-compose:

```
docker-compose -f docker-compose-prod.yml up -d --build
```

*OBS: o parâmetro* `*-f*` *serve para especificar o arquivo a ser usado pelo docker-compose.*

# React Router and Nginx

Se o projeto estiver usando o [React Router](https://reacttraining.com/react-router/), será necessário alterar a configuração padrão do Nginx em tempo de execução durante a construção da Imagem, adicionando estas linhas:

```dockerfile
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx/nginx.conf /etc/nginx/conf.d
```

A nova imagem `Dockerfile-prod`será esta:

```dockerfile
# build environment
FROM node:13-alpine as build
WORKDIR /app
ENV PATH /app/node_modules/.bin:$PATH
COPY package.json /app/package.json
RUN npm install — silent
RUN npm install react-scripts@3.0.1 -g — silent
COPY . /app
RUN npm run build

# production environment
FROM nginx:1.16.0-alpine
COPY --from=build /app/build /usr/share/nginx/html
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx/nginx.conf /etc/nginx/conf.d
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Crie a seguinte estrutura de diretório na raíz do projeto:

```
└── nginx
    └── nginx.conf
```

O conteúdo do arquivo `nginx.conf` :

```sh
server {listen 80;location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
    try_files $uri $uri/ /index.html;
  }error_page   500 502 503 504  /50x.html;location = /50x.html {
    root   /usr/share/nginx/html;
  }}
```

# Scripts para Deploy

Primeiro é preciso criar este arquivo na raíz do projeto, vou chamá-lo de `run-app-deply.sh` . Agora basta adicionar a ele o seguinte conteúdo:

```sh
#!/bin/bashif [ $1 == "--dev" ]; then    echo "Iniciando ambiente de desenvolvimento..."    echo "Desconstruindo containers, caso existam..."
    docker-compose down    echo "Construindo containers de desenvolvimento..."
    docker-compose up -d --buildfiif [ $1 == "--prod" ]; then
    echo "Fazendo deploy em ambiente de Produção"
    
    echo "Desconstruindo containers, caso existam..."
    docker-compose -f docker-compose-prod.yml down    echo "Construindo containers de desenvolvimento"
    docker-compose -f docker-compose-prod.yml up -d --buildfi
```

Agora que o arquivo .sh está pronto vamos executá-lo.

Para iniciar o ambiente de desenvolvimento utilize:

```bash
./run-app-deploy.sh --dev
```

Para realizar o deploy em ambiente de desenvolvimento utilize:

```bash
./run-app-deploy.sh --prod
```

Se ocorrer algum problema para executar o arquivo `run-app-deploy.sh` , possivelmente será por conta da permissão, se for o caso, atribua ao arquivo a permissão para execução.

```bash
chmod +x run-app-deploy.sh
```