# Load balancer com nginx

Kit de treinamento para subir, na sua própria máquina, dois sites HTML e um load balancer na frente deles. O usuário acessa uma URL única e o nginx decide qual dos dois sites responde.

```
+-----------+              +-----------------+              +---------------------+
|           |    :4000     |                 |    :5001     | Servidor 1  (site1) |
| Navegador | -----------> |  Load balancer  | -----------> | página azul         |
|           |              |     (nginx)     |              +---------------------+
+-----------+              |                 |    :5002     +---------------------+
                           |                 | -----------> | Servidor 2  (site2) |
                           +-----------------+              | página amarela      |
                                                            +---------------------+
```

Tudo roda em um único processo do nginx: ele é o load balancer na porta 4000 e, ao mesmo tempo, o servidor web de cada site nas portas 5001 e 5002. Em um ambiente real cada caixa do desenho seria uma máquina diferente. Aqui está tudo junto para você focar no balanceamento.

## O que você vai fazer

1. Entender como o `nginx.conf` monta o load balancer e os dois sites.
2. Instalar o nginx.
3. Apontar o `nginx.conf` para as pastas `site1` e `site2`.
4. Subir tudo com um comando.
5. Ver, no navegador e no terminal, a alternância entre os servidores.
6. Trocar o algoritmo de balanceamento e observar o efeito.

## Por que existe um load balancer

Quando um site cresce, um servidor só não dá conta. A solução é ter várias cópias do site em servidores diferentes e colocar um **load balancer** na frente. Ele recebe todas as requisições em um único endereço e as distribui entre as cópias. Para o usuário nada muda: ele continua acessando uma URL só.

O algoritmo padrão de distribuição é o **Round-Robin**: a primeira requisição vai para o servidor 1, a segunda para o servidor 2, a terceira volta para o 1, e assim por diante. Nos exercícios você vai testar outros algoritmos.

## O que tem neste repositório

| Arquivo | Para que serve |
|---|---|
| `nginx.conf` | Configuração do load balancer e dos dois servidores, explicada na próxima seção. É o arquivo que você vai editar. |
| `site1/index.html` | Página azul que identifica o servidor 1. |
| `site2/index.html` | Página amarela que identifica o servidor 2. |

## Entendendo o nginx.conf

O arquivo descreve três servidores dentro de um único processo do nginx. A melhor forma de entender é seguir o caminho que uma requisição percorre, de cima para baixo. Os trechos abaixo foram tirados do `nginx.conf`, sem os comentários. Leia esta seção antes de mexer no arquivo.

### A estrutura geral

```nginx
events {
}

http {
    upstream meus_sites { ... }     # a lista de servidores que recebem o tráfego

    server { listen 4000; ... }     # o load balancer
    server { listen 5001; ... }     # o site1
    server { listen 5002; ... }     # o site2
}
```

Tudo que importa está dentro de `http { }`, o contexto onde o nginx trata o protocolo HTTP. O bloco `events { }` controla como o nginx lida com conexões simultâneas e pode ficar vazio, porque os valores padrão servem.

Cada bloco `server { }` é um **servidor virtual**: um conjunto de regras que vale para uma porta. Como os três escutam em portas diferentes, o nginx sabe qual bloco usar pela porta em que a requisição chegou.

### O grupo de servidores: `upstream`

```nginx
upstream meus_sites {
    server 127.0.0.1:5001;
    server 127.0.0.1:5002;
}
```

O `upstream` dá um nome, `meus_sites`, a uma lista de destinos. Cada linha `server` é um endereço IP e uma porta. Aqui os dois estão em `127.0.0.1`, a própria máquina. Em produção seriam IPs de máquinas diferentes.

Sem nenhuma opção extra, o nginx distribui as requisições em **Round-Robin**. As opções abaixo mudam esse comportamento e são as que você usa nos exercícios:

| Opção | Onde entra | Efeito |
|---|---|---|
| `weight=2` | no fim de uma linha `server` | aquele servidor recebe o dobro de requisições |
| `down` | no fim de uma linha `server` | aquele servidor sai do rodízio |
| `least_conn;` | linha própria dentro do bloco | manda para o servidor com menos conexões ativas |
| `ip_hash;` | linha própria dentro do bloco | o mesmo cliente cai sempre no mesmo servidor |

### O load balancer: `server` na porta 4000

```nginx
server {
    listen 4000;

    location / {
        proxy_pass http://meus_sites;
        add_header Cache-Control "no-store";
    }
}
```

