# Projeto  (Load Balance com Nodejs + NGINX + Docker)

Tudo o que precisaremos para testar essa arquitetura é o Docker. Como as instâncias de nosso aplicativo Node.js e NGINX serão executadas dentro de contêineres Docker, não precisaremos instalá-las em nossa máquina de desenvolvimento. Para instalar o Docker podemos consultar aqui:

https://docs.docker.com/install/linux/docker-ce/ubuntu/

Nodejs

Como criar um aplicativo simples do Node.js que serve um arquivo HTML, o contêiner com o Docker e uma instância do NGINX que usa o algoritmo round-robin para balancear a carga entre duas instâncias em execução desse aplicativo.

Para criar o diretório, vamos emitir o seguinte comando: mkdir application. Depois disso, vamos criar o index.js arquivo nesse diretório e colar o seguinte código-fonte:

var http = require('http');

var fs = require('fs');

http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/html'});
  res.end(`<h1>${process.env.MESSAGE}</h1>`);
}).listen(8080);


Supondo que você já tenha instalado o Node.js , crie um diretório para armazenar seu aplicativo e torne esse diretório ativo.

$ mkdir myapp
$ cd myapp

Use o npm initcomando para criar um package.jsonarquivo para seu aplicativo. Para obter mais informações sobre como package.jsonfunciona, consulte Específicos do manuseio package.json do npm .

$ npm init

Esse comando solicita várias coisas, como o nome e a versão do seu aplicativo. Por enquanto, você pode simplesmente pressionar RETURN para aceitar os padrões da maioria deles, com a seguinte exceção:

entry point: (index.js)

Digite app.js, ou o que você quiser, o nome do arquivo principal. Se você quiser index.js, tecle RETURN para aceitar o nome do arquivo padrão sugerido.

Agora instale o Express no myappdiretório e salve-o na lista de dependências. Por exemplo:

$ npm install express --save

Execute o aplicativo com o seguinte comando:

$ node app.js
Em seguida, carregue http://localhost:3000/ em um navegador para ver a saída para testar.


"Dockerizando" os aplicativos Node.js

Precisaremos criar um arquivo chamado Dockerfile no application diretório. O conteúdo deste arquivo será:

FROM node
RUN mkdir -p /usr/src/app
COPY index.js /usr/src/app
EXPOSE 8080
CMD [ "node", "/usr/src/app/index" ]

docker build -t load-balanced-app .

Podemos executar as duas instâncias do nosso aplicativo com os seguintes comandos:

docker run -e "MESSAGE=First instance" -p 8081:8080 -d load-balanced-app
docker run -e "MESSAGE=Second instance" -p 8082:8080 -d load-balanced-app

Abrir ambas as instâncias em um navegador da Web, indo para http://localhost:8081 e http://localhost:8082. O primeiro URL mostrará uma mensagem dizendo "Primeira instância", o segundo URL mostrará uma mensagem dizendo "Segunda instância".


Balanceamento de Carga com uma Instância NGINX com Docker

Agora que temos as duas instâncias do nosso aplicativo sendo executadas em diferentes contêineres do Docker e respondendo em portas diferentes em nossa máquina host, vamos configurar uma instância do NGINX para carregar as solicitações de load balance entre elas. Primeiro vamos começar criando um novo diretório.

mkdir nginx-docker

Neste diretório, criaremos um arquivo chamado nginx.confcom o seguinte código:

upstream my-app {
    server 172.17.0.1:8081 weight=1;
    server 172.17.0.1:8082 weight=1;
}

server {
    location / {
        proxy_pass http://my-app;
    }
}

Este arquivo será usado para configurar o NGINX. Nele, definimos um upstream grupo de servidores contendo os dois URLs que respondem pelas instâncias de nosso aplicativo. Ao não definir nenhum algoritmo específico para solicitações de balanceamento de carga, estamos usando round-robin, que é o padrão no NGINX. Existem várias outras opções para solicitações de balanceamento de carga com o NGINX, por exemplo, o menor número de conexões ativas ou o menor tempo médio de resposta .

Depois disso, definimos a configuração do NGINX para passar solicitações HTTP http://my-app, que é manipulado pelo upstream definido anteriormente. Além disso, observe que codificamos 172.17.0.1como IP de gateway, esse é o gateway padrão ao usar o Docker. Se necessário, você pode alterá-lo para atender sua configuração local.

Agora vamos criar o Dockerfile que será usado para "dockerizar" o NGINX com esta configuração. Este arquivo conterá o seguinte código:

FROM nginx
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf

Tendo criado os dois arquivos, agora podemos construir e executar o NGINX em contêineres no Docker. Conseguimos isso executando os seguintes comandos:

docker build -t load-balance-nginx .
docker run -p 8080:80 -d load-balance-nginx

Depois de executar esses comandos, vamos abrir um navegador da web e acessar http://localhost:8080. Se tudo correr bem, veremos uma página web com uma das duas mensagens: First instance e Second instance. Se acertarmos a recarga em nosso navegador algumas vezes, perceberemos que de tempos em tempos a mensagem exibida alterna entre First instance e Second instance. Este é o algoritmo de balanceamento de carga round-robin em ação.



