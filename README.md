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

Executar

```bash
docker-compose up -d
```

Acessar no navegador o endereço IPv4 público da instância EC2 na porta 9000

Ex: 12.123.123.123:9000
