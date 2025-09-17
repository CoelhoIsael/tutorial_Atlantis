# tutorial_Atlantis
Este é  um  tutorial de instalação do Atlantis server 


## 1. Criar usuário dedicado para o Atlantis
<pre> `` 
sudo adduser --system --home /home/atlantis --shell /bin/bash --group atlantis
id atlantis
</pre>


## 2. Baixar o binário do Atlantis
<pre> ``
wget https://github.com/runatlantis/atlantis/releases/latest/download/atlantis_linux_amd64.zip
unzip atlantis_linux_amd64.zip
sudo mv atlantis /usr/local/bin/
sudo chmod +x /usr/local/bin/atlantis
</pre>

## 3. Configuração básica do Atlantis
crie um arquivo /etc/atlantis/config.yaml:

<pre> ``
repos:
  # /etc/atlantis/config.yaml
# Lista de repositórios permitidos (ajuste para o seu org/repo)
repo-allowlist: "github.com/seuRepo"
repos:
  - id: "github.com/seuRepo"
	allowed_overrides: [workflow]
	allow_custom_workflows: true
default-tf-version: "1.13.1"   # se preferir, pode usar "v1.13.1"; use a que não gerar erro
disable-autoplan: true - -sua preferencia aqui
allow-commands: "all"
allow_destroy_plan: true
Autenticação via GitHub App (top-level, sem 'github:'!)
gh-app-id: (seu app id aqui )
gh-app-key-file: "/etc/atlantis/atlantsteste.private-key.pem"
gh-webhook-secret: "sua secrete aqui"
**Obrigatório para GitHub App clonar repos**
write-git-creds: true

(opcional, mas recomendável)
atlantis-url: "https://f3fe78057837.ngrok-free.app"
 log-level: "debug"
</pre>

## 4. Criar o serviço systemd
Crie o arquivo /etc/systemd/system/atlantis.service com o seguinte conteúdo:

<pre> ``
[Unit]
Description=Atlantis Terraform Pull Request Automation
After=network.target
[Service]
User=atlantis
Group=atlantis
Environment=HOME=/etc/atlantis
ExecStart=/usr/local/bin/atlantis server --config=/etc/atlantis/config.yaml /etc/atlantis/repos.yaml
StandardOutput=append:/var/log/atlantis/atlantis.log
StandardError=append:/var/log/atlantis/atlantis.err
Restart=always
[Install]
WantedBy=multi-user.target
</pre>

### 4.1 Ativar e iniciar o serviço
<pre> ``
sudo systemctl daemon-reload
sudo systemctl enable atlantis
sudo systemctl start atlantis
sudo systemctl status atlantis
</pre>


## 5. Instalar o ngrok
No Ubuntu:
<pre> ``
sudo snap install ngrok
Ou com apt (se snap não estiver disponível):
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update
sudo apt install ngrok
ngrok config add-authtoken SEU_TOKEN_AQUI
Expor a porta do Atlantis
O Atlantis por padrão roda na porta 4141.

### 5.1 Rode o ngrok assim:
ngrok http 4141

pegar o endereço público
O ngrok vai mostrar algo como:
Forwarding	https://7a1b-189-55-123-45.ngrok-free.app -> http://localhost:4141
</pre>
segue um exemplo de um projeto terrorm criando uma ec2 na aws 

<img width="390" height="336" alt="image" src="https://github.com/user-attachments/assets/0916a7da-180d-48c3-8780-66cea4ebcb4f" />




arquuivo em /modules/ec2

