# AWS - OBSERVABILITY

## IAM - Criar função para conectar a instância EC2 através do Systems Manager (SSM)

- IAM → Funções → Criar função

- Serviço da AWS - EC2 - Adicionar permissão para o tipo de conexão

- Permissão padrão necessária para cada tipo de conexão:

  - AmazonEC2RoleforSSM → Session Manager

  - EC2InstanceConnect → SSH


## EC2 → Criar instância

Executar instância
  
  - Nome e tags
    - Nome: srv_obs_2023

  - Imagens de aplicação e de sistema operacional
    - Início rápido: Amazon Linux
    - Imagem de máquina da Amazon (AMI): Amazon Linux 2 AMI (HVM) - Kernel 5.10, SSD Volume Type
    - Arquitetura: 64 bits (x86)

  - Tipo de instância: t2.micro

  - Par de chaves (login)
    - Nome do par de chaves: Prosseguir sem um par de chaves

  - Configurações de rede
    - Criar grupo de segurança
    - [ ] Desmarcar opção Permitir tráfego SSH

  - Detalhes avançados
    - Perfil de instância do IAM: Selecione a função criada

Executar instância

## DOCKER

### Instalação:

Ativar bash
```
bash
```

Atualizar os pacotes de instalação
```
sudo yum update -y
```
Instalar a versão mais recente da engine otimizada para Amazon Linux 2 AMI
```
sudo amazon-linux-extras install docker
```

OU para Amazon Linux 2023 AMI
```
yum install docker -y
```

Iniciar o serviço docker
```
sudo service docker start
```
Habilitar o docker daemon para iniciar após cada reinicialização do sistema
```
sudo systemctl enable docker
```
Executar comandos docker sem sudo
```
sudo usermod -a -G docker ssm-user
```
Reiniciar instância EC2 para que funcione o novo grupo de permissões

Obs. Deve aguardar um pouco para conectar novamente
```
sudo reboot
```
```
bash
```
Verificar se pode executar o Docker sem sudo
```
docker info
```

## DOCKER COMPOSE

### Instalação:

Versão latest
```
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
```
Permissões após o download
```
sudo chmod +x /usr/local/bin/docker-compose
```
Verificar instalação
```
docker-compose --version
```
Path simbólico (Não é necessário)

*Somente se o comando docker-compose --version não funcionar
```
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

## CONTAINERS

Subir container com nginx
-d = modo debug
-p = bind de porta do nginx com http
--name = escolher um nome para criar o container
```
docker container run --name meucontainer -d -p 8080:80 nginx
```

Listar containers -a = Listar todos ativos e inativos
```
docker container ls -a
```

Permitir tráfego de entrada para acessar através do navegador

EC2 → Instâncias → Selecionar instância → Segurança → Grupos de segurança

Editar regras de entrada → Adicionar regra

Regras de entrada

  - Tipo

    - Todos os TCPs

  - Origem

    - Meu IP

Salvar regras

Copie o IPv4 público da instância EC2 e acesse no navegador utilizando a porta 8080

Ex: 12.123.123.123:8080

## PROMETHEUS

### Criar estrutura de arquivos

```
cd
```
```
touch docker-compose.yml
```
```
mkdir prometheus
```
```
mkdir grafana
```
```
cd prometheus
```
```
touch alert.yml prometheus.yml
```
```
cd ..
```

Editar arquivos yml com vim ou nano e adicionar o conteúdo

```bash
sudo vim nomedoarquivo.yml
```
i = insert

ESC :wq = escrever e sair

docker-compose.yml

```yaml
version: '3'
services:
  prometheus:
    image: prom/prometheus:v2.46.0
    ports:
      - 9000:9090
    networks:
      - backend
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus-data:/prometheus
    command: --web.enable-lifecycle  --config.file=/etc/prometheus/prometheus.yml

volumes:
  prometheus-data:

networks:
  frontend:
  backend:
