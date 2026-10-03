# ComfyUI Kaggle — Auto GPU V4.4

Notebook para instalar, configurar e manter o [ComfyUI](https://github.com/Comfy-Org/ComfyUI) em uma sessão Kaggle com GPU NVIDIA. O fluxo foi pensado para usar duas T4 quando disponíveis e manter um fallback compatível com P100, incluindo gerenciamento de modelos, autenticação no Hugging Face e acesso remoto temporário.

O projeto está concentrado em [`notebook/comfyui-kaggle-auto-gpu-manager-v4-4-fixed.ipynb`](notebook/comfyui-kaggle-auto-gpu-manager-v4-4-fixed.ipynb).

## O que o notebook faz

- Detecta as GPUs disponíveis e valida CUDA/PyTorch.
- Corrige automaticamente a instalação do Torch para P100 quando a arquitetura `sm_60` não está disponível.
- Cria uma estrutura de armazenamento temporário para modelos, entradas, saídas e arquivos temporários.
- Instala ou atualiza o ComfyUI sem substituir a pilha protegida `torch`, `torchvision` e `torchaudio`.
- Configura o ComfyUI Manager com as instalações arbitrárias por URL Git e pip desabilitadas por padrão.
- Autentica no Hugging Face usando `HF_TOKEN` ou o Kaggle Secret de mesmo nome.
- Valida o acesso a modelos gated e injeta um patch no processo real do ComfyUI para downloads autenticados, com retomada, verificação MD5 e checagem de espaço.
- Redireciona o botão de download de modelos ausentes para o servidor Kaggle, evitando que o arquivo seja baixado no computador local.
- Inicia o ComfyUI com Supervisor, grupos de processos, PID files e health-checks.
- Baixa e inicia um Cloudflare Quick Tunnel para disponibilizar uma URL pública temporária.
- Mantém a sessão monitorada e recria o túnel quando ele falha repetidamente.
- Oferece validação, diagnóstico e controle manual de restart/stop.

Fluxo geral:

```text
Kaggle GPU
   ↓
GPU/CUDA + armazenamento
   ↓
ComfyUI + Manager + roteamento de modelos
   ↓
Patch Hugging Face + bridge de downloads
   ↓
Supervisor + health-checks
   ↓
Quick Tunnel + keep-alive
```

## Requisitos

- Notebook Kaggle com acelerador GPU NVIDIA habilitado.
- Internet habilitada na sessão Kaggle.
- Espaço livre suficiente em `/kaggle/temp` e `/kaggle/working` para o ComfyUI e os modelos.
- Conta Hugging Face e um token `hf_...` para modelos gated ou privados.
- Termos de uso aceitos no Hugging Face para cada modelo gated utilizado.

T4×2 é o cenário preferencial. O notebook também detecta P100 e tenta preparar uma versão do Torch compatível com `sm_60`. Outras GPUs NVIDIA podem funcionar, mas não são o alvo principal deste projeto.

## Como usar

1. Abra o notebook no Kaggle.
2. Selecione uma sessão com GPU e habilite a Internet.
3. Crie um Kaggle Secret chamado `HF_TOKEN` contendo o token do Hugging Face. O token inline permanece desabilitado por padrão.
4. Execute as células na ordem indicada pelos cabeçalhos do notebook:

   `1` → `1B` → `1C` → `2` → `3` → `4` → `4B` → `5` → `6` → `6B` → `7` → `7B` → `8` → `8B`

5. Execute a célula `9 — DIAGNÓSTICO COMPLETO` somente quando precisar investigar uma falha.
6. Execute a célula `10 — KEEP-ALIVE + WATCHDOG DE TÚNEL` e mantenha-a em execução durante o uso.
7. Abra a URL `ACCESS URL` exibida pelo launcher ou pelo keep-alive.

Na primeira abertura do ComfyUI, faça um hard refresh do navegador para garantir que o bridge de downloads seja carregado.

## Configurações importantes

As opções principais ficam na célula `1 — CONFIGURAÇÃO`:

| Opção | Padrão | Função |
| --- | --- | --- |
| `UPDATE_COMFYUI` | `True` | Atualiza o clone do ComfyUI quando possível. |
| `COMFYUI_GIT_REF` | vazio | Permite fixar branch, tag ou commit. |
| `ENABLE_MANAGER` | `True` | Habilita o ComfyUI Manager. |
| `REQUIRE_HF_TOKEN` | `True` | Exige token para continuar a configuração. |
| `PATCH_MANAGER_HF_GATED_DOWNLOADS` | `True` | Habilita downloads autenticados do Hugging Face no processo do Manager. |
| `ENABLE_SERVER_SIDE_DOWNLOAD_BRIDGE` | `True` | Faz downloads de modelos no servidor Kaggle. |
| `ALLOW_GIT_URL_INSTALL` | `False` | Mantém instalações arbitrárias por URL Git desabilitadas. |
| `ALLOW_PIP_INSTALL` | `False` | Mantém instalações arbitrárias por pip desabilitadas. |
| `ENABLE_REMOTE_CORS` | `True` | Permite o acesso da interface através do Quick Tunnel. |

Por compatibilidade com o Quick Tunnel, o padrão usa `CORS_ORIGIN = "*"`. Se o notebook for usado em um cenário mais controlado, prefira restringir o origin e manter as instalações arbitrárias desabilitadas.

## Diretórios usados no Kaggle

- `/kaggle/working/ComfyUI`: clone do ComfyUI.
- `/kaggle/temp/comfyui/models`: modelos baixados e roteados para o ComfyUI.
- `/kaggle/temp/comfyui/input`: arquivos de entrada.
- `/kaggle/temp/comfyui/output`: resultados.
- `/kaggle/temp/comfyui/temp`: arquivos temporários.
- `/kaggle/working/.comfyui-kaggle`: PIDs, URLs, flag de parada e estado do launcher.
- `/kaggle/working/comfyui.log`: log do processo ComfyUI.
- `/kaggle/working/comfyui-supervisor.log`: log do Supervisor.

## Controle manual

A última célula permite definir `ACTION` antes de executá-la:

- `"none"`: não faz nada.
- `"restart_comfyui"`: reinicia somente o processo filho do ComfyUI; o Supervisor permanece ativo.
- `"new_tunnel"`: cria um novo Quick Tunnel.
- `"stop_all"`: encerra Supervisor, ComfyUI e Cloudflare e remove as URLs salvas.

## Segurança e limitações

- O Quick Tunnel fornece uma URL pública, temporária e sem autenticação própria. Ele não deve ser tratado como uma camada de segurança.
- O ComfyUI escuta localmente em `127.0.0.1:8188`; o acesso externo é feito pelo túnel.
- Nunca publique o notebook com um token Hugging Face preenchido em `HF_TOKEN_EXPLICIT`.
- O token é passado aos subprocessos, mas seu valor não é impresso pelo fluxo normal.
- Habilitar `ALLOW_GIT_URL_INSTALL`, `ALLOW_PIP_INSTALL` ou CORS amplo aumenta a superfície de risco e deve ser uma decisão consciente.
- A sessão Kaggle é efêmera. Arquivos em `/kaggle/temp` e `/kaggle/working` podem ser perdidos quando a sessão terminar.

## Diagnóstico rápido

Se uma validação falhar, execute a célula `9 — DIAGNÓSTICO COMPLETO` e verifique primeiro:

1. se `HF_TOKEN` está disponível e se os termos do modelo gated foram aceitos;
2. se `Supervisor PID` e `ComfyUI PID` estão ativos;
3. se a porta local `8188` responde;
4. se o `Manager API`, o bridge e a assinatura do patch HF aparecem como válidos;
5. as últimas linhas de `comfyui.log`, `comfyui-supervisor.log` e do log do Cloudflare.

## Dependências de terceiros

Este repositório contém o notebook de automação e sua documentação. O ComfyUI, o ComfyUI Manager, o `cloudflared`, os custom nodes e os modelos baixados durante a execução são componentes externos e continuam sujeitos às suas próprias licenças, termos de uso e políticas.

Os arquivos deste projeto são disponibilizados sob a [Apache License 2.0](LICENSE).
