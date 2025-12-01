# Melhorias Implementadas no opnsense-vm.sh

## 📅 Data: 2025-12-01

## 🐛 Problema Identificado (2025-12-01)

### Sintoma: Comandos retornando "not found" durante instalação

**Erro observado:**
```
-sh: opnsense: not found
-sh: 2: not found
-sh: 1: not found
-sh: n: not found
-sh: 192.168.0.3: not found
-sh: 192.168.0.2: not found
```

**Causa raiz:**
O script estava enviando comandos do OPNsense (como `opnsense`, `2`, `1`, etc.) para a VM **antes da instalação do OPNsense estar completa**. A função `wait_for_opnsense_ready()` apenas esperava um tempo fixo sem verificar se a instalação realmente terminou. Se a instalação demorasse mais de 20 minutos, os comandos eram enviados para o shell do FreeBSD que não os reconhecia.

**Solução implementada:**
- Criada função `read_serial_output()` que lê o socket serial de forma **não-interativa**
- Usa `socat` ou `nc` para conectar ao socket `/var/run/qemu-server/${vmid}.serial0`
- **Não entra na VM** (não usa `qm terminal` que é interativo)
- Verifica padrões no output para detectar quando instalação realmente terminou
- Aplica verificação em 3 pontos críticos:
  1. Download do bootstrap (`wait_for_bootstrap_download`)
  2. Instalação do OPNsense (`wait_for_opnsense_ready`)
  3. Salvamento de configuração (`wait_for_config_saved`)

---

## ✅ Correções Implementadas

### 1. ✅ Item 1: Corrigir typo no caractere 'X'
**Problema:** Linha 152 tinha `"X") character="shift=x"` (faltava hífen)  
**Correção:** Alterado para `"X") character="shift-x"`  
**Impacto:** Agora o caractere 'X' maiúsculo é enviado corretamente via `qm sendkey`

---

### 2. ✅ Item 2: Remover TEMP_DIR duplicado
**Problema:** `TEMP_DIR` era declarado duas vezes (linhas 82 e 175)  
**Correção:** Removida a segunda declaração desnecessária  
**Impacto:** Evita criação de diretórios temporários duplicados e confusão no cleanup

---

### 3. ℹ️ Item 3: get_freebsd_mirror
**Status:** Removido conforme solicitação  
**Nota:** Script voltou a usar URL padrão hardcoded

---

### 4. ℹ️ Item 4: Flag -4 do curl
**Status:** Removido conforme solicitação  
**Nota:** Curl voltou ao comportamento padrão (tenta IPv6 primeiro)

---

### 5. ✅ Item 5: Polling inteligente - wait_for_opnsense_ready()
**Problema:** `sleep 1000` (16min 40s) fixo e arbitrário, sem verificação real de conclusão  
**Correção:** Criada função `wait_for_opnsense_ready()` que:
- **Lê serial console de forma não-interativa** usando `socat` ou `nc`
- Procura por indicadores de instalação completa: `(login:|Username:|FreeBSD.*OPNsense|Enter an option)`
- Verifica a cada 30 segundos
- Mostra progresso a cada minuto
- Timeout padrão: 1200s (20 minutos)
- Retorna erro se timeout excedido

**Método não-interativo:** Criada função auxiliar `read_serial_output()` que:
- Conecta ao socket serial em `/var/run/qemu-server/${vmid}.serial0`
- Usa `socat - UNIX-CONNECT:${socket_path},nonblock` (preferencial)
- Fallback para `nc -U` se socat não disponível
- **Não entra na VM interativamente** (não usa `qm terminal`)
- Retorna output para análise

**Impacto:** 
- Instalação mais rápida quando OPNsense termina antes dos 20 minutos
- Detecção precisa e real de quando está pronto
- Feedback visual do progresso
- **Resolve o problema de "not found"** ao enviar comandos antes da instalação completar

---

