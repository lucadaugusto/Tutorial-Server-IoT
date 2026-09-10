# Servidor IoT FIWARE com Docker no Ubuntu

Guia de instalação de um servidor FIWARE local, em máquina virtual local ou em VM de nuvem, usando Docker e Docker Compose. O stack sobe a partir do repositório [FIWARE Descomplicado](https://github.com/fabiocabrini/fiware), do prof. Fabio Cabrini, e é validado por uma collection do Postman.

Siga os passos na ordem, de 1 a 27. Cada bloco de comando contém **um comando só**: copie, cole no terminal, aperte Enter, espere terminar, e só então vá para o próximo passo.

Os passos 1 a 8 são para máquina virtual local. Em VM de nuvem (AWS, Azure, Google), pule para o passo 9 e execute tudo pelo terminal SSH.

## Componentes que serão instalados

| Componente | Função | Porta padrão |
|---|---|---|
| Orion Context Broker | gerenciamento de contexto (entidades NGSI-v2) | 1026 |
| IoT Agent MQTT | ponte entre dispositivos MQTT e o Orion | 4041 |
| STH-Comet | histórico de séries temporais | 8666 |
| MongoDB | persistência do Orion e do STH | 27017 |
| Mosquitto | broker MQTT | 1883 |

Confirme as portas no `docker-compose.yml` do repositório clonado antes de liberar regras de firewall.

---

## Passo 1 — Instalar o software de máquina virtual

Baixe e instale um software de máquina virtual. Na demonstração foi usada a versão Free do VMware Workstation Pro, mas o VirtualBox também funciona.

Vídeo tutorial: https://www.youtube.com/watch?v=M-TTEVVNoXA

Explicação: a máquina virtual (VM) permite rodar o Linux dentro do Windows ou do macOS, sem formatar o computador. Instale o VMware (ou o VirtualBox) antes de seguir para o próximo passo.

## Passo 2 — Baixar a imagem ISO do Linux

Baixe o Ubuntu Desktop 22.04.3 LTS de 64 bits: https://ubuntu.com/download/alternative-downloads

Explicação: o arquivo ISO é a imagem de instalação do sistema operacional. Baixe exatamente a versão indicada (64 bits) para evitar problemas de compatibilidade mais adiante.

## Passo 3 — Instalar a imagem na máquina virtual

Instale a ISO na VM e configure a rede.

Referência para o modo Bridge no VMware: https://dirceuresende.com/blog/como-configurar-a-rede-da-sua-vm-no-modo-bridge-no-vmware-player/

Explicação: existem dois modos de rede e a escolha muda o resultado.

- **Bridge** — a VM recebe um IP da rede local e passa a ser enxergada pelos outros computadores da rede. É o modo necessário se algum dispositivo externo (ESP32, celular, outro PC) for publicar no MQTT ou acessar as APIs do servidor.
- **NAT** — mais simples de configurar, mas o IP da VM não é alcançável a partir da rede local. Só funciona para testes feitos de dentro da própria VM ou do computador que hospeda a VM.

Se você vai conectar dispositivos reais, use Bridge.

## Passo 4 — (Alternativa) Baixar a imagem pronta da VM

Imagem do Ubuntu 22.04 e do VMware usadas na demonstração em sala:
https://drive.google.com/drive/folders/1rgZp5JZHzKdwtq1GDwK-IsGk6FKGd63Y?usp=sharing

Explicação: este passo é opcional. Use somente se preferir começar de uma imagem já pronta em vez de instalar o Ubuntu do zero nos passos 2 e 3.

## Passo 5 — Instalar o Postman

https://www.postman.com/downloads/

Explicação: o Postman é o programa usado para enviar comandos (requisições HTTP) ao servidor IoT e verificar se ele está respondendo corretamente.

## Passo 6 — Baixar a collection de comandos do Postman

https://github.com/fabiocabrini/fiware/blob/main/FIWARE%20Descomplicado.postman_collection.json

Explicação: a collection é um arquivo com todos os testes já prontos, assim você não precisa montar cada requisição manualmente no Postman.

## Passo 7 — Importar a collection no Postman

Procedimento de importação: https://nfe.io/docs/documentacao/nota-fiscal-produto-eletronica/importar-colecao-postman/

Explicação: depois de importada, a collection aparece organizada em pastas dentro do Postman, com as requisições já nomeadas e prontas para uso.

## Passo 8 — Abrir o terminal do Linux

No Ubuntu, aperte `Ctrl + Alt + T`. Em VM de nuvem, conecte por SSH.

Explicação: todos os comandos a partir daqui devem ser digitados dentro desse terminal, um de cada vez, apertando Enter após cada linha e esperando o comando terminar.

## Passo 9 — Atualizar a lista de pacotes do Linux

```bash
sudo apt update
```

Explicação: atualiza a lista de pacotes disponíveis para instalação. Rode este comando sempre que abrir uma VM nova. Ele vai pedir a senha do seu usuário; ao digitar, nada aparece na tela, isso é normal.

## Passo 10 — Instalar o pacote de comandos de rede

```bash
sudo apt install -y iproute2
```

Explicação: garante que o comando `ip`, usado no próximo passo para descobrir o endereço da VM, esteja disponível. Na maioria das instalações ele já vem instalado e o comando termina rápido, sem baixar nada.

## Passo 11 — Ler o IP da VM (Virtual Machine)

```bash
ip -4 addr show scope global
```

Explicação: mostra as interfaces de rede da VM. Procure o número depois de `inet` na interface ativa (geralmente `ens33`, `eth0` ou `enp0s3`). Esse é o IP que você vai usar no lugar de `{{url}}` dentro do Postman. Anote esse número.

O tutorial original usa `ifconfig`, que exige instalar o pacote `net-tools`. O `ifconfig` continua funcionando, mas está obsoleto desde o Ubuntu 18.04 e não vem mais instalado por padrão; o `ip` é o substituto oficial.

Em VM de nuvem, este comando mostra o endereço privado da instância. No Postman use o IP público ou o DNS da instância, que aparecem no painel do provedor.

## Passo 12 — Instalar as dependências e o git

```bash
sudo apt-get install -y ca-certificates curl gnupg lsb-release git
```

Explicação: instala as ferramentas necessárias para baixar e validar os arquivos do Docker, e também o `git`, usado no passo 22 para clonar o repositório do FIWARE.

## Passo 13 — Criar a pasta da chave do Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Explicação: cria a pasta onde ficará a chave de segurança do Docker. Este comando não imprime nada na tela quando dá certo.

## Passo 14 — Baixar e salvar a chave GPG do Docker

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor --yes -o /etc/apt/keyrings/docker.gpg
```

Explicação: baixa a chave oficial do Docker e salva no formato que o `apt` entende. Copie a linha inteira de uma vez, sem quebrar no meio.

## Passo 15 — Liberar a leitura da chave

```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Explicação: dá permissão de leitura à chave salva no passo anterior. Sem isso o `apt` reclama de erro de GPG no passo 17.

## Passo 16 — Adicionar o repositório oficial do Docker

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Explicação: adiciona o repositório oficial do Docker ao sistema, para que o `apt` consiga encontrar e instalar o Docker. É uma linha só, mesmo que apareça quebrada na tela. Copie tudo de uma vez, incluindo as aspas.

## Passo 17 — Atualizar novamente a lista de pacotes

```bash
sudo apt-get update -y
```

Explicação: atualiza a lista de pacotes incluindo agora o repositório do Docker adicionado no passo anterior. Se aparecer alguma mensagem com `GPG error`, refaça os passos 13 a 16.

## Passo 18 — Instalar o Docker e o Docker Compose

```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Explicação: instala o Docker Engine, o CLI, o containerd e o Docker Compose, tudo o que é necessário para rodar os containers do FIWARE. Este passo demora alguns minutos.

## Passo 19 — Habilitar e iniciar o Docker

```bash
sudo systemctl enable --now docker
```

Explicação: faz o Docker iniciar automaticamente junto com o sistema e já inicia o serviço agora. O `--now` faz as duas coisas em um comando só.

## Passo 20 — Verificar a versão do Docker

```bash
docker --version
```

Explicação: se aparecer um número de versão, o Docker está instalado.

## Passo 21 — Verificar a versão do Docker Compose

```bash
docker compose version
```

Explicação: se aparecer um número de versão, o Compose está instalado. Repare que é `docker compose`, com espaço, e não `docker-compose` com hífen.

## Passo 22 — Clonar o repositório do FIWARE Descomplicado

```bash
git clone https://github.com/fabiocabrini/fiware
```

Explicação: copia os arquivos do repositório para dentro da VM. Esse comando cria uma pasta nova chamada `fiware` com tudo o que é necessário para subir o servidor.

## Passo 23 — Entrar na pasta do FIWARE

```bash
cd fiware
```

Explicação: entra na pasta que acabou de ser criada. Os próximos comandos precisam ser executados de dentro dela, pois é onde fica o arquivo `docker-compose.yml`. Se você fechar o terminal, precisa rodar `cd fiware` de novo antes de continuar.

## Passo 24 — Subir os containers

```bash
sudo docker compose up -d
```

Explicação: sobe todos os containers do FIWARE definidos no `docker-compose.yml`. Na primeira vez, o Docker precisa baixar as imagens e isso pode levar vários minutos. O parâmetro `-d` roda os containers em segundo plano, liberando o terminal.

## Passo 25 — Verificar se os containers subiram

```bash
sudo docker compose ps
```

Explicação: lista os containers e o estado de cada um. Todos devem aparecer como `running` ou `Up`. Se algum aparecer como `restarting` ou `exited`, ele não subiu, e o passo 27 vai falhar.

## Passo 26 — Ver o consumo de recursos dos containers

```bash
sudo docker stats
```

Explicação: mostra em tempo real o uso de CPU, memória e rede de cada container. Aperte `Ctrl + C` para sair dessa tela e voltar ao terminal normal. Este comando serve para acompanhar consumo; quem diz se o container está saudável é o passo 25.

## Passo 27 — Testar o Health Check dos componentes

Abra o Postman, configure a variável `{{url}}` com o IP anotado no passo 11 e execute as requisições de health check da collection. Cada uma deve retornar `Status: 200 OK`.

| Componente | Requisição |
|---|---|
| IoT Agent MQTT | `GET http://{{url}}:4041/iot/about` |
| Orion Context Broker | `GET http://{{url}}:1026/version` |
| STH-Comet | `GET http://{{url}}:8666/version` |

Explicação: o health check confirma que cada componente do FIWARE está no ar e respondendo. Se o Postman não conseguir se conectar, use os testes do anexo A para descobrir se o problema é no servidor ou na rede.

---

## Anexo A — Testar os health checks pelo terminal

Estes comandos rodam de dentro da própria VM e servem para separar problema de container de problema de rede. Cada um imprime apenas o código de resposta; o esperado é `200`.

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:4041/iot/about
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:1026/version
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8666/version
```

Se o terminal responde `200` e o Postman não alcança o servidor, o problema é de rede: VM em modo NAT, firewall do computador que hospeda a VM, ou security group da nuvem. O FIWARE está funcionando.

## Anexo B — (Opcional) Rodar o Docker sem sudo

```bash
sudo usermod -aG docker $USER
```

```bash
newgrp docker
```

Explicação: adiciona seu usuário ao grupo `docker`, o que dispensa o `sudo` nos comandos do Docker. Pertencer a esse grupo dá poder equivalente a root na máquina. Em servidor exposto na internet, pense se compensa.

## Anexo C — Segurança

O stack do FIWARE Descomplicado é didático e sobe sem autenticação nenhuma. Orion, IoT Agent, STH-Comet, MongoDB e Mosquitto ficam acessíveis a qualquer um que alcance as portas.

Em VM local isso não é problema. Em VM de nuvem com IP público:

- restrinja o security group ao seu IP de origem, nunca `0.0.0.0/0`;
- não exponha a porta 27017 (MongoDB aberto na internet é varrido por bots e sequestrado em questão de horas).

## Anexo D — Problemas comuns

| Sintoma | Causa provável | O que fazer |
|---|---|---|
| `docker compose` não é reconhecido | plugin não instalado, ou uso de `docker-compose` com hífen | refaça o passo 18 e use `docker compose` com espaço |
| `GPG error` no passo 17 | chave gravada no lugar errado ou sem permissão | refaça os passos 13 a 16 |
| Postman não alcança o IP da VM | VM em modo NAT | troque a rede da VM para Bridge (passo 3) |
| Container em `restarting` no passo 25 | porta ocupada no host ou pouca memória RAM | rode `sudo docker compose logs` e leia a mensagem de erro |
| Health check responde 200 no terminal e falha no Postman | firewall do host ou security group da nuvem | veja o anexo A |
| `docker-compose.yml not found` no passo 24 | terminal fora da pasta | rode `cd fiware` |

## Créditos

Stack e collection: [FIWARE Descomplicado](https://github.com/fabiocabrini/fiware), do prof. Fabio Cabrini.