```

prometheus.yml

```yaml
global:
  scrape_interval:     10s
  evaluation_interval: 10s

rule_files:
  - 'alert.yml'

alerting:
  alertmanagers:
  - scheme: http
    static_configs:
    - targets:
      - "alertmanager:9093"

scrape_configs:
  - job_name: 'node'
    scrape_interval: 5s
    static_configs:
            - targets: ['localhost:9090','cadvisor:8080','node-exporter:9100']
  - job_name: 'netdata'
    metrics_path: '/api/v1/allmetrics'
    params:
      format: [prometheus]
    honor_labels: true
    scrape_interval: 5s
    static_configs:
         - targets: ['YOUR_NETDATA_IP:19999']
```

alert.yml

```yaml
groups:
  - name: DemoAlerts
    rules:
      - alert: InstanceDown
        expr: up{job="services"} < 1
        for: 5m
```

Executar (Subir tudo utilizando o compose)

```bash
docker-compose up -d
```

Acessar no navegador o endereço IPv4 público da instância EC2 na porta 9000

Ex: 12.123.123.123:9000

### AULA 2

Atualizar o conteudo do docker-compose.yml utilizando vim
```yaml
version: '3'
services:
  prometheus:
    image: prom/prometheus:v2.46.0
    ports:
      - 9000:9090
    networks:
      - backend
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus-data:/prometheus
    command: --web.enable-lifecycle  --config.file=/etc/prometheus/prometheus.yml

  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    hostname: '{{.Node.ID}}'
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - backend
    deploy:
      mode: global
    ports:
      - 8080:8080

  grafana:
    image: grafana/grafana:10.0.0
    ports:
      - 3000:3000
    networks:
      - backend
      - frontend
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning


  alertmanager:
    image: prom/alertmanager:v0.25.0
    networks:
      - backend
    ports:
      - 9093:9093
    volumes:
      - ./alertmanager:/etc/alertmanager
      - alertmanager-data:/data
    command: --config.file=/etc/alertmanager/alertmanager.yml

  nginx:
    image: nginx
    ports:
        - 80:80
    networks:
        - backend

volumes:
  prometheus-data:
  grafana-data:
  alertmanager-data:

networks:
  frontend:
  backend:
```

Criar diretório do alertmanager 
```bash
mkdir alertmanager
```

Navegar para o diretório
```bash
cd alertmanager
```

Criar file alertmanager.yml e incluir o conteudo utilizando o vim
```bash
touch alertmanager.yml
```

alertmanager.yml
```yaml
route:
  receiver: 'mail'

receivers:
  - name: 'mail'
    email_configs:
      - smarthost: 'smtp.gmail.com:587'
        from: 'your_mail@gmail.com'
        to: 'some_mail@gmail.com'
        auth_username: '...'
        auth_password: '...'