### 6. ✅ Item 6: Polling qm status - wait_for_vm_running()
**Problema:** `sleep 90` fixo após `qm start`  
**Correção:** Criada função `wait_for_vm_running()` que:
- Verifica status da VM via `qm status` a cada 5 segundos
- Aguarda até status = "running"
- Timeout padrão: 300s (5 minutos)
- Retorna erro se VM não iniciar

**Código:**
```bash
qm start $VMID
wait_for_vm_running $VMID 300
sleep 30  # Wait for FreeBSD boot process
```

**Impacto:** VM inicia em ~10-20s ao invés de esperar 90s sempre

---

### 7. ✅ Item 7: Verificar download do script - wait_for_bootstrap_download()
**Problema:** Sem verificação se download foi bem-sucedido antes de executar  
**Correção:** Criada função `wait_for_bootstrap_download()` que:
- Lê output do serial console de forma não-interativa
- Procura por indicadores de sucesso: `(opnsense-bootstrap\.sh\.in.*100%|opnsense-bootstrap\.sh\.in.*saved|root@freebsd)`
- Detecta erros de download: `(fetch.*failed|unable to fetch|no route to host)`
- Verifica a cada 3 segundos
- Timeout padrão: 60s
- Retorna erro se download falhar (permite correção antes de executar)

**Código:**
```bash
send_line_to_vm "fetch https://raw.githubusercontent.com/.../opnsense-bootstrap.sh.in"
wait_for_bootstrap_download $VMID 60
```

**Impacto:** 
- Garante que script foi baixado antes de executar
- Detecta problemas de rede imediatamente
- Evita executar comando `sh` em arquivo inexistente

---

### 8. ✅ Item 8: Verificar salvamento de config - wait_for_config_saved()
**Problema:** `sleep 20` após configuração de rede, sem verificar se foi salva  
**Correção:** Criada função `wait_for_config_saved()` que:
- Lê serial console de forma não-interativa
- Procura por regex `(Enter an option|0\).*Logout)` indicando menu principal
- Indica que OPNsense processou e salvou config
- Verifica a cada 2 segundos
- Timeout padrão: 30s
- Continua mesmo em timeout (para evitar bloqueio total)

**Aplicado em 2 locais:**
1. Após configuração LAN
2. Após configuração WAN (se aplicável)

**Impacto:** 
- Detecta quando config foi realmente salva
- Evita tentar configurar WAN antes de LAN estar pronta
- Economiza tempo se salvamento for rápido (<20s)
- Mais confiável que espera fixa

---

### 9. ℹ️ Item 9: Ordem single/dual
**Status:** Mantido "dual" como padrão conforme solicitação
**Nota:** Usuário confirmou que padrão deve ser "dual" para todos

---

## 📊 Resumo de Tempos

### Antes:
```
sleep 90   (VM start)
sleep 10   (bootstrap download)
sleep 1000 (OPNsense install) ← 16min 40s!
sleep 20   (config LAN)
sleep 10   (antes do logout)
─────────────────────────────
Total: 1130 segundos = ~18min 50s de sleeps fixos
```

### Depois:
```
wait_for_vm_running (até 300s, típico ~20s)
sleep 30 (boot FreeBSD)
wait_for_bootstrap_download (até 60s, típico ~5s)
sleep 5 (pause after network interface)
wait_for_opnsense_ready (até 1200s, típico ~15min)
wait_for_config_saved (até 30s, típico ~5s)
wait_for_config_saved (até 30s, se WAN)
sleep 5 (before logout)
─────────────────────────────
Total típico: ~16-17 minutos
Economia: ~2-3 minutos em cenários normais
Máximo: Ainda seguro com timeouts altos
```

---

## 🎯 Benefícios das Melhorias

### 1. **Performance**
- Script termina mais rápido quando tudo corre bem
- Não desperdiça tempo em sleeps desnecessários

### 2. **Confiabilidade**
- Verifica realmente se cada etapa completou
- Detecta falhas mais cedo (timeouts)
- Retorna códigos de erro apropriados