- `listen 4000;` diz que este bloco atende o que chega na porta 4000. É a URL única que o usuário acessa.
- `location / { }` agrupa regras para um caminho da URL. O `/` casa com qualquer caminho, então toda requisição cai aqui.
- `proxy_pass http://meus_sites;` é a linha que faz o balanceamento. Em vez de apontar para um IP e porta fixos, aponta para o nome do `upstream`. A cada requisição o nginx escolhe um servidor do grupo e repassa a requisição para ele.
- `add_header Cache-Control "no-store";` acrescenta um cabeçalho HTTP na resposta pedindo ao navegador para não guardar a página. Sem isso, um F5 poderia mostrar a página antiga sem passar pelo load balancer.

Repare que este bloco não tem `root`: o load balancer não entrega arquivo nenhum. Ele só recebe e repassa.

### Os sites: `server` nas portas 5001 e 5002

```nginx
server {
    listen 5001;
    server_name localhost;

    root CAMINHO_DO_REPOSITORIO/site1;
    index index.html;

    add_header X-Servidor "servidor1";
}
```

- `listen 5001;` atende a porta 5001. O bloco do site2 é idêntico, trocando 5001 por 5002, `site1` por `site2` e `servidor1` por `servidor2`.
- `server_name localhost;` é o nome de host que este bloco responde. Faz diferença quando vários sites dividem a mesma porta. Aqui cada um tem a sua, então ele não interfere.
- `root ...;` é a pasta de onde os arquivos saem. O nginx junta o caminho da URL com essa pasta para achar o arquivo no disco. É a linha que você edita no passo 2.
- `index index.html;` diz qual arquivo entregar quando a URL termina em `/`. Esse já é o valor padrão do nginx, mas deixar explícito mostra o que está acontecendo.
- `add_header X-Servidor "servidor1";` acrescenta um cabeçalho inventado para este kit. O nginx repassa os cabeçalhos do site para o usuário, então ele chega até o terminal no passo 7.

Aqui não existe `proxy_pass`: este bloco entrega os arquivos ele mesmo. É o nginx no papel de servidor web comum.

### O caminho de uma requisição

Juntando tudo, isto acontece quando você aperta F5 em http://localhost:4000:

1. A requisição chega na porta 4000 e cai no bloco `server` do load balancer.
2. A URL `/` casa com o `location /`.
3. O `proxy_pass` consulta o `upstream meus_sites` e escolhe o próximo servidor do rodízio, por exemplo `127.0.0.1:5001`.
4. O nginx abre uma conexão para a porta 5001 e repete a requisição. Ela cai no bloco `server` do site1.
5. Esse bloco junta `root` e `index` e lê o arquivo `site1/index.html` do disco.
6. A resposta volta com o cabeçalho `X-Servidor: servidor1`. O load balancer acrescenta `Cache-Control: no-store` e devolve tudo para o navegador.
7. No próximo F5, o passo 3 escolhe `127.0.0.1:5002` e a página amarela aparece.

## Pré-requisitos

- Git.
- nginx instalado. Siga a seção do seu sistema.

### Windows

1. Baixe o nginx para Windows em https://nginx.org/en/download.html (arquivo `.zip` da versão *Mainline*).
2. Descompacte e renomeie a pasta para `C:\nginx`. Dentro dela deve existir o `nginx.exe`.
3. Abra o **Prompt de Comando** (cmd) e entre na pasta: `cd C:\nginx`.

Todos os comandos do nginx deste roteiro devem ser executados no cmd, de dentro de `C:\nginx`.

### macOS

```bash
brew install nginx
```

### Linux (Ubuntu/Debian)

```bash
sudo apt update && sudo apt install nginx
```

O pacote já sobe um nginx padrão na porta 80. Pare-o antes de usar o do kit:

```bash
sudo systemctl stop nginx
```

No Linux, todos os comandos do nginx deste roteiro precisam de `sudo` na frente.

### Conferindo a instalação

```bash
nginx -v
```

Deve mostrar algo como `nginx version: nginx/1.29.2`.

## Referência rápida de comandos

Estes são todos os comandos do nginx que você vai usar no passo a passo. Troque `/caminho` pelo caminho completo do repositório.

| O que faz | macOS | Linux | Windows (cmd, em `C:\nginx`) |
|---|---|---|---|
| Testar a configuração | `nginx -t -c /caminho/nginx.conf` | `sudo nginx -t -c /caminho/nginx.conf` | `nginx -t -c C:/caminho/nginx.conf` |
| Subir | `nginx -c /caminho/nginx.conf` | `sudo nginx -c /caminho/nginx.conf` | `start nginx -c C:/caminho/nginx.conf` |
| Aplicar mudança na configuração | `nginx -s reload` | `sudo nginx -s reload` | `nginx -s reload` |
| Parar | `nginx -s stop` | `sudo nginx -s stop` | `nginx -s stop` |
| Ver a versão | `nginx -v` | `nginx -v` | `nginx -v` |

