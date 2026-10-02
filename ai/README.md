# Ollama + Vulkan + AMD Radeon Vega 8 no Ubuntu

## Objetivo

Configurar o Ollama para utilizar a GPU integrada AMD Radeon Vega 8 através do Vulkan, evitando que o modelo de IA fique sendo executado exclusivamente pela CPU.

### Ambiente

* Sistema: Ubuntu 25.04
* CPU: AMD Ryzen 3
* RAM: 16 GB
* GPU: AMD Radeon Vega 8
* GPU: integrada (iGPU)
* Ollama: 0.35.0
* Modelo: Qwen 3.5 4B

---

# 1. Verificar o Ollama

Primeiro, verificar se o Ollama está instalado:

```bash
ollama version
```

Resultado utilizado:

```text
ollama version is 0.35.0
```

Também verificamos o serviço:

```bash
systemctl status ollama --no-pager
```

O Ollama estava sendo executado como um serviço `systemd`:

```text
ollama.service - Ollama Service
Active: active (running)
```

Isso significa que o `systemd` é responsável por iniciar e manter o Ollama rodando em segundo plano.

---

# 2. Identificar a GPU

Foi utilizada:

```bash
lspci | grep -i vga
```

A GPU encontrada foi:

```text
AMD/ATI Raven Ridge [Radeon Vega Series / Radeon Vega Mobile Series]
```

A GPU é uma Radeon Vega 8 integrada.

---

# 3. Instalar as ferramentas Vulkan

O sistema inicialmente não possuía o comando `vulkaninfo`.

Foi instalado:

```bash
sudo apt install vulkan-tools
```

Depois:

```bash
vulkaninfo --summary
```

O Vulkan passou a reconhecer:

```text
GPU0:
AMD Radeon Vega 8 Graphics (RADV RAVEN)
```

Também apareceu:

```text
GPU1:
llvmpipe
```

O `llvmpipe` é uma implementação de renderização por software que utiliza a CPU.

Por isso queríamos selecionar especificamente o dispositivo `GPU0`.

---

# 4. Verificar o funcionamento do Vulkan

O comando:

```bash
vulkaninfo --summary
```

mostrou:

```text
deviceName = AMD Radeon Vega 8 Graphics (RADV RAVEN)
driverID = DRIVER_ID_MESA_RADV
driverName = radv
```

Isso confirmou que o Vulkan estava funcionando corretamente com o driver RADV da AMD.

---

# 5. Verificar o uso inicial do Qwen

Antes da configuração, o modelo:

```bash
ollama run qwen3.5:4b
```

estava sendo executado com:

```bash
ollama ps
```

Resultado:

```text
qwen3.5:4b    3.1 GB    100% CPU    4096
```

Ou seja:

```text
Qwen
 ↓
CPU
 ↓
Ryzen 3
```

O modelo estava sendo executado totalmente pela CPU.

---

# 6. Criar um override do serviço do Ollama

Como o Ollama estava sendo executado pelo `systemd`, não foi necessário modificar diretamente o arquivo original do serviço.

Foi utilizado:

```bash
sudo systemctl edit ollama.service
```

Isso abriu o arquivo:

```text
/etc/systemd/system/ollama.service.d/override.conf
```

Foi adicionada a seguinte configuração:

```ini
[Service]
Environment="OLLAMA_VULKAN=1"
Environment="OLLAMA_IGPU_ENABLE=1"
Environment="GGML_VK_VISIBLE_DEVICES=0"
```

## O que cada configuração faz?

### OLLAMA_VULKAN

```ini
Environment="OLLAMA_VULKAN=1"
```

Ativa o suporte ao Vulkan no Ollama.

---

### OLLAMA_IGPU_ENABLE

```ini
Environment="OLLAMA_IGPU_ENABLE=1"
```

Permite que o Ollama utilize uma GPU integrada.

No nosso caso:

```text
AMD Radeon Vega 8
```

---

### GGML_VK_VISIBLE_DEVICES

```ini
Environment="GGML_VK_VISIBLE_DEVICES=0"
```

Define qual dispositivo Vulkan deve ficar disponível para o Ollama.

O Vulkan mostrou:

```text
GPU0 → AMD Radeon Vega 8
GPU1 → llvmpipe
```

Portanto:

```text
0 → Vega 8
```

