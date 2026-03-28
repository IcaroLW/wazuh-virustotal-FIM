# Writeup — Wazuh FIM + VirusTotal Integration Lab

**Autor:** Ícaro de Lima Wanzeler  
**Data:** Março de 2026  
**Categoria:** Blue Team / Threat Detection / SIEM  

---

## Resumo

Este writeup documenta o processo de integração do módulo de File Integrity Monitoring (FIM) do Wazuh com a API do VirusTotal, com o objetivo de enriquecer automaticamente alertas de alteração de arquivo com dados de reputação de hash. O laboratório simula um cenário realista de detecção Blue Team onde um arquivo suspeito é introduzido em um diretório monitorado e identificado pelo SIEM.

---

## Motivação

Em ambientes SOC reais, alertas de FIM isolados têm valor limitado — eles indicam que algo mudou, mas não dizem se a mudança é maliciosa. A integração com o VirusTotal resolve esse gap ao adicionar contexto de threat intelligence diretamente ao alerta, permitindo que o analista tome decisões mais rápidas e embasadas sem sair do dashboard.

---

## Ambiente do Laboratório

**Infraestrutura:**
- Host: Notebook com 10 GB RAM, VirtualBox
- VM SIEM: Ubuntu Server com Wazuh Manager + Indexer + Dashboard (~5,7 GB RAM alocados)
- VM Agente: Ubuntu Desktop (ubuntu1, Agent ID 001)

**Versões:**
- Wazuh 4.x
- VirusTotal API v3 (chave gratuita)

**Rede:**
- Adaptador Host-Only para comunicação manager ↔ agente
- Adaptador NAT para acesso à internet (necessário para consultas ao VirusTotal)

---

## Passo a Passo

### 1. Configuração da integração no Wazuh Manager

O primeiro passo foi adicionar o bloco de integração no arquivo de configuração principal do manager:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Bloco adicionado:

```xml
<integration>
  <n>virustotal</n>
  <api_key>SUA_CHAVE_AQUI</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

Evidência da configuração do manager: 
<img width="1919" height="1023" alt="virustotalapi" src="https://github.com/user-attachments/assets/3a1d5803-fead-4a91-bbf3-5006dc2c3f24" />

**Por que `group>syscheck`?** O Wazuh classifica alertas de FIM no grupo `syscheck`. Ao apontar a integração para esse grupo, garantimos que toda detecção de mudança de arquivo dispare automaticamente uma consulta ao VirusTotal com o hash do arquivo envolvido.

Após salvar, reiniciamos o manager:

```bash
sudo systemctl restart wazuh-manager
```

### 2. Configuração do FIM no agente

No agente Ubuntu, o diretório alvo foi adicionado ao monitoramento em tempo real no `ossec.conf` do agente:

```xml
<directories realtime="yes" check_all="yes">/media/user/software</directories>
```

O parâmetro `realtime="yes"` garante que eventos sejam reportados imediatamente, sem aguardar o ciclo padrão de verificação periódica do syscheck.

### 3. Simulação do ataque

Para simular a introdução de um arquivo malicioso no diretório monitorado, utilizamos o arquivo de teste EICAR — um padrão da indústria reconhecido por ferramentas de segurança como potencialmente malicioso, sem representar risco real:

```bash
# No agente Ubuntu
sudo apt install curl
sudo chmod 777 /media/user/software/
sudo curl -Lo /media/user/software/suspicious-file.exe https://secure.eicar.org/eicar.com
```

O arquivo foi nomeado deliberadamente como `suspicious-file.exe` para reforçar o caráter suspeito da simulação — um executável Windows depositado em um diretório de software em um sistema Linux.

---

Evidência do arquivo baixado:
<img width="1916" height="1021" alt="baixando app ubuntu" src="https://github.com/user-attachments/assets/bc11bccc-0ffc-4741-b420-2dcb8d268271" />


## Análise da Detecção

### Alerta gerado no Wazuh Dashboard

Após a modificação do arquivo, o Wazuh FIM registrou o seguinte evento:

| Campo | Valor |
|---|---|
| Agente | ubuntu1 (ID: 001) |
| Regra | 550 — Integrity checksum changed |
| Nível de severidade | 7 |
| Tipo de evento | modified |
| Caminho | `/media/user/software/suspicious-file.exe` |
| Owner (após) | root |
| Timestamp | 26/03/2026 @ 02:27:40 UTC |

### Filtros aplicados na investigação

Para isolar o evento no dashboard, foram aplicados os seguintes filtros:

- `rule.groups: syscheck`
- `agent.id: 001`
- `syscheck.uname_after: root`
- `syscheck.event: modified`

Resultado: 1 hit — o evento de modificação do arquivo suspeito, identificado de forma precisa.

Evidência do Ocorrido:
<img width="1919" height="1079" alt="integração com virus total bem feita" src="https://github.com/user-attachments/assets/88a7f9ad-0e8c-4acc-86ec-c3c133a9e2c1" />

### Enriquecimento via VirusTotal

A integração automaticamente consultou o hash do arquivo na API do VirusTotal no momento do alerta, retornando dados de reputação que seriam exibidos no campo de resposta da integração — permitindo ao analista verificar imediatamente se o hash é conhecido como malicioso, sem necessidade de consulta manual.

---

## Mapeamento MITRE ATT&CK

| Técnica | ID | Relevância no cenário |
|---|---|---|
| Indicator Removal | T1070 | Modificação de arquivo em diretório monitorado |
| Masquerading | T1036 | Executável .exe depositado em sistema Linux |
| Data from Local System | T1005 | Acesso e modificação de arquivo local |

---

## Dificuldades Encontradas

**Configuração do ossec.conf:** A tag correta para o nome da integração é `<n>`, não `<name>` — diferença sutil que pode impedir a integração de funcionar corretamente.

**Permissões do diretório:** O diretório `/media/user/software/` precisou de permissão explícita (`chmod 777`) para que o curl pudesse escrever o arquivo como root. Em ambiente real, o monitoramento seria aplicado em diretórios com permissões mais restritivas.

**Chave gratuita do VirusTotal:** A API gratuita tem limite de requisições por minuto. Para ambientes com alto volume de alertas FIM, uma chave premium seria necessária para evitar throttling.

---

## Conclusão

A integração FIM + VirusTotal transforma um alerta simples de mudança de arquivo em um sinal de detecção enriquecido com contexto de threat intelligence. Para um analista SOC, isso significa menos tempo de triagem manual e maior velocidade de resposta.

O projeto demonstra na prática:
- Configuração de monitoramento em tempo real com Wazuh FIM
- Integração nativa com APIs de threat intelligence
- Capacidade de simular e documentar cenários de detecção com mapeamento MITRE ATT&CK

---

## Referências

- [Documentação Wazuh FIM](https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html)
- [Integração Wazuh + VirusTotal](https://documentation.wazuh.com/current/user-manual/capabilities/malware-detection/virus-total-integration.html)
- [Arquivo de teste EICAR](https://www.eicar.org/download-anti-malware-testfile/)
- [MITRE ATT&CK T1070](https://attack.mitre.org/techniques/T1070/)
- [MITRE ATT&CK T1036](https://attack.mitre.org/techniques/T1036/)
