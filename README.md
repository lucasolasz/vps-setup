# 🚀 Script de Hardening Inicial para Ubuntu/Debian

Bem-vindo! Este script automatiza a configuração inicial e a segurança (hardening) de servidores Ubuntu e Debian recém-criados, em **qualquer provedor de VPS**: Contabo, Oracle Cloud, Hetzner, DigitalOcean, Vultr, AWS, Azure, Google Cloud e outros. Com um único comando, o servidor fica protegido contra ataques comuns e pronto para uso, sozinho ou com o [Dokploy](https://dokploy.com).

Ele tem **três modos**, escolhidos logo no início da execução:

| Modo | Use em | Acesso root via SSH |
|---|---|---|
| **1) Normal** (padrão) | Servidor sem Dokploy | Bloqueado |
| **2) Dokploy Remote Server** | Servidor de qualquer provedor que o painel do Dokploy controla de longe | Liberado **só por chave** e **só a partir do IP do painel** |
| **3) Dokploy Painel** | Servidor onde o painel do Dokploy fica instalado | Liberado **só por chave** e **só a partir da rede interna do Docker** |

Em todos os modos, **você** entra pelo usuário `deploy`. O root só é liberado para o próprio Dokploy, nos modos 2 e 3.

> **Exemplo:** a máquina que roda o painel do Dokploy usa o modo **3**; as máquinas que o painel controla usam o modo **2**; qualquer outro servidor usa o modo **1**.

## 📦 O que este script faz e instala?

1. **Atualiza o Sistema:** roda `apt update` e `apt upgrade` para deixar todos os pacotes na versão mais recente.

2. **Cria o Usuário `deploy`:** usuário do dia a dia, com permissões de administrador (`sudo`) sem precisar digitar senha. Ele também entra no grupo `docker`, para usar `docker` sem `sudo`, como recomenda a documentação do Dokploy.