Foi escolhido o dispositivo `0`.

---

# 7. Salvar o override

No nano:

```text
Ctrl + O
```

Depois:

```text
Enter
```

E sair:

```text
Ctrl + X
```

---

# 8. Recarregar o systemd

Depois de alterar um arquivo de serviço, o systemd precisa ler novamente as configurações:

```bash
sudo systemctl daemon-reload
```

---

# 9. Reiniciar o Ollama

Depois:

```bash
sudo systemctl restart ollama
```

---

# 10. Verificar o serviço

Foi utilizado:

```bash
systemctl status ollama --no-pager
```

O resultado mostrou:

```text
Drop-In: /etc/systemd/system/ollama.service.d
         └─override.conf

Active: active (running)
```

Isso confirmou que o override estava sendo carregado.

---

# 11. Verificar os logs do Ollama

Foi utilizado:

```bash
journalctl -u ollama -n 30 --no-pager
```

O log mostrou:

```text
OLLAMA_VULKAN:true
OLLAMA_IGPU_ENABLE:1
GGML_VK_VISIBLE_DEVICES:0
```

E principalmente:

```text
library=Vulkan
name=Vulkan0
description="AMD Radeon Vega 8 Graphics (RADV RAVEN)"
type=iGPU
```

Isso confirmou que o Ollama encontrou a Vega 8 através do Vulkan.

---

# 12. Testar o modelo novamente

Foi executado:

```bash
ollama run qwen3.5:4b
```

Enquanto o modelo estava carregado, foi utilizado:

```bash
ollama ps
```

O resultado passou de:

```text
100% CPU
```

para:

```text
100% GPU
```

Resultado:

```text
qwen3.5:4b    3.1 GB    100% GPU    4096
```

Isso indica que o modelo passou a ser executado pelo backend de GPU do Ollama.

---

# Resultado final

Antes:

```text
Qwen 3.5 4B
      ↓
    Ollama
      ↓
     CPU
      ↓
  Ryzen 3
```

Depois:

```text
Qwen 3.5 4B
      ↓
    Ollama
      ↓
    Vulkan
      ↓
    RADV
      ↓
Radeon Vega 8
```

## Configuração final

Arquivo:

```text
/etc/systemd/system/ollama.service.d/override.conf
```

Conteúdo:

```ini
[Service]
Environment="OLLAMA_VULKAN=1"
Environment="OLLAMA_IGPU_ENABLE=1"
Environment="GGML_VK_VISIBLE_DEVICES=0"
```

## Comandos principais para lembrar

Verificar Ollama:

```bash
systemctl status ollama --no-pager
```

Verificar GPU/Vulkan:

```bash
vulkaninfo --summary
```

Verificar uso do modelo:

```bash
ollama ps
```

Ver logs:

```bash
journalctl -u ollama -n 30 --no-pager
```

Editar configuração:

```bash
sudo systemctl edit ollama.service
```

Recarregar configuração:

```bash
sudo systemctl daemon-reload
```

Reiniciar:

```bash
sudo systemctl restart ollama
```

## Observação sobre a memória da GPU

A Radeon Vega 8 é uma GPU integrada.

Ela não possui uma VRAM dedicada como uma placa de vídeo dedicada.

A GPU compartilha a memória RAM do computador.

Portanto, quando o Ollama mostrou:

```text
total="8.5 GiB"
```

isso não significa que existem 8,5 GB de VRAM física dedicada.

## Observação sobre ROCm

O log também apresentou:

```text
dropping ROCm device — no rocblas support for gfx target
```

Isso significa que o caminho ROCm não foi utilizado para essa GPU.

Entretanto, o Ollama encontrou a GPU através do Vulkan:

```text
library=Vulkan
```

Portanto, não foi necessário instalar ROCm para realizar esta configuração.

---

# Resumo rápido

```text
CPU
↓
boa para tarefas variadas e lógica geral

GPU
↓
boa para muitas operações matemáticas em paralelo

Vulkan
↓
API que permite ao software utilizar a GPU

RADV
↓
driver Vulkan utilizado pela AMD no Linux

Ollama
↓
executa o modelo

Qwen 3.5 4B
↓
modelo de IA

Resultado
↓
Qwen 3.5 4B → Vulkan → Radeon Vega 8
```
