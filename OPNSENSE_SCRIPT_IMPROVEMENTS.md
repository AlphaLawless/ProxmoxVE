# Melhorias Implementadas no opnsense-vm.sh

## 📅 Data: 2025-12-01

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
**Problema:** `sleep 1000` (16min 40s) fixo e arbitrário  
**Correção:** Criada função `wait_for_opnsense_ready()` que:
- Verifica serial console a cada 10 segundos
- Procura por string "OPNsense.*localdomain" indicando instalação completa
- Mostra progresso a cada minuto
- Timeout padrão: 1200s (20 minutos)
- Retorna erro se timeout excedido

**Impacto:** 
- Instalação mais rápida quando OPNsense termina antes
- Detecção precisa de quando está pronto
- Feedback visual do progresso

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
**Problema:** `sleep 10` após fetch, sem verificar se download foi bem-sucedido  
**Correção:** Criada função `wait_for_bootstrap_download()` que:
- Captura output do serial console
- Procura por "opnsense-bootstrap.sh.in" no output
- Verifica a cada 2 segundos
- Timeout padrão: 60s

**Código:**
```bash
send_line_to_vm "fetch https://raw.githubusercontent.com/.../opnsense-bootstrap.sh.in"
wait_for_bootstrap_download $VMID 60
```

**Impacto:** Garante que script foi baixado antes de executar

---

### 8. ✅ Item 8: Verificar salvamento de config - wait_for_config_saved()
**Problema:** `sleep 20` após configuração de rede, sem verificar se foi salva  
**Correção:** Criada função `wait_for_config_saved()` que:
- Verifica serial console para menu principal
- Procura por regex `(Enter an option|0\).*Logout)`
- Indica que OPNsense processou e salvou config
- Timeout padrão: 30s

**Aplicado em 2 locais:**
1. Após configuração LAN
2. Após configuração WAN (se aplicável)

**Impacto:** 
- Detecta quando config foi realmente salva
- Evita tentar configurar WAN antes de LAN estar pronta
- Economiza tempo se salvamento for rápido (<20s)

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

### `wait_for_vm_running(vmid, max_wait)`
Aguarda VM entrar em estado "running"

### `wait_for_bootstrap_download(vmid, max_wait)`
Verifica se script bootstrap foi baixado

### `wait_for_opnsense_ready(vmid, max_wait)`
Aguarda instalação completa do OPNsense (substitui sleep 1000)

### `wait_for_config_saved(vmid, max_wait)`
Verifica se configuração foi salva e menu retornou

---

## ⚠️ Notas Importantes

1. **Serial Console**: As funções dependem de `qm terminal` com serial0
2. **Timeout Values**: Foram mantidos generosos para redes lentas
3. **Sleeps Residuais**: Alguns sleeps curtos (5-30s) foram mantidos para estabilidade
4. **Compatibilidade**: Todas as mudanças são backward-compatible

---

## 📝 Próximas Melhorias Sugeridas (Não Implementadas)

- Item 10: Validação de checksum SHA256 do FreeBSD download
- Item 11-14: Melhorias de qualidade de código (quotes, eval)
- Item 15: Retry logic para qm sendkey
- Item 16-18: Melhorias estéticas e logging

---

**Script atualizado em:** /home/alpha/Projects/study/ProxmoxVE/vm/opnsense-vm.sh  
**Testado:** ❌ Aguardando teste real  
**Branch:** fix-opnsense-vm
