# 🚀 Script de Hardening Inicial para Ubuntu

Bem-vindo! Este script foi criado para automatizar a configuração inicial e a segurança (hardening) de servidores Ubuntu recém-criados. Ele é otimizado para provedores como **Contabo** e **Oracle Cloud**, garantindo que seu servidor fique seguro contra ataques comuns e pronto para uso com apenas um comando.

## 📦 O que este script faz e instala?

Ao executar este script, ele fará automaticamente o seguinte:

1. **Atualiza o Sistema:** Roda `apt update` e `apt upgrade` para garantir que todos os pacotes estejam na versão mais recente.

2. **Cria o Usuário `deploy`:** Cria um novo usuário padrão para você usar no dia a dia, com permissões de administrador (`sudo`) sem precisar digitar senha.

3. **Migração Segura de Chaves SSH:** Procura por chaves SSH existentes nos usuários `root` ou `ubuntu` e as copia para o novo usuário `deploy`.

4. **Hardening do SSH (Segurança):**
   * Desabilita o acesso direto via `root` (uma das principais portas de entrada para hackers).
   * Se você tiver chaves SSH, ele desativa o login por senha completamente.
   * Se não tiver chaves, ele gera uma senha forte e permite o login por senha apenas para o usuário `deploy`.
   * Adiciona um limite de tempo para derrubar conexões inativas.

5. **Configura Memória Swap:** Cria **2GB de Swap** para evitar que o servidor trave por falta de memória RAM.

6. **Instala o CrowdSec:** Instala um moderno sistema de proteção (`crowdsec` e `iptables-bouncer`) que bloqueia automaticamente IPs maliciosos que tentam fazer ataques de força bruta.

7. **Configura o Firewall (iptables):** Libera as portas essenciais **80 (HTTP)**, **443 (HTTPS)** e **3000 (Dokploy/Apps)** — um passo crucial para instâncias da Oracle Cloud.

## 🛠️ Passo a Passo: Como usar

**Passo 1: Acesse seu servidor pela primeira vez**
Acesse seu servidor recém-criado usando o usuário padrão fornecido pela sua hospedagem (geralmente `root` ou `ubuntu`).
```bash
ssh root@SEU_IP_AQUI
```

**Passo 2: Execute o script de instalação**
Basta colar o comando abaixo no terminal do servidor e apertar `Enter`. O script fará o download e executará toda a configuração automaticamente:
```bash
curl -sSL https://gist.githubusercontent.com/lucasolasz/e75daf8c85bbc1d213d85aa6b2e403b9/raw/ | sudo bash
```

## 🔐 Como acessar o servidor após a instalação

Após o script terminar de rodar, preste muita atenção na mensagem final exibida na tela. O acesso direto ao usuário `root` foi desativado! A partir de agora, **você sempre acessará com o usuário `deploy`**.

Abra um **NOVO** terminal no seu computador (não feche o atual ainda, por segurança) e acesse:
```bash
ssh deploy@SEU_IP_AQUI
```

Existem dois cenários possíveis na hora do login:

* **Cenário A (Você já usava Chave SSH):** O script detectou sua chave e desativou o uso de senhas. Basta usar o comando acima e você entrará direto, com máxima segurança.
* **Cenário B (Você não tinha Chave SSH):** O script gerou uma **senha aleatória de 12 caracteres** e a exibiu no final da execução. Você precisará dessa senha para fazer o login. **Guarde essa senha em um local seguro!**

🎉 **Pronto!** Seu servidor agora está atualizado, protegido contra ataques de força bruta e pronto para receber suas aplicações.
