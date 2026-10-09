# Instalação

Passo a passo para instalar (ou reinstalar do zero) o serviço numa NVIDIA DGX Spark.
Rode tudo **na Spark**, como um usuário com `sudo` e no grupo `docker`. Confira a saída de
cada passo antes de seguir.

Tempo total: ~1 h, a maior parte em downloads (~65 GB do Qwen3-32B e ~44 GB do GGUF).

## Requisitos

- DGX OS (Ubuntu 24.04, ARM64) com Docker e NVIDIA Container Toolkit (já vêm instalados).
- Driver NVIDIA 580.x. **A imagem do vLLM precisa ser compatível com o driver**: com o 580,
  use `nvcr.io/nvidia/vllm:26.02-py3` (vLLM 0.15.1). A 26.04+ exige driver 595.58+ e para na
  inicialização com "compatibility mode is UNAVAILABLE".
- ~150 GB livres em disco e acesso à internet (Hugging Face, NGC, GitHub).
- Ninguém mais usando a GPU/memória de forma pesada (confira no passo 2).

## 0. Reinstalação: remover a instalação anterior

Pule se for a primeira instalação. Os modelos já baixados ficam em `hf-cache` e são
reaproveitados.

```bash
cd ~/llm-server/dgx-spark-llm-server
docker rm -f vllm llamacpp 2>/dev/null
docker compose down -v          # -v apaga chaves e contas do Open WebUI
cd ~/llm-server && rm -rf dgx-spark-llm-server
```

## 1. Obter o código

```bash
mkdir -p ~/llm-server && cd ~/llm-server
git clone <URL-DO-REPOSITÓRIO> dgx-spark-llm-server    # ou: unzip dgx-spark-llm-server.zip
cd dgx-spark-llm-server
chmod +x scripts/*.sh
mkdir -p ~/llm-server/hf-cache/hub
```

Daqui em diante, todos os comandos rodam dentro de `~/llm-server/dgx-spark-llm-server`.

## 2. Preparar a máquina

Seu usuário precisa estar no grupo `docker` (um administrador roda
`sudo usermod -aG docker <usuario>`; depois saia e entre de novo).

Veja se há outros processos ocupando memória (jobs antigos, outro vLLM...):

```bash
ps -eo user,pid,rss,etime,cmd --sort=-rss | head -8 | awk '{ $3 = int($3/1024/1024) "G"; print }' | cut -c1-150
nvidia-smi --query-compute-apps=pid,process_name --format=csv
```

Combine com os donos antes de encerrar qualquer coisa. Depois, esvazie o swap (precisa de
memória livre maior que o swap usado; pode levar alguns minutos):

```bash
sudo swapoff -a && sudo swapon -a
free -h
```

## 3. Configurar

```bash
cp .env.example .env
scripts/gerar-segredos.sh                  # deve mostrar 5 linhas "gerado:"
sed -i "s|^HF_CACHE_DIR=.*|HF_CACHE_DIR=$HOME/llm-server/hf-cache|" .env
nano .env
```

No `nano`, ajuste:

- `SERVER_NAME`: o IP da Spark na rede (`ip -4 -br addr`) ou o nome DNS, se houver.
- `HF_TOKEN`: token do Hugging Face (opcional para modelos abertos; evita limites).
- Confira `VLLM_IMAGE=nvcr.io/nvidia/vllm:26.02-py3` (ver Requisitos).

Se a rede local usar a faixa `10.254.x.x`, mude `DOCKER_SUBNET_BACKEND/FRONTEND` para uma
faixa livre (o Docker, por padrão, usa 172.17–172.31.x.x, que costuma colidir com redes
institucionais).

## 4. Certificado e verificação

```bash
scripts/gerar-certificado.sh               # autoassinado, para o SERVER_NAME
scripts/verificar-host.sh                  # deve terminar em "Pronto para seguir"
```

Se houver certificado institucional, coloque-o em `nginx/certs/fullchain.pem` e
`nginx/certs/privkey.pem` em vez de gerar.

## 5. Baixar os modelos

O vLLM e o llama.cpp rodam **sem acesso à internet**: os modelos precisam ser baixados antes.
Use `tmux` para downloads longos (se a conexão cair, `tmux attach` e rode de novo: o download
continua de onde parou).

