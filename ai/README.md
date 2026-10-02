# Ollama + Vulkan + AMD Radeon Vega 8 no Ubuntu

## Objetivo

Configurar o Ollama para utilizar a GPU integrada AMD Radeon Vega 8 através do Vulkan, evitando que o modelo de IA fique sendo executado exclusivamente pela CPU.

Também configurar o **Open WebUI** através do Docker para fornecer uma interface gráfica para conversar com os modelos do Ollama.

### Ambiente

* Sistema: Ubuntu 25.04
* CPU: AMD Ryzen 3
* RAM: 16 GB
* GPU: AMD Radeon Vega 8
* GPU: integrada (iGPU)
* Ollama: 0.35.0
* Modelo inicial: Qwen 3.5 4B
* Modelo testado posteriormente: Ministral 3 3B
* Open WebUI: Docker

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

Posteriormente, foi adicionada também a configuração necessária para permitir que o Open WebUI, executado em Docker, acessasse o Ollama:

```ini
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

A configuração final ficou:

```ini
[Service]
Environment="OLLAMA_VULKAN=1"
Environment="OLLAMA_IGPU_ENABLE=1"
Environment="GGML_VK_VISIBLE_DEVICES=0"
Environment="OLLAMA_HOST=0.0.0.0:11434"
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

### OLLAMA_HOST

```ini
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

Define o endereço em que o servidor do Ollama ficará disponível.

Inicialmente, o Ollama estava acessível somente através de:

```text
127.0.0.1:11434
```

Isso funcionava para programas executados diretamente no computador, mas o Open WebUI estava dentro de um container Docker.

O container não conseguia acessar o Ollama através de `127.0.0.1`, porque dentro do container `localhost` se refere ao próprio container.

Por isso foi utilizado:

```text
0.0.0.0:11434
```

Isso permite que o Ollama aceite conexões vindas do Docker.

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

# 13. Instalar o Open WebUI

O Open WebUI foi instalado utilizando Docker.

Primeiro, foi executado:

```bash
docker run -d \
  -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  -e WEBUI_SECRET_KEY=your-secret-key \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

## O que esse comando faz?

### Porta

```bash
-p 3000:8080
```

A porta `8080` do container é disponibilizada na porta `3000` do computador.

Assim, o Open WebUI pode ser acessado em:

```text
http://localhost:3000
```

---

### Acesso ao host

```bash
--add-host=host.docker.internal:host-gateway
```

Cria dentro do container o endereço:

```text
host.docker.internal
```

Esse endereço permite que o container encontre o computador host.

Isso é necessário porque o Ollama está rodando no Ubuntu, enquanto o Open WebUI está rodando dentro do Docker.

A comunicação fica:

```text
Open WebUI
   ↓
Docker
   ↓
host.docker.internal
   ↓
Ollama
   ↓
localhost:11434 / servidor Ollama
```

---

### Volume

```bash
-v open-webui:/app/backend/data
```

Cria um volume Docker para armazenar os dados do Open WebUI.

Isso permite manter os dados mesmo que o container seja recriado.

---

### Reinicialização automática

```bash
--restart always
```

Faz o Docker tentar iniciar novamente o Open WebUI caso o container seja reiniciado.

---

# 14. Verificar o container do Open WebUI

Foi utilizado:

```bash
docker ps
```

O container apareceu como:

```text
open-webui
0.0.0.0:3000->8080/tcp
```

Isso significa:

```text
Computador
localhost:3000
       ↓
Docker
       ↓
Open WebUI
porta 8080
```

---

# 15. Configurar o Ollama no Open WebUI

Dentro do Open WebUI, foi configurado o endereço do Ollama como:

```text
http://host.docker.internal:11434
```

O Open WebUI utiliza esse endereço para acessar o servidor Ollama que está rodando no computador.

---

# 16. Testar a comunicação entre Docker e Ollama

Antes da configuração correta do `OLLAMA_HOST`, o container não conseguia acessar o Ollama:

```text
curl: (7) Failed to connect to host.docker.internal port 11434
```

Isso acontecia porque o Ollama estava escutando somente em:

```text
127.0.0.1:11434
```

Depois de configurar:

```ini
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

e reiniciar o serviço, foi realizado o teste:

```bash
docker exec open-webui curl http://host.docker.internal:11434/api/tags
```

O Ollama passou a responder com a lista de modelos:

```json
{
  "models": [
    {
      "name": "qwen3.5:4b",
      "model": "qwen3.5:4b"
    }
  ]
}
```

Isso confirmou que:

```text
Docker
   ↓
Open WebUI
   ↓
host.docker.internal:11434
   ↓
Ollama
```

estava funcionando corretamente.

---

# 17. Utilizar modelos no Open WebUI

Depois que a comunicação foi estabelecida, os modelos instalados no Ollama podem ser utilizados pelo Open WebUI.

Por exemplo:

```bash
ollama list
```

pode mostrar modelos como:

```text
qwen3.5:4b
ministral-3:3b
```

Para baixar um novo modelo:

```bash
ollama pull ministral-3:3b
```

Depois disso, o modelo pode ser selecionado no Open WebUI.

## Modelo utilizado nos testes

Além do Qwen 3.5 4B, foi testado:

```bash
ollama run ministral-3:3b
```

O `ministral-3:3b` apresentou resposta significativamente mais rápida no terminal e mostrou bom desempenho para explicações didáticas.

Exemplo de teste:

```text
Explique o que é um ponteiro em C para uma pessoa que está aprendendo
programação pela primeira vez. Use um exemplo simples.
```

O modelo conseguiu produzir uma explicação estruturada, incluindo conceito, exemplos de código, operadores `&` e `*`, aplicações e riscos.

---

# 18. Arquitetura final

A configuração completa ficou aproximadamente assim:

```text
                    ┌──────────────────────┐
                    │      Open WebUI      │
                    │       Docker         │
                    │      :3000           │
                    └──────────┬───────────┘
                               │
                               │ HTTP
                               ↓
                    host.docker.internal
                         :11434
                               │
                               ↓
                    ┌──────────────────────┐
                    │        Ollama        │
                    │      systemd         │
                    └──────────┬───────────┘
                               │
                               ↓
                            Vulkan
                               │
                               ↓
                             RADV
                               │
                               ↓
                    ┌──────────────────────┐
                    │   Radeon Vega 8      │
                    │        iGPU          │
                    └──────────────────────┘
```

---

# 19. Resultado final

Antes da configuração:

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

Com o Open WebUI:

```text
Usuário
   ↓
Open WebUI
   ↓
Docker
   ↓
host.docker.internal:11434
   ↓
Ollama
   ↓
Vulkan
   ↓
Radeon Vega 8
```

---

# Configuração final

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
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

---

# Comandos principais para lembrar

## Ollama

Verificar Ollama:

```bash
systemctl status ollama --no-pager
```

Verificar versão:

```bash
ollama version
```

Listar modelos:

```bash
ollama list
```

Executar um modelo:

```bash
ollama run ministral-3:3b
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

---

## Vulkan

Verificar GPU:

```bash
vulkaninfo --summary
```

---

## Docker / Open WebUI

Ver containers:

```bash
docker ps
```

Ver logs do Open WebUI:

```bash
docker logs open-webui
```

Testar acesso do Docker ao Ollama:

```bash
docker exec open-webui curl http://host.docker.internal:11434/api/tags
```

Open WebUI:

```text
http://localhost:3000
```

---

# Observação sobre a memória da GPU

A Radeon Vega 8 é uma GPU integrada.

Ela não possui uma VRAM dedicada como uma placa de vídeo dedicada.

A GPU compartilha a memória RAM do computador.

Portanto, quando o Ollama mostrou:

```text
total="8.5 GiB"
```

isso não significa que existem 8,5 GB de VRAM física dedicada.

A memória é compartilhada com os demais componentes do sistema.

---

# Observação sobre ROCm

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
executa os modelos de IA

Open WebUI
↓
interface gráfica para utilizar os modelos

Docker
↓
executa o Open WebUI isoladamente

Qwen / Ministral
↓
modelos de IA

Resultado
↓
Open WebUI
→ Docker
→ Ollama
→ Vulkan
→ RADV
→ Radeon Vega 8
```

## Configuração essencial

```text
Ollama
    ↓
OLLAMA_VULKAN=1
OLLAMA_IGPU_ENABLE=1
GGML_VK_VISIBLE_DEVICES=0
OLLAMA_HOST=0.0.0.0:11434
    ↓
Radeon Vega 8 via Vulkan

Open WebUI
    ↓
http://host.docker.internal:11434
    ↓
Ollama
```