**main.tf**

 <pre> ``` module terraform {
  required_version = ">= 1.3.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

# AMI Amazon Linux 2023 (só usada se var.ami == null)
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["137112412989"] # Amazon

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"] # ajuste para arm64 se quiser
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}

locals {
  ec2_tags = merge(
    {
      Name = var.name
    },
    var.tags
  )

  ami_id = coalesce(var.ami, data.aws_ami.al2023.id)
}

resource "aws_security_group" "this" {
  name        = "${var.name}-sg"
  description = "SG para ${var.name}"
  vpc_id      = var.vpc_id

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.ssh_ingress_cidr_blocks
  }

  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = local.ec2_tags
}

resource "aws_instance" "this" {
  ami                         = local.ami_id
  instance_type               = var.instance_type
  subnet_id                   = var.subnet_id
  vpc_security_group_ids      = [aws_security_group.this.id]
  associate_public_ip_address = var.associate_public_ip
  key_name                    = var.key_name
  user_data                   = var.user_data

  root_block_device {
    volume_size = var.root_volume_size
    volume_type = var.root_volume_type
    encrypted   = true
  }

  tags = local.ec2_tags
}


**outputs.tf**



output "instance_id" {
  description = "ID da instância."
  value       = aws_instance.this.id
}

output "public_ip" {
  description = "IP público (se houver)."
  value       = aws_instance.this.public_ip
}

output "private_ip" {
  description = "IP privado."
  value       = aws_instance.this.private_ip
}

output "sg_id" {
  description = "Security Group ID."
  value       = aws_security_group.this.id
}



variable "name" {
  description = "Nome base da instância (usado em Name tag e SG)."
  type        = string
}

variable "vpc_id" {
  description = "ID da VPC onde criar o SG."
  type        = string
}

variable "subnet_id" {
  description = "Subnet onde a EC2 será criada."
  type        = string
}

variable "instance_type" {
  description = "Tipo da instância (ex: t3.micro)."
  type        = string
  default     = "t3.micro"
}

variable "ami" {
  description = "AMI a ser usada. Se null, usa Amazon Linux 2023 mais recente."
  type        = string
  default     = null
}

variable "associate_public_ip" {
  description = "Se deve associar IP público."
  type        = bool
  default     = true
}

variable "key_name" {
  description = "Nome da key pair existente para SSH. Se null, sem key."
  type        = string
  default     = null
}

variable "ssh_ingress_cidr_blocks" {
  description = "CIDRs autorizados a acessar SSH (porta 22)."
  type        = list(string)
  default     = ["0.0.0.0/0"]
}

variable "root_volume_size" {
  description = "Tamanho do volume root (GB)."
  type        = number
  default     = 16
}

variable "root_volume_type" {
  description = "Tipo do volume root."
  type        = string
  default     = "gp3"
}

variable "user_data" {
  description = "User data (script cloud-init)."
  type        = string
  default     = null
}

variable "tags" {
  description = "Tags adicionais."
  type        = map(string)
  default     = {}
}


arquuivos environments/dev/ec2



main.tf 


terraform {
  required_version = ">= 1.3.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}



resource "aws_key_pair" "default" {
  key_name   = "minha-key"
  public_key = var.public_key
}



module "vpc" {
  source = "../../../modules/vpc"

  name                 = "${var.project_name}-vpc"
  vpc_cidr             = var.vpc_cidr
  azs                  = var.azs
  public_subnet_cidrs  = var.public_subnet_cidrs
  private_subnet_cidrs = var.private_subnet_cidrs
  enable_nat_gateway   = var.enable_nat_gateway
  single_nat_gateway   = var.single_nat_gateway

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

module "ec2_web" {
  source = "../../../modules/ec2"

  name                 = "${var.project_name}-ec2"
  vpc_id               = module.vpc.vpc_id
  subnet_id            = module.vpc.public_subnet_ids[0] # pública
  instance_type        = var.instance_type
  ami                  = var.ami # pode deixar null para usar Amazon Linux 2023
  associate_public_ip  = true
  key_name             = var.key_name
  ssh_ingress_cidr_blocks = var.ssh_ingress_cidr_blocks
  root_volume_size     = 16

  user_data = null
    
  tags = {
    Environment = var.environment
    Project     = var.project_name
    Role        = "web"
  }
}

output "ec2_public_ip" {
  value = module.ec2_web.public_ip
}



terraform.tfvars


aws_region           = "us-east-1"

project_name         = "meuprojeto-dev"
environment          = "dev"

vpc_cidr             = "10.0.0.0/16"
azs                  = ["us-east-1a", "us-east-1b"]
public_subnet_cidrs  = ["10.0.0.0/24", "10.0.1.0/24"]
private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]

enable_nat_gateway   = true
single_nat_gateway   = true

instance_type        = "t3.micro"
ami                  = null        # usa Amazon Linux 2023
key_name             = "minha-key" # precisa existir
ssh_ingress_cidr_blocks = ["0.0.0.0/0"] # troque para seu IP público

public_key = "ssh-rsa su chave publica aqui "


variables.tf



variable "aws_region" { type = string }

variable "project_name" { type = string }
variable "environment"  { type = string }

variable "vpc_cidr" { type = string }
variable "azs" { type = list(string) }
variable "public_subnet_cidrs" { type = list(string) }
variable "private_subnet_cidrs" { type = list(string) }

variable "enable_nat_gateway" {
  type    = bool
  default = true
}
variable "single_nat_gateway" {
  type    = bool
  default = true
}

# EC2-related
variable "instance_type" {
  type    = string
  default = "t3.micro"
}
variable "ami" {
  type    = string
  default = null
}
variable "key_name" {
  type    = string
  default = null
}
variable "ssh_ingress_cidr_blocks" {
  type    = list(string)
  default = ["0.0.0.0/0"]
}

variable "public_key" {
  type = string
}  </pre>

no comentario do  PR digite: atlantis plan -p dev-exemploec2  (no seu caso ponhao o nome do seu projeto que esteja no atlantis.yaml )


<img width="904" height="521" alt="image" src="https://github.com/user-attachments/assets/928fadd0-dd8c-45ab-bce7-fa187dcbb1e6" />

