# 🚀 Script de Hardening Inicial para Ubuntu

Bem-vindo! Este script automatiza a configuração inicial e a segurança (hardening) de servidores Ubuntu recém-criados. Ele é otimizado para provedores como **Contabo** e **Oracle Cloud**, garantindo que seu servidor fique seguro contra ataques comuns e pronto para uso com apenas um comando.

Ele tem **dois modos**, escolhidos logo no início da execução:

| Modo | Para quê | Acesso root via SSH |
|---|---|---|
| **1) Normal** (padrão) | Servidor de uso geral ou que vai rodar o painel do Dokploy | Bloqueado |
| **2) Dokploy Remote Server** | Servidor que será conectado a um painel Dokploy em outra máquina | Liberado **só por chave** e **só a partir do IP do painel** |

## 📦 O que este script faz e instala?

1. **Atualiza o Sistema:** Roda `apt update` e `apt upgrade` para garantir que todos os pacotes estejam na versão mais recente.

2. **Cria o Usuário `deploy`:** Cria um novo usuário padrão para você usar no dia a dia, com permissões de administrador (`sudo`) sem precisar digitar senha.

3. **Migração Segura de Chaves SSH:** Procura por chaves SSH existentes nos usuários `ubuntu` ou `root` e as copia para o usuário `deploy`, sem apagar chaves que ele já tenha.

4. **Hardening do SSH (Segurança):**
   * Desabilita o acesso direto via `root` (uma das principais portas de entrada para hackers).
   * Se você tiver chaves SSH, desativa o login por senha completamente.
   * Se não tiver chaves, gera uma senha forte e permite o login por senha apenas para o usuário `deploy`.
   * Adiciona um limite de tempo para derrubar conexões inativas.
   * As regras ficam em `/etc/ssh/sshd_config.d/00-hardening.conf`, que tem prioridade sobre os arquivos que o provedor coloca no servidor (ex.: `50-cloud-init.conf`).
   * Antes de reiniciar o SSH, a configuração é validada com `sshd -t`. Se algo estiver errado, o backup é restaurado e o SSH **não** é reiniciado, para você não ficar trancado fora do servidor.

5. **Configura Memória Swap:** Cria **2GB de Swap** para evitar que o servidor trave por falta de memória RAM.

6. **Instala o CrowdSec:** Instala um moderno sistema de proteção (`crowdsec` e `iptables-bouncer`) que bloqueia automaticamente IPs maliciosos que tentam fazer ataques de força bruta.

7. **Configura o Firewall (iptables):** Libera as portas essenciais — um passo crucial para instâncias da Oracle Cloud:
   * **Modo Normal:** **80 (HTTP)**, **443 (HTTPS)** e **3000 (painel do Dokploy)**.
   * **Modo Remote Server:** **80** e **443**. A porta **3000** fica fechada, porque o painel não roda nessa máquina.

### Extras do modo Dokploy Remote Server

* Libera o login do `root` **somente por chave SSH** e **somente a partir do IP do painel** do Dokploy (o Dokploy exige acesso root nos servidores remotos). Para qualquer outro IP, o `root` continua bloqueado.
* Coloca o IP do painel na whitelist do CrowdSec, para ele não ser banido por abrir várias conexões SSH seguidas.
* **Não** adiciona a chave do Dokploy: o próprio painel fornece o comando para isso (veja abaixo).

## 🛠️ Passo a Passo: Como usar

**Passo 1: Acesse seu servidor pela primeira vez**
Acesse seu servidor recém-criado usando o usuário padrão fornecido pela sua hospedagem (geralmente `root` ou `ubuntu`).
```bash
ssh root@SEU_IP_AQUI
```

**Passo 2: Execute o script de instalação**
Cole o comando abaixo no terminal do servidor e aperte `Enter`:
```bash
curl -sSL https://gist.githubusercontent.com/lucasolasz/e75daf8c85bbc1d213d85aa6b2e403b9/raw/ | sudo bash
```

**Passo 3: Escolha o modo**
Antes de alterar qualquer coisa, o script pergunta o modo. Apertar `Enter` escolhe o **Normal**:
```text
Escolha o modo de configuração:
  1) Normal                  root sem SSH, acesso só pelo usuário deploy
  2) Dokploy Remote Server   igual ao normal + root liberado só por chave
                             e só a partir do IP do painel do Dokploy
Opção [1]:
```

No modo **2**, ele também pede o **IP do servidor onde o painel do Dokploy está instalado**. Atenção: **não é o IP da máquina que você está configurando** (o script avisa se você digitar o IP dela por engano). Você pode informar vários IPs separados por vírgula ou uma faixa (ex.: `10.0.0.0/24`).

