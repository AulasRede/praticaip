# Laboratório de Redes com 2 VMs Ubuntu Server 24.04 LTS na AWS (EC2)

> **Importante**: Este laboratório será realizado **no ambiente AWS Academy**. A **criação das VMs** ocorrerá **presencialmente e de forma interativa durante a aula**, e o **acesso às instâncias será feito via Console Web (EC2 Instance Connect)** — **não** use SSH local ou chaves no seu computador.
>
> **Nomenclatura usada neste roteiro:**
> - **vm-a**: primeira instância EC2 do par de trabalho.
> - **vm-b**: segunda instância EC2 do par de trabalho.
> - **ambas as VMs**: execute **em vm-a e em vm-b** (cada uma na sua sessão do Console Web).
>
> Sempre que houver um bloco de comandos, ele virá precedido de um **selo** indicando **onde executar**: 🅰️ = vm-a, 🅱️ = vm-b, 🔁 = ambas as VMs.

---

## 📚 1) Objetivos de Aprendizagem
Ao final desta atividade, você será capaz de:
- Identificar e explicar o **MAC address** de uma interface virtual na nuvem (EC2/ENI).
- Visualizar e **configurar endereços IP**.
- Diferenciar **IP público vs IP privado** no contexto de VPC/EC2.
- Executar e interpretar **traceroute/tracepath** (equivalente Linux ao TRACERT do Windows).
- Revisar e aplicar conceitos de **máscara de rede (CIDR)**.
- Entender e observar o funcionamento do **ARP** (cache de vizinhos).
- Configurar e testar **conectividade de rede** entre VMs.

---

## ⚙️ 2) Pré-requisitos
- Estar inscrito na **AWS Academy** (Learner Lab ativo).
- Conhecimentos básicos de redes (camadas de enlace e rede).
- Familiaridade com linha de comando Linux.

---

## 🏗️ 3) Visão Geral da Arquitetura
Você utilizará **duas instâncias** Ubuntu Server 24.04 LTS (Noble) na **mesma subnet** IPv4 de uma VPC. Cada instância terá um IP **privado** e, se necessário, um IP **público** (definido durante a aula).

```
Seu Dispositivo (Navegador)
        |
        |  (Console Web: EC2 Instance Connect)
        v
+-------------------+      VPC (ex.: 172.31.0.0/16)      +-------------------+
|  EC2: vm-a        |<---- mesma subnet (ex.: /20) ----->|  EC2: vm-b        |
|  Ubuntu 24.04     |                                     |  Ubuntu 24.04     |
|  IP privado: A_p  |                                     |  IP privado: B_p  |
|  IP público: A_P  |                                     |  IP público: B_P  |
+-------------------+                                     +-------------------+
```

---

## 🛠️ 4) Preparação das VMs (realizada em sala)

A **criação e configuração inicial das VMs** (VPC, sub-rede, security groups, atributos de rede, IP público/privado, etc.) serão **demonstradas e executadas presencialmente** com o professor **no ambiente AWS Academy**. Não execute esta etapa por conta própria antes da aula.

---

## 🔐 5) Parte 1 – Acesso via **Console Web** e Preparação do Ambiente

### 5.1 Conectar via EC2 Instance Connect (Console Web)
1. No **AWS Console (Academy)**, acesse **EC2 → Instances**.  
2. Selecione sua instância **vm-a** e clique em **Connect → EC2 Instance Connect → Connect**.  
3. Abra **nova aba/guia** e conecte-se também na **vm-b** pelo mesmo caminho.


### 5.2 Instalar utilitários de rede
🔁 **Executar em ambas as VMs (vm-a e vm-b):**
```bash
sudo apt update
sudo apt install -y traceroute ipcalc iputils-arping net-tools tcpdump nmap mtr-tiny curl
traceroute --version || which traceroute
ipcalc --version || ipcalc -h | head -n 1
mtr --version || mtr -v
```

