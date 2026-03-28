# Wazuh FIM + VirusTotal Integration Lab

> Detecção de alterações em arquivos monitorados com enriquecimento automático via VirusTotal API

## Objetivo

Demonstrar como detectar alterações de integridade em diretórios monitorados utilizando o módulo de File Integrity Monitoring (FIM) do Wazuh, enriquecido com consultas automáticas de threat intelligence via API do VirusTotal — simulando um fluxo real de detecção Blue Team.

## Ambiente

| Componente | Detalhe |
|---|---|
| SIEM | Wazuh 4.x (Manager + Indexer + Dashboard) |
| Agente | Ubuntu Linux (ubuntu1 — Agent ID 001) |
| Diretório monitorado | `/media/user/software/` |
| Threat Intel | VirusTotal API (integração nativa Wazuh) |
| Virtualização | VirtualBox (Host-Only + NAT) |

## Arquitetura

```
[Agente Ubuntu]
  └── FIM monitora /media/user/software/
        └── Alteração de arquivo detectada (syscheck)
              └── [Wazuh Manager]
                    └── Consulta à API do VirusTotal (hash)
                          └── Alerta gerado → Dashboard
```

## Mapeamento MITRE ATT&CK

| Técnica | ID | Descrição |
|---|---|---|
| Indicator Removal | T1070 | Adversário modifica arquivos para evadir detecção |
| Masquerading | T1036 | Arquivo suspeito inserido em diretório de software |
| Data from Local System | T1005 | Alteração de integridade em caminho monitorado |

## Configuração

### 1. Wazuh Manager — ossec.conf

A integração com o VirusTotal foi adicionada em `/var/ossec/etc/ossec.conf`:

```xml
<integration>
  <name>virustotal</name>
  <api_key>SUA_CHAVE_AQUI</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

Isso instrui o manager a encaminhar cada alerta do grupo `syscheck` para a API do VirusTotal para consulta de reputação por hash.

### 2. Agente — monitoramento de diretório

O `ossec.conf` do agente foi configurado para monitorar `/media/user/software/` em tempo real, registrando eventos de criação, modificação e deleção de arquivos.

## Simulação do Ataque

Um arquivo de teste foi introduzido e modificado no diretório monitorado:

```bash
sudo apt install curl
sudo chmod 777 /media/user/software/
sudo curl -Lo /media/user/software/suspicious-file.exe https://secure.eicar.org/eicar.com
```

O arquivo EICAR é um padrão da indústria para validação de ferramentas de segurança sem uso de malware real.

## Resultado da Detecção

O Wazuh FIM detectou o evento de modificação em segundos:

| Campo | Valor |
|---|---|
| Agente | ubuntu1 (ID: 001) |
| Tipo de evento | modified |
| Caminho | `/media/user/software/suspicious-file.exe` |
| Regra | 550 — Integrity checksum changed |
| Nível | 7 |
| Owner | root |
| Timestamp | 26/03/2026 @ 02:27:40 |

## Principais Aprendizados

- O FIM fornece visibilidade em tempo real sobre alterações em diretórios sensíveis
- A integração com o VirusTotal enriquece alertas com dados de reputação de hash, reduzindo o esforço manual de triagem
- O grupo `syscheck` no ossec.conf é o gatilho para as consultas ao VirusTotal
- A regra 550 (nível 7) é um sinal confiável para modificações não autorizadas de arquivos

## Referências

- [Documentação Wazuh FIM](https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html)
- [Integração Wazuh + VirusTotal](https://documentation.wazuh.com/current/user-manual/capabilities/malware-detection/virus-total-integration.html)
- [Arquivo de teste EICAR](https://www.eicar.org/download-anti-malware-testfile/)
- [MITRE ATT&CK T1070](https://attack.mitre.org/techniques/T1070/)