```bash
docker compose --profile qwen32b pull      # imagens (vários GB)
scripts/baixar-modelo.sh                   # Qwen3-32B, ~62 GB
scripts/baixar-gguf.sh HauhauCS/Qwen3.6-35B-A3B-Uncensored-HauhauCS-Aggressive --listar
scripts/baixar-gguf.sh HauhauCS/Qwen3.6-35B-A3B-Uncensored-HauhauCS-Aggressive Q8_K_P
```

O aviso "Could not set the permissions on the file" durante o download pode ser ignorado.

## 6. Subir o motor principal

```bash
scripts/motor.sh qwen32b                   # carrega o Qwen3-32B (~8 min)
free -h                                    # ~19 GB disponíveis com o modelo carregado
```

## 7. Subir gateway, interface e nginx

```bash
docker compose up -d litellm
sleep 20 && curl -s http://127.0.0.1:4000/health/liveliness; echo   # "I'm alive!"
scripts/chave-webui.sh                     # cria a chave do Open WebUI no .env
docker compose up -d
docker compose ps                          # 5 serviços; open-webui leva 1-3 min até "healthy"
```

Abra `https://<SERVER_NAME>` no navegador, aceite o aviso do certificado e **crie logo a
conta de administrador** (o primeiro cadastro vira admin).

## 8. Testar

```bash
K=$(scripts/nova-chave.sh teste 10 1d)
curl -s --cacert nginx/certs/fullchain.pem https://$(grep ^SERVER_NAME .env | cut -d= -f2):8443/v1/chat/completions \
  -H "Authorization: Bearer $K" -H "Content-Type: application/json" \
  -d '{"model":"modelo-principal","max_tokens":100,"messages":[{"role":"user","content":"Diga olá."}]}'
curl -sk -o /dev/null -w "sem chave: %{http_code}\n" https://localhost:8443/v1/models   # 401
curl -sk -o /dev/null -w "painel: %{http_code}\n"    https://localhost:8443/ui          # 404
scripts/revogar-chave.sh teste
```

## 9. Motor alternativo (llama.cpp)

```bash
scripts/motor.sh uncensored    # 1ª vez compila o llama.cpp para a GB10 (~3-30 min)
scripts/motor.sh qwen32b       # volta ao principal
```

Durante a compilação o motor atual continua no ar; o serviço só fica sem modelo enquanto o
novo carrega.

## Problemas conhecidos

| Sintoma | Causa | Solução |
|---|---|---|
| `compatibility mode is UNAVAILABLE` no vLLM | Imagem do vLLM nova demais para o driver | `VLLM_IMAGE=nvcr.io/nvidia/vllm:26.02-py3` com driver 580 |
| `Failed to resolve 'huggingface.co'` no vLLM | Motor sem internet (de propósito) | Baixar antes: `scripts/baixar-modelo.sh` |
| `"auto" tool choice requires --enable-auto-tool-choice` | Ferramentas não habilitadas no vLLM | `VLLM_EXTRA_ARGS` com `--enable-auto-tool-choice --tool-call-parser hermes` |
| Resposta começa com `<think>` e vem cortada | Raciocínio maior que `max_tokens` | Aumentar `max_tokens` ou desligar o raciocínio |
| Open WebUI: "Nenhum modelo disponível" após trocar a chave | O Open WebUI ignora o `.env` depois da 1ª inicialização | `scripts/chave-webui.sh` e colar a chave em Painel do Administrador > Configurações > Conexões |
| `docker compose up` não recria um serviço após editar o `.env` | Variável com o mesmo nome exportada no terminal tem prioridade | Abrir um terminal novo (ou `unset` a variável) e repetir |
| Compilação do llama.cpp falha em `ui-assets.cmake` / `loading.html` | Interface web embutida do llama.cpp | Já desligada no Dockerfile (`-DLLAMA_BUILD_UI=OFF`) |
| `qwen3: command not found` ao rodar um script | `.env` lido como shell (versão antiga) | Scripts atuais usam `scripts/_env.sh`; valores com espaço entre aspas |
| Máquina lenta ou travando ao carregar o modelo | Swap cheio / outros processos na memória | Passo 2 |