### 3. **Feedback ao Usuário**
- Mensagens claras sobre o que está acontecendo
- Progresso visível durante instalação longa
- Indica exatamente onde está se houver problema

### 4. **Manutenibilidade**
- Funções reutilizáveis para polling
- Timeouts configuráveis
- Código mais limpo e legível

---

## 🔧 Funções Criadas

### `read_serial_output(vmid, timeout)`
**Nova função auxiliar para leitura não-interativa do serial console**
- Conecta ao socket `/var/run/qemu-server/${vmid}.serial0`
- Usa `socat - UNIX-CONNECT:${socket_path},nonblock` (preferencial)
- Fallback para `nc -U $socket_path` se socat não disponível
- Timeout configurável (padrão: 2s)
- **Não entra na VM interativamente**
- Retorna output capturado para análise

### `wait_for_vm_running(vmid, max_wait)`
Aguarda VM entrar em estado "running"

### `wait_for_bootstrap_download(vmid, max_wait)`
**Nova função - verifica se script bootstrap foi baixado com sucesso**
- Detecta sucesso: `(opnsense-bootstrap\.sh\.in.*100%|saved|root@freebsd)`
- Detecta erros: `(fetch.*failed|unable to fetch|no route to host)`
- Retorna erro se download falhar

### `wait_for_opnsense_ready(vmid, max_wait)`
**Melhorada - agora verifica realmente quando instalação terminou**
- Substitui `sleep 1000` por verificação ativa
- Procura por: `(login:|Username:|FreeBSD.*OPNsense|Enter an option)`
- Retorna erro se timeout (evita enviar comandos para VM não pronta)

### `wait_for_config_saved(vmid, max_wait)`
**Melhorada - verifica se configuração foi salva**
- Procura por menu principal: `(Enter an option|0\).*Logout)`
- Continua mesmo em timeout (non-blocking)

---

## ⚠️ Notas Importantes

1. **Serial Console**: As funções leem `/var/run/qemu-server/${vmid}.serial0` (não-interativo)
2. **Dependências**: Requer `socat` (preferencial) ou `nc` (fallback)
   - Proxmox geralmente tem ambos instalados por padrão
   - Se necessário: `apt-get install socat netcat-openbsd`
3. **Timeout Values**: Foram mantidos generosos para redes lentas
4. **Sleeps Residuais**: Alguns sleeps curtos (5-30s) foram mantidos para estabilidade
5. **Compatibilidade**: Todas as mudanças são backward-compatible
6. **Não-interativo**: Nenhuma função usa `qm terminal` interativo

---

## 📝 Próximas Melhorias Sugeridas (Não Implementadas)

- Item 10: Validação de checksum SHA256 do FreeBSD download
- Item 11-14: Melhorias de qualidade de código (quotes, eval)
- Item 15: Retry logic para qm sendkey
- Item 16-18: Melhorias estéticas e logging

---

## 📊 Resumo das Melhorias (2025-12-01)

### Problema Resolvido
✅ **Comandos "not found" durante instalação** - Agora verifica quando instalação realmente terminou

### Novas Funções
- ✅ `read_serial_output()` - Leitura não-interativa do serial console
- ✅ `wait_for_bootstrap_download()` - Verifica download do bootstrap

### Funções Melhoradas
- ✅ `wait_for_opnsense_ready()` - Verifica instalação real (era só timeout)
- ✅ `wait_for_config_saved()` - Verifica salvamento real (era só sleep)

### Método de Verificação
- **Antes:** Timeouts fixos, sem verificação
- **Depois:** Leitura não-interativa do socket serial via `socat`/`nc`
- **Vantagem:** Detecta quando cada etapa realmente terminou

---

**Script atualizado em:** /home/alpha/Projects/study/ProxmoxVE/vm/opnsense-vm.sh  
**Testado:** ⏳ Aguardando teste real do usuário  
**Branch:** fix-opnsense-vm
**Data da correção:** 2025-12-01
