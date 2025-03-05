# Entrega do Desafio: Infraestrutura AWS com Terraform - VExpenses

## 1. Descrição Técnica (Tarefa 1)

Este projeto utiliza o Terraform para provisionar uma infraestrutura básica na AWS. O código original (main.tf) cria os seguintes recursos:

- **Provider AWS:** Define a região `us-east-1` para os recursos.
- **Variáveis:** Define as variáveis `projeto` e `candidato` para nomear dinamicamente os recursos.
- **Par de Chaves SSH:**
  - **tls_private_key:** Gera uma chave RSA de 2048 bits.
  - **aws_key_pair:** Cria um par de chaves na AWS utilizando a chave pública gerada.
- **VPC:** Cria uma VPC com o CIDR `10.0.0.0/16` com suporte a DNS e hostnames.
- **Subnet:** Cria uma subnet na VPC com o CIDR `10.0.1.0/24` na zona de disponibilidade `us-east-1a`.
- **Internet Gateway:** Cria um Internet Gateway para permitir comunicação com a Internet.
- **Tabela de Rotas:** Define uma rota para todo o tráfego (`0.0.0.0/0`) via o Internet Gateway e associa essa tabela à subnet.
- **Grupo de Segurança:** Cria um grupo que permite acesso SSH (porta 22) e HTTP (porta 80) e libera todo o tráfego de saída.
- **AMI e Instância EC2:**
  - Busca a imagem mais recente do Debian 12.
  - Cria uma instância EC2 do tipo `t2.micro`, associada à subnet e ao grupo de segurança.
  - Configura um volume de 20 GB, com exclusão automática na terminação.
  - Utiliza `user_data` para executar comandos de atualização do sistema.
- **Outputs:** Exibe a chave privada e o endereço IP público da instância.

## 2. Código Terraform Modificado (main.tf)

```hcl
provider "aws" {
  region = "us-east-1"
}

variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}

# Variável para restringir o acesso SSH a um IP específico.
variable "allowed_ssh_cidr" {
  description = "IP permitido para acesso SSH (ex.: \"192.168.1.100/32\")"
  type        = string
  default     = "YOUR_IP_HERE/32"  # Substitua pelo IP do seu ambiente
}

# Local para padronizar os nomes dos recursos.
locals {
  name_prefix = "${var.projeto}-${var.candidato}"
}

resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${local.name_prefix}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}

resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${local.name_prefix}-vpc"
  }
}

resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${local.name_prefix}-subnet"
  }
}

resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${local.name_prefix}-igw"
  }
}

resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${local.name_prefix}-route_table"
  }
}

resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name = "${local.name_prefix}-route_table_association"
  }
}

resource "aws_security_group" "main_sg" {
  name        = "${local.name_prefix}-sg"
  description = "Permitir acesso SSH restrito e HTTP para Nginx"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description = "Acesso SSH restrito"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.allowed_ssh_cidr]
  }

  ingress {
    description = "Acesso HTTP para Nginx"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Permitir todo o tráfego de saída"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${local.name_prefix}-sg"
  }
}

data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}

resource "aws_instance" "debian_ec2" {
  ami                    = data.aws_ami.debian12.id
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.main_subnet.id
  key_name               = aws_key_pair.ec2_key_pair.key_name
  vpc_security_group_ids = [aws_security_group.main_sg.id]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  # Automação da instalação do Nginx via user_data
  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get install -y nginx
              systemctl enable nginx
              systemctl start nginx
              EOF

  tags = {
    Name = "${local.name_prefix}-ec2"
  }
}

output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
```

# 3. Descrição Técnica das Melhorias Implementadas (Tarefa 2)
## Melhoria de Segurança
### Restrição do Acesso SSH:
Foi criada a variável allowed_ssh_cidr para que o acesso via SSH (porta 22) seja permitido apenas de um IP específico. Isso evita o risco de exposição do servidor a qualquer origem.

Justificativa: Restringir o acesso SSH minimiza a superfície de ataque, aumentando a segurança da instância.

### Uso do vpc_security_group_ids:
Na definição da instância EC2, o atributo security_groups foi substituído por vpc_security_group_ids, o que é a prática recomendada para recursos em VPC.

Justificativa: Garante uma associação mais robusta e clara dos grupos de segurança à instância.

### Uso de locals para Padronização de Nomes:
A definição do local.name_prefix simplifica e padroniza a nomeação dos recursos, facilitando a manutenção e a identificação dos mesmos na AWS.

Justificativa: A consistência nos nomes ajuda na organização e rastreabilidade dos recursos, especialmente em ambientes com múltiplos componentes.

## Automação da Instalação do Nginx
### Atualização do Script user_data:
O script foi modificado para que, ao inicializar, a instância execute os comandos necessários para instalar e iniciar o Nginx automaticamente:
 Atualização dos pacotes (apt-get update -y)
Instalação do Nginx (apt-get install -y nginx)
Habilitação do Nginx para iniciar automaticamente (systemctl enable nginx)
Inicialização imediata do serviço (systemctl start nginx)

Justificativa: Automatizar a instalação do Nginx garante que a instância esteja pronta para servir requisições web imediatamente após a criação, eliminando etapas manuais e agilizando o processo de provisionamento.
Outras Melhorias
## Comentários e Organização:
Foram adicionados comentários e o uso de variáveis e locais para melhorar a legibilidade e facilitar futuras manutenções.

## Uso Consistente de Tags:
Todas as partes da infraestrutura recebem tags padronizadas, facilitando a identificação e o gerenciamento dos recursos na AWS.

# 4. Instruções de Uso
## Pré-requisitos
### Terraform Instalado: Certifique-se de que o Terraform está instalado. Você pode baixá-lo em terraform.io.
### Credenciais AWS Configuradas: Configure suas credenciais da AWS (via variáveis de ambiente, arquivo ~/.aws/credentials ou outro método suportado pelo Terraform).
### Editor de Código: Recomenda-se o uso do VS Code ou outro editor com suporte a HCL para facilitar a edição.
## Passos para Inicializar e Aplicar a Configuração
### Clone ou Baixe o Projeto:

``` bash
git clone <URL_DO_REPOSITÓRIO>
cd <NOME_DA_PASTA>
```
### Inicialize o Diretório do Terraform:

Este comando baixa os plugins do provider AWS e inicializa o ambiente.

``` bash
terraform init
```
### Verifique a Configuração:

Execute um plano para ver as mudanças que serão aplicadas:

``` bash
terraform plan
```
### Aplique a Configuração:

Se o plano estiver de acordo, aplique a configuração:

``` bash
terraform apply
``` 
Você será solicitado a confirmar a execução (digite yes).

### Acessando a Instância:

Após a aplicação, use o output ec2_public_ip para obter o IP público da instância.

Utilize a chave privada (output private_key) para se conectar via SSH:

``` bash
ssh -i <CAMINHO_DA_CHAVE_PRIVADA> admin@<EC2_PUBLIC_IP>
```

## Observações
Substitua o valor da variável allowed_ssh_cidr no arquivo main.tf para o IP ou intervalo de IPs que você deseja autorizar para o acesso SSH.