3. **Migração Segura de Chaves SSH:** copia as **suas** chaves para o usuário `deploy`, sem apagar as que ele já tenha. A origem é o usuário com que você entrou no servidor (veja [Funciona em qualquer provedor](#-funciona-em-qualquer-provedor)). **Chaves do Dokploy não contam** como suas: se contassem, o script acharia que você usa chave e desligaria o login por senha.

4. **Hardening do SSH (Segurança):**
   * Desabilita o acesso direto via `root` (uma das principais portas de entrada para hackers).
   * Se você tiver chaves SSH, desativa o login por senha completamente.
   * Se não tiver chaves, mantém o login por senha para o usuário `deploy` e gera uma senha forte quando cria o usuário.
   * Adiciona um limite de tempo para derrubar conexões inativas.
   * As regras ficam em `/etc/ssh/sshd_config.d/00-hardening.conf`, que tem prioridade sobre os arquivos que o provedor coloca no servidor (ex.: `50-cloud-init.conf`).
   * Antes de reiniciar o SSH, valida a configuração com `sshd -t` **e confere o resultado final com `sshd -T`**: root bloqueado para a internet, senha ligada ou desligada como esperado e root liberado só para o Dokploy. Se algo estiver errado, o backup é restaurado e o SSH **não** é reiniciado, para você não ficar trancado fora do servidor.

5. **Configura Memória Swap:** cria **2GB de Swap** para evitar que o servidor trave por falta de memória RAM. Se o provedor já entrega swap, ela é reaproveitada.

6. **Instala o CrowdSec:** instala um moderno sistema de proteção (`crowdsec` e bouncer de `iptables`) que bloqueia automaticamente IPs maliciosos que tentam fazer ataques de força bruta.

7. **Configura o Firewall do sistema:** libera as portas essenciais.
   * **Modos Normal e Remote Server:** **80 (HTTP)** e **443 (HTTPS)**. A porta **3000** fica fechada.
   * **Modo Painel:** **80**, **443** e **3000 (painel do Dokploy)**.

8. **Instala o comando `dokploy-key`:** autoriza a chave do Dokploy no `root`, seja qual for o usuário com que você está logado (veja [O comando dokploy-key](#-o-comando-dokploy-key)).

### Extras do modo 2 (Dokploy Remote Server)

* Libera o login do `root` **somente por chave SSH** e **somente a partir do IP do painel** do Dokploy (o Dokploy exige acesso root nos servidores remotos). Para qualquer outro IP, o `root` continua bloqueado.
* Aceita IPv4 e IPv6, vários IPs separados por vírgula e faixas CIDR (ex.: o IP público do painel e o IP dele na rede privada do provedor). Liberar a internet inteira (`/0`) é recusado.
* Coloca o IP do painel na whitelist do CrowdSec, para ele não ser banido por abrir várias conexões SSH seguidas.

### Extras do modo 3 (Dokploy Painel)

* O painel do Dokploy roda num container e, para abrir o **terminal do servidor**, entra nesta máquina por SSH como `root`, vindo da rede interna do Docker. Por isso o modo libera o `root` **somente por chave** e **somente a partir dessa rede** (`172.16.0.0/12` por padrão). Para qualquer outro IP, o `root` continua bloqueado.
* Se o Docker já estiver instalado, confere a rede que o painel usa (`docker_gwbridge`) e a inclui se ela estiver fora da faixa padrão.
* Coloca a rede do Docker na whitelist do CrowdSec.
* Se o painel já tiver gerado a chave do terminal, ela é autorizada no `root` automaticamente.

## 🌍 Funciona em qualquer provedor

| O que muda de provedor para provedor | O que o script faz |
|---|---|
| Usuário de entrada (`root`, `ubuntu`, `admin`, `debian`, `azureuser`, o seu usuário no Google Cloud...) | Copia as chaves de quem rodou o `sudo`; se não achar, de outro usuário comum em `/home`; por fim, do `root` |
| Root com a restrição do provedor (`Please login as the user "ubuntu"`) | A restrição não é copiada para o `deploy`, e a chave do Dokploy entra no root numa linha própria, sem ela |
| UFW já ligado (ex.: Vultr) | Mantém o UFW e só acrescenta as regras, inclusive a porta do SSH para o Dokploy |
| Bloqueio no fim do iptables (ex.: `REJECT` da Oracle) | Insere as regras antes do bloqueio |
| Regras de SSH escritas pelo provedor | Põe a leitura do `00-hardening.conf` no topo do `sshd_config` e confere o resultado com `sshd -T`; se o provedor ainda prevalecer, desfaz tudo e mostra as linhas responsáveis |
| SSH fora da porta 22 | Detecta a porta e usa nas regras do UFW e no resumo final |
| Swap já existente | Reaproveita |
| Imagem mínima, sem `sudo` ou `openssl` | Instala o `sudo` e gera a senha de outro jeito |

## 🛠️ Passo a Passo: Como usar

**Passo 1: Acesse seu servidor pela primeira vez**
Acesse seu servidor recém-criado usando o usuário padrão fornecido pela sua hospedagem (geralmente `root` ou `ubuntu`).
```bash
ssh root@SEU_IP_AQUI
```

**Passo 2: Execute o script de instalação**
Cole o comando abaixo no terminal do servidor e aperte `Enter`:
```bash
curl -fsSL https://gist.githubusercontent.com/lucasolasz/e75daf8c85bbc1d213d85aa6b2e403b9/raw/ | sudo bash
```

**Passo 3: Escolha o modo**
Antes de alterar qualquer coisa, o script pergunta o modo. Apertar `Enter` escolhe o **Normal**:
```text
Escolha o modo de configuração:
  1) Normal                  root sem SSH, acesso só pelo usuário deploy
  2) Dokploy Remote Server   máquina de qualquer provedor que o painel controla:
                             igual ao normal + root liberado só por chave e só
                             a partir do IP do painel do Dokploy
  3) Dokploy Painel          servidor onde o painel do Dokploy fica instalado:
                             igual ao normal + root liberado só por chave e só
                             da rede interna do Docker (terminal do painel)
Opção [1]:
```

No modo **2**, ele também pede o **IP do servidor onde o painel do Dokploy está instalado**. Atenção: **não é o IP da máquina que você está configurando** (o script avisa se você digitar um IP dela por engano).

Em seguida aparece um resumo do que será feito, com o modo, as portas, a porta do SSH e o firewall que será usado. Confirme com `Enter` e o script roda sozinho até o fim.

### Rodando sem perguntas (automação / cloud-init)

```bash
# Modo normal
curl -fsSL https://gist.githubusercontent.com/lucasolasz/e75daf8c85bbc1d213d85aa6b2e403b9/raw/ | sudo bash -s -- --mode normal --yes

# Modo Dokploy Remote Server
curl -fsSL https://gist.githubusercontent.com/lucasolasz/e75daf8c85bbc1d213d85aa6b2e403b9/raw/ | sudo bash -s -- --mode remote --dokploy-ip IP_DO_PAINEL --yes

# Modo Dokploy Painel
curl -fsSL https://gist.githubusercontent.com/lucasolasz/e75daf8c85bbc1d213d85aa6b2e403b9/raw/ | sudo bash -s -- --mode painel --yes
```

| Opção | Descrição |
|---|---|
| `--mode normal\|remote\|painel` | Define o modo sem perguntar (também aceita `1`, `2` e `3`) |
| `--dokploy-ip IP[,IP]` | (modo 2) IP do servidor onde roda o painel do Dokploy: IPv4 ou IPv6, vários separados por vírgula, aceita CIDR |
| `--docker-cidr REDE[,REDE]` | (modo 3) Rede interna do Docker de onde o terminal do painel conecta (padrão `172.16.0.0/12`) |
| `-y`, `--yes` | Não pede confirmação |
| `-h`, `--help` | Mostra a ajuda |

## 🔐 Como acessar o servidor após a instalação

Após o script terminar de rodar, preste muita atenção na mensagem final exibida na tela. A partir de agora, **você sempre acessará com o usuário `deploy`**.

Abra um **NOVO** terminal no seu computador (não feche o atual ainda, por segurança) e acesse:
```bash
ssh deploy@SEU_IP_AQUI
```
Se o SSH do servidor não estiver na porta 22, use `ssh -p PORTA deploy@SEU_IP_AQUI`. A mensagem final mostra o comando certo.

Existem dois cenários possíveis na hora do login:

* **Cenário A (Você já usava Chave SSH):** O script copiou sua chave e desativou o uso de senhas. Basta usar o comando acima e você entrará direto, com máxima segurança.
* **Cenário B (Você não tinha Chave SSH):** O script gerou uma **senha aleatória de 16 caracteres** e a exibiu no final da execução. Você precisará dessa senha para fazer o login. **Ela só aparece uma vez: guarde-a em um local seguro!**

## 🐳 Conectando ao Dokploy

### Servidor do painel (modo 3)

1. **Instale o Dokploy**, se ainda não estiver instalado:
   ```bash
   curl -sSL https://dokploy.com/install.sh | sudo sh
   ```
2. **Abra o terminal do servidor no painel uma vez.** Na primeira vez ele falha e mostra um comando: é nesse momento que o Dokploy gera a chave do terminal.
3. **Em vez de colar o comando do painel, rode no servidor:**
   ```bash
   sudo dokploy-key
   ```
4. **Reabra o terminal no painel.** Ele entra como `root`, na porta 22 (o padrão do painel). Se o SSH do servidor usar outra porta, ajuste nas configurações de conexão da própria janela do terminal.

Para conferir a regra efetiva:
```bash
sudo sshd -T -C user=root,addr=172.18.0.2,host=dokploy | grep -i permitrootlogin
```
O esperado é `permitrootlogin without-password` (equivalente a `prohibit-password`).

### Remote Server (modo 2)

1. **Autorize a chave do painel no root.** Copie a chave que o painel mostra e rode, logado com qualquer usuário:
   ```bash
   sudo dokploy-key "ssh-rsa AAAA... dokploy"
   ```
2. **Libere as portas no firewall do provedor**, se ele tiver um (*Security List* na Oracle, *Security Group* na AWS, *Cloud Firewall* na Hetzner, DigitalOcean ou Vultr...): a porta do SSH para o IP do painel e as portas 80 e 443 para todos.
3. **Cadastre o servidor no painel** com usuário `root`, o IP da máquina e a porta do SSH (a mensagem final mostra os dois).
4. **(Opcional) Confira a regra efetiva** do SSH para o IP do painel:
   ```bash
   sudo sshd -T -C user=root,addr=IP_DO_PAINEL,host=dokploy | grep -i permitrootlogin
   ```
   O esperado é `permitrootlogin without-password` (equivalente a `prohibit-password`). Para qualquer outro IP, o resultado deve ser `no`.

> Na aba de segurança do Dokploy podem aparecer alertas sobre UFW e Fail2Ban. É esperado: este script protege o servidor com **iptables + CrowdSec** (ou com o UFW do provedor, quando ele já vem ligado). Os alertas não impedem a conexão.
>
> Se a máquina tiver **Fail2Ban** ativo, o script avisa no final: coloque o IP do painel em `ignoreip` no `/etc/fail2ban/jail.local`, senão o Fail2Ban pode banir o Dokploy.

### 🔑 O comando `dokploy-key`

Os comandos que o painel do Dokploy mostra gravam a chave no usuário que os executa. Como você normalmente está logado como `deploy`, a chave iria para o lugar errado, já que o Dokploy entra como `root`. O `dokploy-key` resolve isso: grava a chave **sempre** no `root`, retira a mesma chave dos outros usuários e confere se o SSH libera o root.

| Comando | Quando usar |
|---|---|
| `sudo dokploy-key` | Modo 3: autoriza a chave do terminal do painel. Se ela ainda não existir, pede para você colar a chave ou o comando do painel (termine com `Ctrl+D`) |
| `sudo dokploy-key "ssh-rsa AAAA... dokploy"` | Modo 2: autoriza a chave do painel |
| `sudo dokploy-key 'COMANDO DO PAINEL'` | Também aceita o comando inteiro que o painel mostra, entre aspas simples |
| `sudo dokploy-key < chave.pub` | Lê a chave de um arquivo |

Pode ser rodado como `deploy`, `ubuntu`, `root` ou qualquer outro usuário com `sudo`.

## 🔁 Rodando de novo / trocando de modo

O script pode ser executado mais de uma vez na mesma máquina, sem duplicar usuário, swap, chaves ou regras de firewall. A cada execução, a liberação do `root` é trocada pela do modo escolhido:

* **Normal** remove a liberação do root e a whitelist do CrowdSec.
* **Remote Server** libera o root só para o IP do painel.
* **Painel** libera o root só para a rede interna do Docker.

Uma máquina configurada com uma versão anterior do script não precisa ser reinstalada: rode a versão atual por cima, no modo certo, e ela fica igual a uma máquina nova.

A cada execução, a configuração anterior do SSH é salva em `/root/ssh-backup-AAAAMMDD-HHMMSS/`.

> ⚠️ Em servidores que já rodam aplicações, o `apt upgrade` pode atualizar o Docker e reiniciar os containers por cerca de um minuto. Rode num horário em que isso não atrapalhe.

## 🆘 Se algo der errado

* **Não feche a sessão original** até confirmar o login com o `deploy` em um novo terminal.
* **O script parou com "O SSH não ficaria como esperado nesta máquina"**: algum arquivo do provedor está prevalecendo sobre o script. Nada foi alterado no SSH. Comente as linhas que ele listou e rode de novo.
* **Terminal do painel com "Authentication failed"** (modo 3): rode `sudo dokploy-key` e reabra o terminal. Se o erro continuar, confira se a máquina foi configurada no modo **3**.
* **Remote Server com "Authentication failed"** (modo 2): a chave provavelmente foi parar no usuário errado. Rode `sudo dokploy-key "CHAVE DO PAINEL"`. Se der *timeout* em vez de erro de autenticação, o bloqueio está no firewall do provedor.
* **Perdeu o acesso?** Use o console web do provedor (Contabo: *VNC*; Oracle: *Cloud Shell* / *Console Connection*; Hetzner, DigitalOcean e Vultr: *Console* no painel) e restaure o backup:
  ```bash
  ls /root/ssh-backup-*
  cp /root/ssh-backup-AAAAMMDD-HHMMSS/sshd_config /etc/ssh/sshd_config
  rm /etc/ssh/sshd_config.d/00-hardening.conf
  sshd -t && systemctl restart ssh
  ```
* Se as atualizações pedirem reboot, o script avisa no final. Rode `sudo reboot` quando puder.

🎉 **Pronto!** Seu servidor agora está atualizado, protegido contra ataques de força bruta e pronto para receber suas aplicações: sozinho, como painel do Dokploy ou como Remote Server.