## Passo a passo

### 1. Clone o repositório

```bash
git clone https://github.com/leopellizzoni/load-balancer.git
cd load-balancer
```

Anote o caminho completo da pasta. Você vai precisar dele nos próximos passos.

- Windows: `cd` sem argumentos mostra o caminho. Exemplo: `C:\Users\maria\load-balancer`.
- macOS e Linux: `pwd`. Exemplo: `/Users/maria/load-balancer`.

### 2. Aponte o nginx.conf para as suas pastas

Abra o `nginx.conf` em um editor de texto. Troque `CAMINHO_DO_REPOSITORIO` (aparece duas vezes, nas linhas `root` que você viu na seção Entendendo o nginx.conf) pelo caminho que você anotou. Use barras normais `/` mesmo no Windows.

Antes:

```nginx
root CAMINHO_DO_REPOSITORIO/site1;
```

Depois, por exemplo no Windows:

```nginx
root C:/Users/maria/load-balancer/site1;
```

Se o caminho tiver espaços, coloque entre aspas: `root "C:/Meus Projetos/load-balancer/site1";`

### 3. Teste a configuração

O nginx consegue verificar o arquivo sem subir nada. Passe o caminho completo do `nginx.conf`:

macOS:

```bash
nginx -t -c /Users/maria/load-balancer/nginx.conf
```

Linux:

```bash
sudo nginx -t -c /home/maria/load-balancer/nginx.conf
```

Windows (cmd, dentro de `C:\nginx`):

```
nginx -t -c C:/Users/maria/load-balancer/nginx.conf
```

Resultado esperado: duas linhas terminando em `syntax is ok` e `test is successful`. Se aparecer `[emerg]`, leia a mensagem: ela diz o arquivo e a linha do problema.

Atenção: o teste não confere se as pastas das linhas `root` existem. Um caminho errado só aparece no passo 5, como erro 404.

### 4. Suba o nginx

É o mesmo comando do passo 3, sem o `-t`:

macOS:

```bash
nginx -c /Users/maria/load-balancer/nginx.conf
```

Linux:

```bash
sudo nginx -c /home/maria/load-balancer/nginx.conf
```

Windows (cmd, dentro de `C:\nginx`):

```
start nginx -c C:/Users/maria/load-balancer/nginx.conf
```

Quando dá certo o comando não mostra nada e o nginx fica rodando em segundo plano.

### 5. Confira cada servidor sozinho

Antes de testar o balanceador, confirme que os dois sites estão no ar:

- http://localhost:5001 deve mostrar a página azul **Servidor 1**.
- http://localhost:5002 deve mostrar a página amarela **Servidor 2**.

Se aparecer `404 Not Found`, o caminho do passo 2 está errado. Corrija e recarregue com `nginx -s reload`.

### 6. Acesse pela URL única

Agora abra http://localhost:4000 e aperte **F5** várias vezes.

A página deve alternar: azul, amarela, azul, amarela. Cada F5 é uma requisição nova, e o load balancer manda uma para cada servidor, em rodízio. Esse é o Round-Robin funcionando.

### 7. Confira pelo terminal

Cada servidor devolve um cabeçalho HTTP chamado `X-Servidor` dizendo quem respondeu. Dispare 10 requisições seguidas e mostre só esse cabeçalho:

macOS e Linux:

```bash
for i in {1..10}; do curl -sI localhost:4000 | grep X-Servidor; done
```

Windows (cmd):

```
for /L %i in (1,1,10) do @curl -sI localhost:4000 | findstr X-Servidor
```

Resultado esperado: `servidor1` e `servidor2` se alternando linha a linha.

Se estiver no PowerShell em vez do cmd, use `curl.exe` no lugar de `curl`: no PowerShell, `curl` é um apelido de outro comando.

### 8. Para parar

macOS: `nginx -s stop`

Linux: `sudo nginx -s stop`

Windows (cmd, dentro de `C:\nginx`): `nginx -s stop`

Sempre que alterar o `nginx.conf` com o nginx no ar, aplique a mudança com `nginx -s reload` (com `sudo` no Linux). Não precisa parar e subir de novo.

## Problemas comuns

