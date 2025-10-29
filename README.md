**Detalhamento do nginx.conf**

**http** contexto principal para configurações relacionadas ao protocolo HTTP. Todas as configurações de servidores virtuais (server) e de balanceamento de carga (upstream) ficam aqui.

**upstream meus_sites** grupo de servidores para os quais as requisições serão distribuídas.

**server** ponto de contato (o load balancer em si). Redireciona as requisições para o grupo meus_sites.

**location** captura todas as requisições que chegam ao servidor na porta 4000. O / é um path genérico que corresponde a qualquer URL.

**proxy_pass** diretiva que vai receber as requisições e enviar para o grupo de servidores definidos no upstream que foi chamado de meus_sites


**Principais Comandos**

Inicia o servidor. Ao acessar localhost:8080 deverá ver uma mensagem Welcome to nginx
```
nginx
```

define um arquivo de configuração alternativo ao original (nginx.conf) da instalação 
```
nginx -c arquivo.conf 
```

para tudo
```
nginx -s stop
```

recarrega depois de alterar a configuração 
```
nginx -s reload
```

