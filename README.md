# Local AI & High-Performance Computing Lab

Este repositório documenta a implementação de um ambiente local para execução de LLMs (Large Language Models) e geração de imagens via Stable Diffusion, utilizando aceleração de hardware (GPU).

## 🛠️ Infraestrutura e Configuração

### Sistema Operacional: Windows + WSL 2
Optei por utilizar o **WSL 2 (Windows Subsystem for Linux)** para criar um ambiente de desenvolvimento híbrido.

* **Por que WSL 2?**
    * Permite o uso de ferramentas nativas Linux (essenciais para Data Science e IA) sem a sobrecarga de uma Máquina Virtual tradicional.
    * **GPU Passthrough:** O WSL 2 oferece acesso direto aos drivers da NVIDIA (CUDA) instalados no Windows, permitindo que o container Linux utilize todo o poder da minha RTX 5060 para cálculos tensoriais.
    * Facilita a integração com Docker, eliminando problemas de compatibilidade comuns no Windows nativo.

**Specs do Ambiente:**
* **CPU:** Intel Core i5 12400F
* **GPU:** NVIDIA RTX 5060 (8GB VRAM)
* **RAM:** 32GB
* **OS:** Ubuntu 22.04 LTS rodando sobre Windows 11 via WSL 2.

### ⚙️ Configuração do Ambiente Linux (Ubuntu)

Após a ativação do subsistema, foi realizada a configuração inicial do ambiente Linux:

1.  **Distribuição:** Ubuntu 22.04 LTS (Jammy Jellyfish).
    * *Escolha:* Padrão da indústria para servidores e desenvolvimento de IA, garantindo compatibilidade máxima com bibliotecas como PyTorch e TensorFlow.
2.  **Gestão de Usuários e Permissões:**
    * Criação de usuário não-root dedicado para desenvolvimento.
    * Configuração de privilégios de superusuário (`sudo`) para administração de pacotes e serviços.
3.  **Arquitetura de Filesystem:**
    * O ambiente Linux opera em seu próprio sistema de arquivos virtual (VHDX), mas mantém acesso de leitura/escrita aos arquivos do host Windows (montados em `/mnt/c/`), facilitando a troca de dados entre os sistemas.

    ### 🚀 Aceleração de Hardware e Container Runtime

Para orquestrar os modelos de IA, configurei um ambiente baseado em containers Docker com suporte a GPGPU (General-Purpose computing on Graphics Processing Units).

1.  **Integração CUDA (NVIDIA Driver Passthrough):**
    * Verificação realizada via `nvidia-smi`.
    * O WSL 2 abstrai o hardware, permitindo que bibliotecas como PyTorch acessem a RTX 5060 diretamente através da camada de compatibilidade do DirectX 12/WDDM 2.9, sem necessidade de drivers proprietários no kernel Linux.

2.  **Docker Engine (Native Linux):**
    * Optei pela instalação da *Docker Engine* nativa no ambiente Ubuntu (em vez do Docker Desktop for Windows).
    * **Benefício de Performance:** Redução de overhead de I/O de sistema de arquivos e gerenciamento de rede direto pelo kernel Linux.
    * Configuração de grupos de usuários (`usermod -aG docker`) para execução segura de containers sem privilégios de root (rootless execution context).

    ## 🧠 Engine de Inferência e Runtime

Para habilitar a execução de modelos de Deep Learning dentro de containers, foi necessária a configuração do runtime proprietário da NVIDIA.

### 1. NVIDIA Container Toolkit
O Docker padrão isola o hardware do host. Para "perfurar" esse isolamento de forma segura, instalei o `nvidia-container-toolkit`.
* **Função:** Atua como um *wrapper* para o runtime `runc` do Docker, injetando automaticamente os drivers e bibliotecas CUDA (libcuda.so, libcudart.so) dentro do container na hora da execução.
* **Comando de validação:** `sudo nvidia-ctk runtime configure --runtime=docker`

### 2. Ollama (LLM Backend)
Utilizei o **Ollama** como servidor de inferência local.
* **Modelo de Teste:** Llama 3.2 (Quantização 4-bit).
* **Performance:** A execução ocorre nativamente no Linux, acessando a GPU via chamadas CUDA diretas, resultando em latência mínima para geração de tokens.

## 🔧 Troubleshooting e Lições Aprendidas

Durante a configuração do serviço de inferência, encontrei e solucionei os seguintes comportamentos:

1.  **Conflito de Portas (Error: address already in use):**
    * **Causa:** Tentativa de iniciar o servidor `ollama serve` manualmente enquanto o serviço de background (systemd) já estava ativo pós-instalação.
    * **Solução:** O Ollama opera como um daemon. A interação deve ser feita diretamente via comandos de cliente (`ollama run`), sem necessidade de invocar o servidor manualmente na sessão do usuário.

2.  **Sintaxe de Modelos:**
    * A CLI do Ollama requer a nomenclatura exata dos modelos conforme o registro oficial (ex: `llama3.2` em vez de `llama 3.2`), sem espaços que quebrem o parsing do manifesto.

    ## 🖥️ User Interface & Orchestration (Open WebUI)

Para proporcionar uma experiência de uso comparável a soluções comerciais (SaaS), implementei uma interface web moderna via container Docker.

### Arquitetura da Solução:
* **Frontend/Application Server:** Open WebUI rodando em container Docker isolado.
* **Comunicação Inter-Processos:** Utilização da flag `--add-host=host.docker.internal:host-gateway` para permitir que o container Docker acesse o serviço do Ollama rodando no host (WSL 2), superando o isolamento padrão de rede do Docker.
* **Persistência de Dados:** Configuração de volumes Docker (`-v`) para garantir a durabilidade do histórico de conversas e configurações de usuário (RAG documents) entre reinicializações.