```

acessar o diretorio grafana

Retornar ao diretório padrão
```bash
cd
```

```bash
cd grafana
```

Criar diretório provisioning
```bash
mkdir provisioning
```

Navegar para o diretório
```bash
cd provisioning
```

Criar diretório datasources
```bash
mkdir datasources
```

Navegar para o diretório
```bash
cd datasources
```

criar file prometheus_ds.yml adicionar conteúdo com o vim
```bash
touch prometheus_ds.yml
```

prometheus_ds.yml
```yaml
datasources:
  - name: Prometheus
    access: proxy
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
```

Executar para fazer as alterações
```bash
docker-compose up -d
```

Edereços de acesso:
- Prometheus: ip:9000
- cAdvisor: ip:8080
- Grafana: ip:3000
  - Username: admin
  - Password: admin
- AlertManager: ip:9093

Obs caso apresente algum erro use esse comando para remover todos os containers
```bash
docker container rm -f $(docker container ls -a -q)
```
Execute o docker-compose para subir os containers novamente
```bash
docker-compose up -d
```

### AULA 3

Iniciar containers
```bash
docker-compose up -d
```

Verificar security group da EC2 se a regra de entrada personalizada esta setada para meu ip **recente**.
Obs. Os provedores de internet costumam trocar o ip quando o modem é reiniciado.

Acessar o Grafana
ip:3000

Dashboard > New Dashboard
Datasource: Prometheus

#### PromQL:

Qtd containers:
```
count(rate(container_last_seen{name=~".+"}[1m]))
```

Memoria:
```
(container_memory_usage_bytes{container_label_com_docker_compose_service=~"alertmanager|cadvisor|grafana|nginx|prometheus",image!=""} - container_memory_cache{container_label_com_docker_compose_service=~"alertmanager|cadvisor|grafana|nginx|prometheus",image!=""})
```

CPU:
```
(rate(container_cpu_user_seconds_total{container_label_com_docker_compose_service=~"alertmanager|cadvisor|grafana|nginx|prometheus",image!=""}[5m]) * 100)
```

Trafego Entrada:
```
sum(rate(container_network_receive_bytes_total{name=~".+"}[1m])) by (name)
```

Trafego Saída:
```
sum(rate(container_network_transmit_bytes_total{name=~".+"}[1m])) by (name)
```

### AULA 4

#### Exporters
```
cd
```
Criar diretório para api
```
mkdir apiastronauta
```
Navegar para o diretório
```
cd apiastronauta
```
Criar arquivos
```
touch exporter_novo.py requirements.txt Dockerfile
```
Adicionar os conteúdos com o vim
```
sudo vim exporter_novo.py
```
exporter_novo.py(**Criar**)
```
import requests # Importa o módulo requests para fazer requisições HTTP
import json # Importa o módulo json para converter o resultado em JSON
import time # Importa o módulo time para fazer o sleep
from prometheus_client import start_http_server, Gauge

url_numero_pessoas = 'http://api.open-notify.org/astros.json'

def pega_numero_astronautas():
    try:
        """
        Pegar o número de astronautas no espaço
        """
        response = requests.get(url_numero_pessoas)
        data = response.json()
        return data['number']
    except Exception as e:
        print("Não foi possível acessar a url!")
        raise e

def atualiza_metricas():
    try:
        """
        Atualiza as métricas com o número de astronautas e local da estação espacial internacional
        """
        numero_pessoas = Gauge('numero_de_astronautas', 'Número de astronautas no espaço')

        while True:
            numero_pessoas.set(pega_numero_astronautas())
            time.sleep(10)
            print("O número atual de astronautas no espaço é: %s" % pega_numero_astronautas())
    except Exception as e:
        print("A quantidade de astronautas não pode ser atualizada!")
        raise e

def inicia_exporter():
    try:
        """
        Iniciar o exporter
        """
        start_http_server(8895)
        return True
    except Exception as e:
        print("O Servidor não pode ser iniciado!")
        raise e

def main():
    try:
        inicia_exporter()
        print('Exporter Iniciado')
        atualiza_metricas()
    except Exception as e:
        print('\nExporter Falhou e Foi Finalizado! \n\n======> %s\n' % e)
        exit(1)


if __name__ == '__main__':
    main()
    exit(0)
```

```
sudo vim requirements.txt
```
requirements.txt(**Criar**)
```
requests
prometheus-client
prometheus_client
```

```
sudo vim Dockerfile
```
Dockerfile(**Criar**)
```
FROM python:3.8-slim

# Adicionando algumas labels para identificar a imagem
LABEL description "Dockerfile para criar o nosso primeiro  exporter para o Prometheus"

# Adicionando o exporter.py para a nossa imagem
COPY . .

# Instalando as bibliotecas necessárias para o exporter
# através do `requirements.txt`.
RUN pip3 install -r requirements.txt

