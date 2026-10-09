# Servidor de inferência LLM para NVIDIA DGX Spark

Serviço compartilhado de LLMs abertos numa DGX Spark (GB10 Grace Blackwell, ARM64, 128 GB de
memória unificada), com interface de chat no navegador e API remota compatível com a OpenAI.

```
navegador ─── https://<SERVIDOR>         ─> nginx ─> Open WebUI ─┐
cliente API ─ https://<SERVIDOR>:8443/v1 ─> nginx ───────────────┼─> LiteLLM ─> motor ativo
na máquina ── http://127.0.0.1:4000/v1 ──────────────────────────┘    (vLLM ou llama.cpp)
```

- **Open WebUI:** chat no navegador, com login e aprovação de novos usuários.
- **LiteLLM:** gateway da API: uma chave por pessoa, limites de uso e registro.
- **vLLM / llama.cpp:** os motores de inferência, **um ativo por vez**, com a memória toda.
- **nginx:** único serviço exposto à rede, com TLS.

Instalação e reinstalação: veja [INSTALL.md](INSTALL.md).

Nos exemplos abaixo, `<SERVIDOR>` é o IP ou o nome da máquina na rede (`SERVER_NAME` no `.env`).

## Modelos

| Nome na API | Motor | Modelo | Ativar com |
|---|---|---|---|
| `modelo-principal` | vLLM | Qwen/Qwen3-32B (bf16) | `scripts/motor.sh qwen32b` |
| `modelo-secundario` | llama.cpp | Qwen3.6-35B-A3B Uncensored (GGUF Q8_K_P) | `scripts/motor.sh uncensored` |

Os dois nomes aparecem sempre na lista, mas só o do motor ativo responde. Pedir o outro
devolve erro, em vez de cair no modelo errado sem aviso.

## Trocar o motor

Na máquina do servidor, dentro da pasta do projeto:

```bash
scripts/motor.sh status        # qual motor está ativo
scripts/motor.sh qwen32b       # vLLM + Qwen3-32B
scripts/motor.sh uncensored    # llama.cpp + GGUF
```

Durante a troca o serviço fica alguns minutos sem modelo (o novo precisa carregar). LiteLLM,
Open WebUI, nginx e chaves não são reiniciados. Avise quem estiver usando, principalmente
quem roda jobs em lote.

## Uso pelo navegador

Abra `https://<SERVIDOR>`. Com o certificado autoassinado, o navegador mostra um aviso na
primeira visita: aceite a exceção. Novas contas ficam pendentes até um administrador
aprovar. Escolha o modelo no topo do chat.

## Uso pela API

A API é compatível com a da OpenAI: qualquer biblioteca ou ferramenta que aceite um
`base_url` personalizado funciona. Toda requisição precisa de uma chave
(`Authorization: Bearer sk-...`), entregue pelo administrador.

| De onde | Endereço | Certificado |
|---|---|---|
| Na própria máquina do servidor | `http://127.0.0.1:4000/v1` | não precisa |
| Pela rede | `https://<SERVIDOR>:8443/v1` | `fullchain.pem` (pedir ao administrador) |

O `fullchain.pem` é público e pode ser compartilhado. Ele serve para o cliente verificar
que está falando com o servidor certo. Nunca desative essa verificação: a chave de API
trafega nessa conexão.

### Teste rápido com curl

```bash
# na máquina do servidor
curl -s http://127.0.0.1:4000/v1/chat/completions \
  -H "Authorization: Bearer $LLM_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"modelo-principal","max_tokens":100,
       "messages":[{"role":"user","content":"Diga olá em uma frase."}]}'

# pela rede
curl -s --cacert fullchain.pem https://<SERVIDOR>:8443/v1/chat/completions \
  -H "Authorization: Bearer $LLM_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"modelo-principal","max_tokens":100,
       "messages":[{"role":"user","content":"Diga olá em uma frase."}]}'
```

### Python

```bash
pip install openai httpx
export LLM_API_KEY=sk-...
# só para uso remoto:
export LLM_BASE_URL=https://<SERVIDOR>:8443/v1
export LLM_CA_CERT=/caminho/fullchain.pem
```

```python
import os, httpx
from openai import OpenAI

cliente = OpenAI(
    base_url=os.getenv("LLM_BASE_URL", "http://127.0.0.1:4000/v1"),
    api_key=os.environ["LLM_API_KEY"],
    http_client=httpx.Client(verify=os.getenv("LLM_CA_CERT") or True, timeout=600),
)

r = cliente.chat.completions.create(
    model="modelo-principal",
    messages=[{"role": "user", "content": "O que é aprendizado de máquina?"}],
    max_tokens=400,
)
print(r.choices[0].message.content)
```

