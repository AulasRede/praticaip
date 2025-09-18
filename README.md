# Laboratório de Redes com 2 VMs Ubuntu Server 24.04 LTS na AWS (EC2)

---

## 📚 1) Objetivos de Aprendizagem
Ao final desta atividade, você será capaz de:
- Identificar e explicar o **MAC address** de uma interface virtual na nuvem (EC2/ENI)
- Visualizar e **configurar endereços IP** (incluindo endereço privado secundário via Console AWS)
- Diferenciar **IP público vs IP privado** no contexto de VPC/EC2
- Executar e interpretar **traceroute** (equivalente Linux ao TRACERT do Windows)
- Revisar e aplicar conceitos de **máscara de rede (CIDR)**
- Ler e modificar **tabelas de roteamento** (no SO e na VPC)
- Entender e observar o funcionamento do **ARP** (cache de vizinhos)
- Configurar e testar **conectividade de rede** entre VMs
- Analisar **tráfego de rede** básico com ferramentas do Linux

---

## ⚙️ 2) Pré-requisitos
- Estar inscrito na AWS Academy e no curso do Learn Lab
- Conhecimentos básicos de redes TCP/IP
- Familiaridade com linha de comando Linux
- Cliente SSH instalado (PuTTY no Windows ou terminal no Mac/Linux)

---

## 🏗️ 3) Visão Geral da Arquitetura
Você criará **duas instâncias** Ubuntu Server 24.04 LTS (Noble) na **mesma subnet** IPv4 de um VPC. Cada instância terá um IP **privado** e (por praticidade) um IP **público** automático para SSH pela Internet.

```
Seu Dispositivo (Internet)
        |
        |  (SSH ou console Web)
        v
+-------------------+      VPC (ex.: 172.31.0.0/16)      +-------------------+
|  EC2: vm-a        |<---- mesma subnet (ex.: /20) ----->|  EC2: vm-b        |
|  Ubuntu 24.04     |                                     |  Ubuntu 24.04     |
|  IP privado: A_p  |                                     |  IP privado: B_p  |
|  IP público: A_P  |                                     |  IP público: B_P  |
+-------------------+                                     +-------------------+
```

---

## 🛠️ 4) Parte 0 – Preparação no **AWS Console**

### 4.1 Configuração Inicial
1. **Escolha a Região** (ex.: `us-east-1`)
2. **Crie um Key Pair**: 
   - Vá em *EC2 > Key Pairs > Create key pair*
   - Nome: `lab-redes-keypair`
   - Tipo: RSA, formato `.pem`
   - **IMPORTANTE**: Guarde o arquivo em local seguro

### 4.2 Criação do Security Group
3. **Crie um Security Group (SG)**: *EC2 > Security Groups > Create security group*
   - Nome: `sg-lab-redes`
   - Descrição: `Security Group para Laboratório de Redes`
   - **Inbound rules**:
     - **SSH (TCP 22)**: *Source* = **My IP** (seu IP público atual)
     - **ICMP - Echo Request (IPv4)**: *Source* = **sg-lab-redes** (auto-referência) – permite **ping** entre as VMs
     - **All ICMP - IPv4**: *Source* = **sg-lab-redes** (para traceroute funcionar)
   - **Outbound**: **All traffic** (padrão)

### 4.3 Lançamento das Instâncias
4. **Lance 2 Instâncias**: *EC2 > Instances > Launch instances*
   
   **Para vm-a:**
   - **Name**: `vm-a`
   - **AMI**: *Ubuntu Server 24.04 LTS (HVM), SSD Volume Type*
   - **Tipo**: `t3.micro` (ou `t2.micro`)
   - **Key pair**: `lab-redes-keypair`
   - **Network**: VPC padrão
   - **Subnet**: escolha uma subnet disponível
   - **Auto-assign public IP**: **Enable**
   - **Security Group**: `sg-lab-redes`
   - **Storage**: padrão (8 GB)
   - **Launch**

   **Para vm-b:** Repita o processo, alterando apenas o nome para `vm-b` e **mantendo a mesma subnet**

5. **Aguarde** até ambas ficarem no estado `Running`
6. **Anote os IPs** (vá em *EC2 > Instances* e clique em cada VM):
   - `vm-a`: IP público: _________ | IP privado: _________
   - `vm-b`: IP público: _________ | IP privado: _________

---

## 🔐 5) Parte 1 – Acesso às VMs e Preparação

### 5.1 Conexão SSH
No seu terminal local (ajuste o caminho para sua chave):

```bash
# Para vm-a
ssh -i /caminho/para/lab-redes-keypair.pem ubuntu@IP_PUBLICO_VM_A

# Em outro terminal, para vm-b
ssh -i /caminho/para/lab-redes-keypair.pem ubuntu@IP_PUBLICO_VM_B
```

### 5.2 Instalação de Ferramentas
Em **ambas as VMs**, execute:

```bash
# Atualizar pacotes
sudo apt update

# Instalar utilitários de rede
sudo apt install -y traceroute ipcalc iputils-arping net-tools tcpdump nmap

# Verificar instalação
traceroute --version
ipcalc --version
```