**📋 ENTREGÁVEL 1**: Screenshot (em **cada VM**) mostrando os comandos de verificação da instalação.

---

## 🔗 6) Parte 2 – **Camada de Enlace (L2)**

### 6.1 Visualização de **MAC Address**
🔁 **Executar em ambas as VMs:**
```bash
ip -brief link          # nomes reais das interfaces (ex.: ens5)
ifconfig                # visão tradicional
ip link show <nome_da_interface_real>
```

### 6.2 **ARP** (Address Resolution Protocol)

#### 6.2.1 Limpeza e observação inicial
🅰️ **Executar em vm-a:**
```bash
sudo ip neigh flush dev <nome_da_interface_real>
ip neigh show
```

#### 6.2.2 Gerando tráfego ARP
🅰️ **Executar em vm-a (apontando para vm-b):**
```bash
ping -c 3 IP_PRIVADO_VM_B
ip neigh show
```

#### 6.2.3 ARP direto
🅰️ **Executar em vm-a (apontando para vm-b):**
```bash
sudo arping -c 3 IP_PRIVADO_VM_B
ip neigh show dev <nome_da_interface_real>
```

> (Opcional) Repita os passos **6.2.1 a 6.2.3** invertendo os papéis, ou seja, **🅱️ vm-b → vm-a**, para comparar saídas.

**📋 ENTREGÁVEL 2**:  
- Screenshot do `ip -brief link` em **ambas as VMs** (evidenciando os **MAC addresses**).  
- Screenshot do `ip neigh show` na **vm-a** após o ping para a **vm-b** (e, se optar, na vm-b após ping para vm-a).

---

## 🌐 7) Parte 3 – **Camada de Rede (L3)**

### 7.1 Visualização e entendimento de IPs

#### 7.1.1 Comandos básicos
🔁 **Executar em ambas as VMs:**
```bash
ip -brief addr
ip addr show <nome_da_interface_real>
hostname -I
curl -s ifconfig.me ; echo
```

#### 7.1.2 Rotas e gateway
🔁 **Executar em ambas as VMs:**
```bash
ip route show
ip route show default
```

#### 7.1.3 Análise da subnet
🅰️ **Executar em vm-a (exemplo) — personalize para sua rede:**
```bash
# Use o CIDR exibido em 'ip -brief addr' (ex.: 172.31.16.23/20)
ipcalc 172.31.16.23/20
```

> (Opcional) 🅱️ Repita na **vm-b** com o endereço e prefixo dela para comparar.

#### 7.1.4 Exercícios de cálculo (prática guiada)
🔁 **Executar em ambas as VMs (ou apenas em uma, a seu critério):**
```bash
ipcalc 172.31.16.0/20
ipcalc 10.0.1.0/24
ipcalc 192.168.0.0/16
```

**📋 ENTREGÁVEL 3**:  
- Screenshot do `ipcalc` mostrando os cálculos da **sua** subnet (pelo menos na **vm-a**).  
- Resposta breve: **Quantos hosts teóricos** cabem na sua subnet?

### 7.2 **TRACEROUTE / TRACEPATH**

#### 7.2.1 Tráfego local (Intra-VPC)
🅰️ **Executar em vm-a (apontando para vm-b):**
```bash
traceroute IP_PRIVADO_VM_B
tracepath IP_PRIVADO_VM_B
ping -c 1 -t 1 IP_PRIVADO_VM_B
```

> (Opcional) 🅱️ Refaça na **vm-b** em direção à **vm-a**.

#### 7.2.2 Tráfego externo
🔁 **Executar em ambas as VMs (se permitido pelo ambiente):**
```bash
traceroute 8.8.8.8
traceroute 1.1.1.1
traceroute google.com
traceroute -m 3 8.8.8.8
```

#### 7.2.3 Análise detalhada
🔁 **Executar em ambas as VMs:**
```bash
traceroute -I 8.8.8.8
mtr --no-dns --report-cycles 10 8.8.8.8
```