# Executando o exporter
CMD python3 exporter_novo.py
```

Build da imagem do container
```
docker build -f Dockerfile . -t primeiro-exporter:0.1
```

Verificar se a imagem primeiro-exporter foi criada
```
docker image ls
```

Retornar ao diretorio raiz
```
cd
```

Editar docker-compose.yml e **adicionar conteúdo** referente a nova api, imagem container, portas e rede
```
sudo vim docker-compose.yml
```
docker-compose.yml (**Adicionar**)
```
  apiastronauta:
    image: primeiro-exporter:0.1
    ports:
      - 8895:8895
    networks:
      - backend
```

Executar o container
```
docker-compose up -d
```

Editar prometheus.yml e **adicionar conteúdo** referente ao job e target
```
sudo vim prometheus/prometheus.yml
```
prometheus.yml(**Adicionar**)
```
  - job_name: "Primeiro Exporter" # Nome do job que vai coletar as métricas do primeiro exporter.
    scrape_interval: 5s
    static_configs:
      - targets: ['apiastronauta:8895'] # Endereço do alvo monitorado, ou seja, o nosso primeiro exporter.
```


Mostrar containers ativos e suas ids
```
docker container ps
```
reiniciar o container do prometheus para capturar novas metricas
```
docker container restart idprometheus
```

Para capturar metricas no Grafana:
```
numero_de_astronautas{job="Primeiro Exporter"}
```
label:
```
{{Instance}}
```

#### HTTP
```
mkdir http
```
```
cd http
```
```
touch exporter_chamada_http.py requirements.txt Dockerfile
```
Adicionar conteúdos

```
sudo vim exporter_chamada_http.py
```
```
#!/usr/bin/env python

import os
import re
import platform
from time import sleep

from http.server import BaseHTTPRequestHandler, HTTPServer

from prometheus_client import start_http_server, Counter, REGISTRY

class HTTPRequestHandler(BaseHTTPRequestHandler):

    def do_GET(self):
        if self.path == '/':
            self.send_response(200)
            self.send_header('Content-type','text/html')
            self.end_headers()
            self.wfile.write(bytes("<b> Hello World !</b>", "utf-8"))
            request_counter.labels(status_code='200', instance=platform.node()).inc()
        else:
            self.send_error(404)
            request_counter.labels(status_code='404', instance=platform.node()).inc()


if __name__ == '__main__':

    start_http_server(9000)
    request_counter = Counter('http_requests', 'HTTP request', ["status_code", "instance"])

    webServer = HTTPServer(("localhost", 8888), HTTPRequestHandler).serve_forever()
    print("Server started")
```

```
sudo vim requirements.txt
```
```
requests
prometheus-client
prometheus_client
```

```
sudo vim Dockerfile
```
```
# Vamos utilizar a imagem slim do Python
FROM python:3.8-slim

# Adicionando algumas labels para identificar a imagem
LABEL description "Dockerfile para criar o nosso segundo  exporter para o Prometheus"

# Adicionando o exporter.py para a nossa imagem
COPY . .

# Instalando as bibliotecas necessárias para o exporter
# através do `requirements.txt`.
RUN pip3 install -r requirements.txt

# Executando o exporter
CMD python3 exporter_chamada_http.py
```

Build da imagem do container
```
docker build -f Dockerfile . -t segundo-exporter:0.1
```

Verificar se a imagem primeiro-exporter foi criada
```
docker image ls
```

```
cd
```

```
sudo vim docker-compose.yml
```
docker-compose.yml(**Adicionar**)
```
  apipython:
    image: segundo-exporter:0.1
    ports:
      - 8888:8888
    networks:
      - backend
```

Subir o container
```
docker-compose up -d
```

```
sudo vim prometheus/prometheus.yml
```
prometheus.yml(**Adicionar**)
```
  - job_name: "Segundo Exporter" # Nome do job que vai coletar as métricas do primeiro exporter.
    scrape_interval: 5s
    static_configs:
      - targets: ['apipython:9000'] # Endereço do alvo monitorado, ou seja, o nosso primeiro exporter.
```

Mostrar containers ativos e suas ids
```
docker container ps
```
reiniciar o container do prometheus para capturar novas metricas
```
docker container restart idprometheus
```