**📋 ENTREGÁVEL 1**: Screenshot mostrando a conexão SSH bem-sucedida para ambas as VMs e a instalação dos pacotes.

---

## 🔗 6) Parte 2 – **Camada de Enlace (L2)**

### 6.1 Visualização de **MAC Address**
Em **cada VM**, execute:

```bash
# Comando moderno
ip -brief link

# Comando tradicional
ifconfig

# Apenas a interface principal
ip link show eth0
```


### 6.2 **ARP** (Address Resolution Protocol)

#### 6.2.1 Limpeza e Observação Inicial
Na `vm-a`:
```bash
# Limpar cache ARP
sudo ip neigh flush dev eth0

# Verificar cache vazio
ip neigh show
```

#### 6.2.2 Gerando Tráfego ARP
```bash
# Ping para vm-b (substitua IP_PRIVADO_VM_B pelo IP real)
ping -c 3 IP_PRIVADO_VM_B

# Verificar entrada ARP criada
ip neigh show
```

#### 6.2.3 ARP Direto
```bash
# Gerar ARP request direto
sudo arping -c 3 IP_PRIVADO_VM_B

# Verificar novamente
ip neigh show dev eth0
```

**📋 ENTREGÁVEL 2**: 
- Screenshot do comando `ip -brief link` em ambas as VMs mostrando os MAC addresses
- Screenshot do comando `ip neigh show` após o ping, mostrando a entrada ARP da outra VM

---

## 🌐 7) Parte 3 – **Camada de Rede (L3)**

### 7.1 Visualização e Entendimento de IPs

#### 7.1.1 Comandos Básicos
Em cada VM:
```bash
# Visualizar IPs configurados
ip -brief addr
ip addr show eth0

# IP interno reportado pelo sistema
hostname -I

# IP público (como visto na Internet)
curl -s ifconfig.me ; echo
```

#### 7.1.2 Análise de Configuração
```bash
# Informações detalhadas da interface
ip addr show eth0
ip route show

# Gateway padrão
ip route show default
```


#### 7.1.3 Análise da Subnet
```bash
# Descobrir CIDR da interface
ip -brief addr | grep eth0

# Calcular informações da rede (exemplo com /20)
ipcalc IP_PRIVADO_VM_A/20
```

#### 7.1.4 Exercícios de Cálculo
```bash
# Para diferentes máscaras
ipcalc 172.31.16.0/20
ipcalc 10.0.1.0/24
ipcalc 192.168.0.0/16
```

**📋 ENTREGÁVEL 3**: 
- Screenshot do comando `ipcalc` mostrando os cálculos da sua subnet
- Resposta: Quantos hosts teóricos cabem na sua subnet?

#
### 7.2 **TRACEROUTE/TRACEPATH**

#### 7.2.1 Tráfego Local (Intra-VPC)
```bash
# Para a outra VM (1 hop esperado)
traceroute IP_PRIVADO_VM_B
tracepath IP_PRIVADO_VM_B

# Usando ping com TTL
ping -c 1 -t 1 IP_PRIVADO_VM_B
```

#### 7.2.2 Tráfego Externo
```bash
# Para serviços externos
traceroute 8.8.8.8
traceroute 1.1.1.1
traceroute google.com

# Analisar o primeiro hop (gateway)
traceroute -m 3 8.8.8.8
```

#### 7.2.3 Análise Detalhada
```bash
# Com informações extras
traceroute -I 8.8.8.8  # ICMP em vez de UDP
mtr --no-dns --report-cycles 10 8.8.8.8  # Estatísticas
```

**📋 ENTREGÁVEL 4**: 
- Screenshot do traceroute entre as VMs
- Screenshot do traceroute para 8.8.8.8
- Explicação: Qual é o primeiro hop em cada caso?

### 7.3 **IP Público vs IP Privado**

#### 7.3.1 Conceitos Práticos
```bash
# IP privado (interface local)
ip addr show eth0 | grep 'inet '

# IP público (NAT 1:1)
curl -s ifconfig.me ; echo
curl -s ipinfo.io/ip ; echo

# Teste de conectividade
ping -c 3 IP_PRIVADO_VM_B    # Funciona
ping -c 3 IP_PUBLICO_VM_B    # Não deveria funcionar diretamente
```

#### 7.3.2 Experimento de Conectividade
```bash
# Da sua máquina local, teste:
ping IP_PUBLICO_VM_A    # Deve funcionar


# De vm-a para vm-b via IP público
ping IP_PUBLICO_VM_B    # Analise o resultado
```
---


## 📚 Referências e Material Complementar

- [AWS VPC User Guide](https://docs.aws.amazon.com/vpc/)
- [Ubuntu Networking Documentation](https://ubuntu.com/server/docs/network-configuration)
- [Linux IP Command Cheat Sheet](https://access.redhat.com/sites/default/files/attachments/rh_ip_command_cheatsheet_1214_jcs_print.pdf)
- [Wireshark University](https://www.wiresharkbooks.com/)

---