Em seguida aparece um resumo do que será feito. Confirme com `Enter` e o script roda sozinho até o fim.

### Rodando sem perguntas (automação / cloud-init)

```bash
# Modo normal
curl -sSL https://gist.githubusercontent.com/lucasolasz/e75daf8c85bbc1d213d85aa6b2e403b9/raw/ | sudo bash -s -- --mode normal --yes

# Modo Dokploy Remote Server
curl -sSL https://gist.githubusercontent.com/lucasolasz/e75daf8c85bbc1d213d85aa6b2e403b9/raw/ | sudo bash -s -- --mode remote --dokploy-ip IP_DO_PAINEL --yes
```

| Opção | Descrição |
|---|---|
| `--mode normal\|remote` | Define o modo sem perguntar |
| `--dokploy-ip IP[,IP]` | IP (ou faixa CIDR) do servidor onde roda o painel do Dokploy |
| `-y`, `--yes` | Não pede confirmação |
| `-h`, `--help` | Mostra a ajuda |

## 🔐 Como acessar o servidor após a instalação

Após o script terminar de rodar, preste muita atenção na mensagem final exibida na tela. A partir de agora, **você sempre acessará com o usuário `deploy`**.

Abra um **NOVO** terminal no seu computador (não feche o atual ainda, por segurança) e acesse:
```bash
ssh deploy@SEU_IP_AQUI
```

Existem dois cenários possíveis na hora do login:

* **Cenário A (Você já usava Chave SSH):** O script detectou sua chave e desativou o uso de senhas. Basta usar o comando acima e você entrará direto, com máxima segurança.
* **Cenário B (Você não tinha Chave SSH):** O script gerou uma **senha aleatória de 12 caracteres** e a exibiu no final da execução. Você precisará dessa senha para fazer o login. **Guarde essa senha em um local seguro!**

## 🐳 Conectando ao Dokploy (modo Remote Server)

Depois de rodar o script no modo **2**:

1. **Adicione a chave do Dokploy no root.** No painel, gere/copie o comando que adiciona a chave SSH ao servidor e rode-o **como root** (senão a chave vai para o usuário `deploy`):
   ```bash
   sudo -i
   # cole aqui o comando fornecido pelo Dokploy
   ```
2. **Libere a porta 22 no provedor.** Na Oracle Cloud, confira a *Security List* da VCN: a porta 22 precisa aceitar conexões do IP do painel.
3. **Cadastre o servidor no painel** com usuário `root`, o IP da máquina e porta `22`.
4. **(Opcional) Confira a regra efetiva** do SSH para o IP do painel:
   ```bash
   sudo sshd -T -C user=root,addr=IP_DO_PAINEL,host=dokploy | grep -i permitrootlogin
   ```
   O esperado é `permitrootlogin without-password` (equivalente a `prohibit-password`). Para qualquer outro IP, o resultado deve ser `no`.

> Na aba de segurança do Dokploy podem aparecer alertas sobre UFW e Fail2Ban. É esperado: este script protege o servidor com **iptables + CrowdSec**. Os alertas não impedem a conexão.

## 🔁 Rodando de novo / trocando de modo

O script pode ser executado mais de uma vez na mesma máquina, sem duplicar usuário, swap, chaves ou regras de firewall:

* Rodar no modo **Remote Server** num servidor que já foi configurado no modo Normal apenas **acrescenta** a liberação do root para o painel.
* Rodar no modo **Normal** depois **remove** a liberação do root, remove a whitelist do CrowdSec e reabre a porta 3000.

A cada execução, a configuração anterior do SSH é salva em `/root/ssh-backup-AAAAMMDD-HHMMSS/`.

## 🆘 Se algo der errado

* **Não feche a sessão original** até confirmar o login com o `deploy` em um novo terminal.
* Se perder o acesso, use o console de recuperação do provedor (Oracle: *Cloud Shell / Console Connection*; Contabo: *VNC*) e restaure o backup:
  ```bash
  ls /root/ssh-backup-*
  cp /root/ssh-backup-AAAAMMDD-HHMMSS/sshd_config /etc/ssh/sshd_config
  rm /etc/ssh/sshd_config.d/00-hardening.conf
  sshd -t && systemctl restart ssh
  ```
* Se as atualizações pedirem reboot, o script avisa no final. Rode `sudo reboot` quando puder.

🎉 **Pronto!** Seu servidor agora está atualizado, protegido contra ataques de força bruta e pronto para receber suas aplicações — sozinho ou como Remote Server do Dokploy.