Mais exemplos em [`exemplos/`](exemplos/):

| Arquivo | O que faz |
|---|---|
| `teste_conexao.py` | Confere endereço, certificado, chave e modelo com uma pergunta curta |
| `uso_basico.py` | Pergunta simples, streaming, raciocínio, histórico de conversa, saída em JSON |
| `batch_cliente.py` | Processamento em lote de um JSONL, com paralelismo, novas tentativas e retomada |

### Raciocínio (Qwen3)

No `modelo-principal`, o raciocínio vem **desligado por padrão** (respostas diretas e
rápidas). Para ligar numa requisição:

```python
r = cliente.chat.completions.create(
    model="modelo-principal", max_tokens=3000,
    messages=[{"role": "user", "content": "Quanto é 17 x 23?"}],
    extra_body={"chat_template_kwargs": {"enable_thinking": True}},
)
print(r.choices[0].message.reasoning_content)   # o raciocínio
print(r.choices[0].message.content)             # a resposta final
```

Reserve `max_tokens` de sobra (2000+): se o limite cortar o raciocínio antes do fim, ele
aparece misturado na resposta. No `modelo-secundario` (llama.cpp), o raciocínio vem ligado
por padrão.

### Jobs em lote

`exemplos/batch_cliente.py` substitui o uso do vLLM como biblioteca
(`LLM(model=...).generate()`), que carregaria uma segunda cópia do modelo na memória:

```bash
# entrada.jsonl: {"id": "1", "prompt": "..."} por linha
python exemplos/batch_cliente.py entrada.jsonl saida.jsonl
```

- Envia requisições em paralelo (`LLM_CONCORRENCIA`, padrão 16) e o servidor agrupa tudo
  sozinho (continuous batching).
- Trata limite de taxa (429) e falhas temporárias com novas tentativas.
- Grava cada resultado assim que chega. Rodar de novo refaz só o que faltou ou falhou
  (use a última linha de cada `id`).
- `LLM_MODO=completion` equivale a `llm.generate(prompts)` (texto cru, sem chat template);
  `LLM_MODO=chat` (padrão) equivale a `llm.chat(...)`.
- Parâmetros de amostragem (`temperature`, `max_tokens`, `top_k`, `seed`...) ficam em
  `PARAMS` e `EXTRA` no topo do script.
- Para jobs grandes, peça uma chave sem limite por minuto.

### Limites

- Cada IP pode fazer até 20 requisições por segundo na porta 8443 (picos de até 100).
- Cada chave pode ter limite próprio por minuto e data de validade.
- O contexto máximo é de 32K tokens por requisição.

## Administração

Na máquina do servidor, dentro da pasta do projeto.

```bash
scripts/nova-chave.sh fulano 30 90d        # 30 req/min, válida por 90 dias
scripts/nova-chave.sh fulano-batch 0 365d  # sem limite por minuto, 1 ano
scripts/revogar-chave.sh fulano            # revoga pelo apelido
scripts/chave-webui.sh                     # troca a chave usada pelo Open WebUI
```

Entregue cada chave por canal privado. As chaves valem para os dois modelos.

```bash
docker compose ps                          # estado dos serviços
docker compose logs -f --tail 100 litellm  # logs (vllm, llamacpp, open-webui, nginx...)
free -h                                    # memória (o nvidia-smi não mostra na Spark)
scripts/verificar-host.sh                  # checagem geral da máquina
```

- Os serviços voltam sozinhos depois de um reboot, com o motor que estava ativo.
- Com o serviço no ar, ninguém deve carregar modelos diretamente na máquina (vLLM,
  PyTorch...): a memória é unificada e as duas cargas disputam os mesmos 128 GB.
- Para usar a GPU inteira por um tempo, combine uma janela e pare o motor:
  `docker rm -f vllm llamacpp`. Para voltar: `scripts/motor.sh qwen32b`.
- Outro modelo no vLLM: `scripts/trocar-modelo.sh <repo-hf>`. Outro GGUF:
  `scripts/baixar-gguf.sh <repo-hf> --listar` e depois `<quantização>`.

## Segurança

- Só o nginx (80, 443, 8443) é exposto. Pela porta 8443 só a rota `/v1/` funciona; o
  painel e as rotas de administração do LiteLLM ficam acessíveis apenas em `127.0.0.1:4000`.
- vLLM, llama.cpp e Postgres ficam numa rede interna do Docker, sem acesso à internet.
- O `.env` guarda a chave mestra: fica com `chmod 600` e fora do git (`.gitignore`).
- `nginx/certs/privkey.pem` nunca sai da máquina; só o `fullchain.pem` é distribuído.