### Stack Visual:
A interface permite:
1.  Troca dinâmica de modelos (Hot-swapping).
2.  Formatação de código com Syntax Highlighting (essencial para Code Review).
3.  Histórico local e privado.

### ⏱️ Latência de Inicialização (Cold Start)
Notei que, ao iniciar o container do Open WebUI pela primeira vez, existe um *delay* de inicialização da aplicação interna (backend Python).
* **Sintoma:** O navegador retorna `ERR_EMPTY_RESPONSE` nos primeiros 30-60 segundos.
* **Diagnóstico:** Verificação via `docker logs -f open-webui` confirmou que o processo de *boot* do servidor Uvicorn ainda estava em andamento.
* **Solução:** Aguardar a mensagem `Application startup complete` nos logs antes de tentar o acesso via browser.

### 🌐 Configuração de Rede e Exposição de Serviços

Para viabilizar a comunicação entre o container da aplicação (Open WebUI) e o serviço de inferência no host (Ollama), realizei duas configurações críticas de rede:

1.  **Host Binding (0.0.0.0):**
    * **Problema:** O Ollama, por padrão, escuta apenas na interface de loopback (`127.0.0.1`), rejeitando conexões da bridge network do Docker.
    * **Solução:** Alteração via `systemd` override para definir a variável de ambiente `OLLAMA_HOST=0.0.0.0`, permitindo escuta em todas as interfaces de rede.

2.  **DNS Interno do Docker:**
    * Configurei o frontend para apontar para `http://host.docker.internal:11434`.
    * Isso utiliza o resolver interno do Docker para encaminhar as requisições HTTP da API para o gateway do host (WSL 2), transpassando o isolamento do container.

    ## 🤖 Seleção de Modelos (Model Zoo)

Para otimizar o uso dos recursos da GPU (8GB VRAM), selecionei uma arquitetura de modelos especializados em vez de um único modelo monolítico:

1.  **General Purpose (Raciocínio e Chat):** `Meta Llama 3.1 8B` (Quantização 4-bit).
    * Escolhido pelo equilíbrio entre coerência lógica e velocidade de inferência.
2.  **Development & Coding:** `DeepSeek Coder V2`.
    * Modelo especializado (Fine-tuned) em linguagens de programação (C, Java, Python), oferecendo performance superior em geração de sintaxe e refatoração de código.
3.  **Edge/Fast Tasks:** `Llama 3.2 3B`.
    * Utilizado apenas para tarefas de sumarização simples onde a baixa latência é prioritária.

    ### ⚠️ Limitações de Raciocínio (Known Issues)

Testes de lógica temporal demonstraram que modelos da classe 8B (Llama 3.1) tendem a falhar em "pegadinhas" semânticas simples (Zero-shot prompting).
* **Mitigação:** A implementação de técnicas de *Chain-of-Thought (CoT)* e *Few-Shot Prompting* mostrou-se necessária para forçar o modelo a decompor o problema lógico antes da geração da resposta final.

## 🎨 Pipeline de Visão Computacional (Stable Diffusion)

Para a geração de imagens, implementei uma arquitetura baseada em nós (Node-based Architecture) utilizando o **ComfyUI**.

### Decisões de Arquitetura:
* **Workflow Visual:** Diferente de interfaces monolíticas, o ComfyUI permite a visualização e manipulação granular do fluxo de tensores latentes (Latent Space tensors).
* **Otimização de VRAM:** O ComfyUI gerencia a memória da GPU de forma agressiva, carregando e descarregando modelos do VRAM conforme necessário, permitindo a execução de workflows complexos (ex: High-Res Fix, ControlNet) mesmo com limitações de hardware.

### Stack Tecnológica:
* **Framework:** PyTorch (com suporte a CUDA 12.x).
* **Motor de Difusão:** Latent Diffusion Models (LDMs).
* **Ambiente:** Execução isolada via `python venv` no WSL 2 para garantir a reprodutibilidade das dependências.

### 📦 Gestão de Modelos e Extensões

Para mitigar problemas de "Link Rot" (URLs quebradas) em scripts de automação, implementei duas estratégias de redundância:
1.  **Fallback Repositories:** Utilização do Hugging Face Hub como fonte primária para checkpoints pesados (safetensors), devido à maior estabilidade de banda e imutabilidade dos links em comparação ao Civitai.
2.  **ComfyUI Manager:** Instalação do orquestrador de dependências `ltdrdata/ComfyUI-Manager`. Isso permite a instalação via GUI de Custom Nodes e modelos, resolvendo automaticamente conflitos de dependência Python e atualizações de versão.

## ✅ Conclusão e Próximos Passos

O projeto foi concluído com sucesso, estabelecendo um laboratório de IA generativa local e privado, capaz de executar tarefas de processamento de linguagem natural e visão computacional sem dependência de nuvem.

**Resultados Chave:**
* **Infraestrutura Híbrida:** Sucesso na integração WSL 2 + Docker + NVIDIA Container Toolkit, provando ser uma arquitetura viável e performática para desenvolvimento de IA no Windows.
* **LLM (Ollama):** Implementação de modelos capazes de auxiliar em tarefas de programação (C/Python) com latência zero de rede.
* **Image Gen (ComfyUI):** Geração de imagens fotorrealistas (SDXL) em menos de 30 segundos [tempo estimado], demonstrando o poder de processamento da RTX 5060 para cargas de trabalho criativas.

**Roadmap Futuro (Férias):**
* Explorar **ControlNet** para guiar a geração de imagens usando esboços ou poses.
* Testar **Upscaling com IA** para transformar as imagens de 1024px em 4K.
* Criar um bot de Telegram que se conecta ao meu Ollama local para conversar pelo celular.