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