**📋 ENTREGÁVEL 4**:  
- Screenshot do traceroute **entre as VMs** (obrigatório na **vm-a → vm-b**; opcional no sentido inverso).  
- Screenshot do traceroute para **8.8.8.8** (em pelo menos **uma** das VMs).  
- Resposta breve: **Qual é o primeiro hop** em cada caso e por quê?

### 7.3 **IP Público vs IP Privado**
🔁 **Executar em ambas as VMs:**
```bash
ip addr show <nome_da_interface_real> | grep 'inet '
curl -s ifconfig.me ; echo
```

🅰️ **Teste de conectividade vm-a → vm-b (IP privado e, se aplicável, público):**
```bash
ping -c 3 IP_PRIVADO_VM_B
ping -c 3 IP_PUBLICO_VM_B   # Pode não responder, depende do SG/NAT
```

> (Opcional) 🅱️ Repita os testes **vm-b → vm-a** para comparar latências e respostas.

---

## ✅ 8) **Resumo dos Entregáveis** (Envio pelo **Microsoft Teams**)
Envie **um único PDF** no **Microsoft Teams** contendo:
1. **ENTREGÁVEL 1** — Prints da verificação de ferramentas nas **duas VMs**.  
2. **ENTREGÁVEL 2** — Prints do `ip -brief link` (duas VMs) e do `ip neigh show` na **vm-a** após o ping para a **vm-b**.  
3. **ENTREGÁVEL 3** — Print(s) do `ipcalc` da sua subnet (pelo menos na **vm-a**) + **resposta** sobre o número teórico de hosts.  
4. **ENTREGÁVEL 4** — Prints de `traceroute/tracepath` (**vm-a → vm-b** e externo) + **explicação** do primeiro hop em cada cenário.


> **Observação**: Cada print deve incluir o **hostname** ou identificador claro (**vm-a/vm-b**) e o **comando executado** visível.

---

## 🧭 9) Orientações diversas e **erros comuns**
- **Nome da interface ≠ `eth0`**: Em instâncias Ubuntu na AWS, a interface costuma ser `ens5` (ou similar).  
  - Descubra com: `ip -brief link` e utilize **o nome real** nos comandos (ex.: `ip addr show ens5`).  
- **Permissões**: Comandos como `ip neigh flush` exigem **sudo**.  
- **`traceroute` vs `tracepath`**: Se `traceroute` falhar por política de rede, tente `traceroute -I` (ICMP) ou `tracepath`.  
- **ICMP no Security Group**: O **ping** entre VMs requer **ICMP liberado**. Essa regra será configurada **em sala**; se o ping falhar, verifique o SG.  
- **Ping para IP público da outra VM**: Pode **não** responder, a depender do SG/NAT. Use o **IP privado** para testes intra-VPC.  
- **Ferramentas ausentes**: Se `curl`, `ipcalc` ou `mtr` não existirem, instale com `sudo apt install -y ...`.  
- **Tempo de sessão (Console Web)**: O EC2 Instance Connect pode **expirar** por inatividade. Salve/cole os comandos novamente se a sessão cair.  
- **Saída confusa do `ip addr`**: Use `ip -brief addr` para uma visão compacta das interfaces e endereços.  
- **Rotas**: Se não aparecer rota default, verifique `ip route show` e a configuração de gateway (será tratada em aula).

---

## 📚 Referências e Material Complementar
- Documentação AWS VPC: https://docs.aws.amazon.com/vpc/
- Documentação EC2 Instance Connect: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-connect-methods.html
- Ubuntu — Network configuration: https://ubuntu.com/server/docs/network-configuration
- Linux **ip** command cheat sheet (Red Hat): https://access.redhat.com/sites/default/files/attachments/rh_ip_command_cheatsheet_1214_jcs_print.pdf
- Wireshark University: https://www.wiresharkbooks.com/