Se algum passo não deu o resultado esperado, procure o sintoma aqui antes de seguir para os exercícios.

| Sintoma | Causa provável | O que fazer |
|---|---|---|
| `nginx: [emerg] open() "...nginx.conf" failed` | Caminho do `-c` errado | Confira o caminho completo do arquivo. Precisa ser absoluto. |
| `404 Not Found` em :5001 ou :5002 | `CAMINHO_DO_REPOSITORIO` não foi trocado ou está errado | Refaça o passo 2 e rode `nginx -s reload`. |
| `403 Forbidden` no Linux | O nginx não tem permissão para entrar na sua pasta home | Rode `chmod o+x ~` ou clone o repositório em `/opt` ou `/srv`. |
| `bind() to 0.0.0.0:4000 failed (Address already in use)` | O nginx do kit já está rodando, ou outro programa usa a porta | Se o nginx já estiver no ar, use `nginx -s reload`. Senão, troque a porta no `nginx.conf`. |
| Alterei o `nginx.conf` e nada mudou | Faltou aplicar a mudança | Rode `nginx -s reload`. |
| A página não alterna no navegador | Cache do navegador ou extensão interferindo | Teste pelo terminal (passo 7). Se alterna lá, abra uma aba anônima. |
| `nginx -s stop` reclama que não achou o pid | O nginx não está rodando | Nada a fazer. |
| Windows: `curl` não é reconhecido ou dá erro estranho | Você está no PowerShell, onde `curl` é outro comando | Use o cmd, ou troque `curl` por `curl.exe`. |
| Windows: `nginx -s stop` não funciona | Comando executado fora de `C:\nginx` | Entre na pasta com `cd C:\nginx` e repita. |

## Exercícios

Todos seguem o mesmo ciclo: edite o `nginx.conf`, rode `nginx -s reload`, repita o passo 7 e anote o que mudou. As linhas necessárias já estão comentadas dentro do bloco `upstream` do `nginx.conf`, e a tabela de opções da seção Entendendo o nginx.conf explica cada uma.

### 1. Pesos (Weighted Round-Robin)

No bloco `upstream`, troque:

```nginx
server 127.0.0.1:5001;
```

por:

```nginx
server 127.0.0.1:5001 weight=2;
```

Recarregue e dispare 9 requisições. Quantas foram para cada servidor? Em que situação faz sentido dar peso maior a um servidor?

### 2. Sessão fixa (ip_hash)

Desfaça o exercício anterior. Adicione `ip_hash;` como primeira linha dentro do bloco `upstream`. Recarregue e aperte F5 várias vezes em http://localhost:4000.

O que aconteceu com a alternância? O `ip_hash` leva o mesmo cliente sempre para o mesmo servidor. Pense em uma situação em que isso é necessário. Dica: um carrinho de compras guardado na memória do servidor.

### 3. Menos conexões (least_conn)

Troque o `ip_hash;` por `least_conn;`. Recarregue e repita o passo 7.

Você provavelmente não vai notar diferença em relação ao Round-Robin. Por quê? Cada página aqui é entregue em milissegundos, então nunca há conexões acumuladas em um servidor. Em que tipo de sistema o `least_conn` faria diferença?

### 4. Terceiro servidor

Remova o `least_conn;` para voltar ao Round-Robin. Depois:

1. Copie a pasta `site2` para `site3` e mude o texto e a cor do `index.html` para identificar o servidor 3.
2. No `nginx.conf`, copie o bloco `server` do site2, trocando a porta para 5003, a pasta para `site3` e o cabeçalho para `servidor3`.
3. Adicione `server 127.0.0.1:5003;` no bloco `upstream`.
4. Recarregue e repita o passo 7.

Esse é o ponto principal de um load balancer: você aumentou a capacidade sem mudar a URL que o usuário acessa.

### 5. Simule a queda de um servidor

Com os três servidores no ar, comente o bloco `server` inteiro do site2 (coloque `#` no início de cada linha) e recarregue. A porta 5002 deixa de existir, como se a máquina tivesse desligado.

Aperte F5 em http://localhost:4000 várias vezes. Apareceu algum erro? O que o load balancer fez com as requisições que iriam para o servidor 2?

Depois descomente o bloco e recarregue. Observe se o servidor 2 voltou ao rodízio.

Variação: em vez de comentar o bloco, marque o servidor no `upstream` com `server 127.0.0.1:5002 down;`. É assim que se tira um servidor do ar de propósito, para manutenção, sem derrubar nada.

## Licença

MIT. Veja o arquivo [LICENSE](LICENSE).
